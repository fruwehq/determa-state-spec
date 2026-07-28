# Determa State — specification

Status: **pre-release alpha**. Normative unless a section says informative.
Document format: **1**.
Spec version: **0.0.7** (see `VERSION`; synchronized across the Determa State
repositories).
Keywords MUST / SHOULD / MAY are interpreted as in RFC 2119.

## 0. References and semantic independence

- David Harel, *Statecharts: A Visual Formalism for Complex Systems*.
- OMG UML State Machines, for terminology and established statechart concepts.
- Miro Samek, *Practical UML Statecharts in C/C++*, for implementation lessons and
  event-driven design patterns.
- CEL — Common Expression Language (<https://cel.dev/>), used for guards and computed
  action values.

Determa State has Harel/UML lineage, but this document defines Determa semantics. A
similar name or diagram shape does not import behavior from another framework,
runtime, or notation. Where established statechart dialects differ, this document
makes an explicit choice and the conformance suite pins that choice.

## 1. Purpose and core boundary

Determa State defines portable, deterministic statechart behavior:

1. typed immutable input envelopes;
2. hierarchical state configuration and transition selection;
3. run-to-completion processing of one envelope;
4. typed state-scoped variables and pure CEL expressions;
5. structured actions that update logical state or emit immutable intents;
6. isolated reusable components and owned spawned runtimes; and
7. deterministic identities, lifecycle, rollback, and fault results.

The core deliberately does **not** provide:

- an event queue, delivery worker, scheduler, or background thread;
- dead-letter storage or a dead-letter policy;
- a clock, timer, delay, sleep, or time-event implementation;
- transport, broker, retry, acknowledgement, or delivery guarantees;
- a state store, transaction manager, credentials, or external I/O; or
- plugin discovery, installation, configuration schemas, or package resolution.

Those are host or plugin responsibilities (§11). The core receives one envelope,
processes it atomically, and returns state plus ordered emissions. It never calls a
queue, timer, broker, database, or remote service from a guard or action.

This boundary permits an in-memory foreground host, a database-backed request/response
host, a durable worker, or a distributed broker without changing statechart semantics.
End-to-end behavior remains conditional on the delivery trace and guarantees of the
selected plugins.

## 2. Conformance, parsing, and format identity

An implementation is conformant iff it passes every applicable case in
[`fruwehq/determa-state-conformance`](https://github.com/fruwehq/determa-state-conformance).
The conformance suite is the executable arbiter. If prose and the suite disagree, a bug
MUST be filed and resolved; implementations MUST NOT choose their preferred result.

Machine documents MUST be parsed with the YAML 1.2 core schema and validated against
`schema/machine.schema.json` before semantic validation.

The accepted parsed value model is deliberately narrower than general YAML:

- a source contains exactly one document;
- every mapping key is a string and occurs exactly once;
- duplicate JSON object names and duplicate YAML mapping keys are rejected before
  schema validation;
- YAML anchors, aliases, merge keys, and explicit tags are unsupported;
- the expanded value is an acyclic JSON-compatible tree of maps, lists, strings,
  Booleans, nulls, and numeric values; and
- every numeric leaf satisfies §5.2, including leaves nested inside `map`, `list`, or
  `meta`.

Loaders MUST detect source-level duplicates before constructing an ordinary host
map; “last value wins” and “first value wins” are nonconformant. The exact pre-schema
load codes are `duplicate_key`, `non_string_map_key`, `unsupported_yaml_feature`,
`non_json_value`, `invalid_unicode`, `invalid_numeric_syntax`,
`invalid_boolean_syntax`, `invalid_null_syntax`, and `numeric_value_out_of_range`.
A later JSON Schema or semantic failure uses that layer's code instead.

Every string in a bundle, binding, envelope, or logical state MUST be a sequence of
Unicode scalar values. Unpaired UTF-16 surrogates are `invalid_unicode`, including when
introduced through an escaped JSON/YAML string. Code points are preserved exactly:
the core performs no NFC, NFD, case, or newline normalization. String equality compares
code-point sequences. Every specified byte ordering and every hash encodes those code
points as strict UTF-8.

For a machine source, invalid Unicode is the pre-schema `invalid_unicode` load error.
Programmatic values crossing `create` or `dispatch` are checked recursively before CEL,
state mutation, or identity hashing:

- invalid `root_instance_id` or `creation_id` rejects creation with
  `invalid_creation_request`;
- an invalid creation-binding map key or nested string value rejects creation with
  `invalid_binding`;
- an invalid envelope event name or `event_id` rejects with `invalid_event`;
- an invalid `correlation_id` rejects with `invalid_correlation`;
- an invalid target identity/reference string rejects with `invalid_instance_target`;
- an invalid payload key or nested string value, including reserved `env.changed`,
  rejects with `invalid_payload`; and
- an invalid string anywhere in supplied prior state rejects with
  `invalid_prior_state`.

These are pre-step rejections with no counter, state, fault, or emission change.
Author CEL literals are bundle source and must also pass CEL parsing; every runtime
CEL string value is restricted to the same scalar-value domain.

Portable YAML numeric scalars use exactly this JSON number grammar:

```text
-?(0|[1-9][0-9]*)(\.[0-9]+)?([eE][+-]?[0-9]+)?
```

The `+` metacharacters above denote repetition in grammar notation; a leading plus sign
is not an accepted numeric sign. Hexadecimal, octal, underscores, leading `+`, leading/trailing decimal
points, `.inf`, and `.nan` are `invalid_numeric_syntax`, even when a YAML library would
assign them a numeric core tag. A quoted token is a string. An accepted token without a
fraction/exponent is an `int`; a token with either is a `double`. Thus `-0` is integer
zero, `1e0` is double `1.0`, and `-0.0` normalizes to positive double zero under §5.2.

Number, Boolean, and null resolution is one pre-schema pass over the source spelling
of every plain scalar token, including an empty mapping or sequence value position.
This pass MUST occur before library tag coercion can erase the original spelling. The
portable resolution is exact:

- plain `true` and `false` are Booleans;
- plain `True`, `TRUE`, `False`, and `FALSE` are `invalid_boolean_syntax`;
- plain `null` is null;
- plain `Null`, `NULL`, `~`, and an empty scalar are `invalid_null_syntax`;
- plain `yes`, `no`, `on`, `off`, `y`, and `n`, including their ASCII case variants,
  are strings and remain valid identifiers where an identifier is allowed;
- plain `1` and `0` are integers under the numeric grammar above; and
- every quoted form is a string, including quoted Boolean-like, null-like, and numeric
  tokens.

A loader MUST apply these source-token rules directly rather than accepting its YAML
library's implicit scalar tags and attempting to reconstruct spelling afterward.

> **Non-normative authoring note:** Quote YAML-1.1 Boolean-like identifiers such as
> `yes`, `no`, `on`, and `off` when a bundle may pass through YAML 1.1 tooling. This
> improves interoperability with nonconforming intermediary tools; it is not an
> additional §5.1 identifier restriction.

Every document MUST carry the YAML/JSON integer:

```yaml
format: 1
```

`format` is required. Omission, a string value, or any number other than `1` is
`unsupported_format`. One document is one self-contained bundle; embedded or mixed
formats are invalid.

Format 1 is still pre-release and has no compatibility commitment. Earlier draft
documents, schemas, snapshots, and implementation behavior MAY become invalid while
the format is being designed. There are no legacy aliases, implicit conversions, or
dual parsers in this alpha. Compatibility rules begin only when a format is explicitly
published as stable.

The document `format` is independent of:

- repository/package SemVer (`VERSION`);
- an author's machine `version`; and
- independently versioned hosts, plugins, and the umbrella launcher.

Identifiers MUST match `^[A-Za-z_][A-Za-z0-9_]*$`. Namespaces are dot-separated
identifiers. Public fields use unabbreviated names except for the deliberately retained
keywords `init`, `lang`, `meta`, `on_events`, and `transition_to`.

## 3. Vocabulary

- **Bundle** — one document containing shared event contracts and one or more machine
  definitions.
- **Machine definition** — a named statechart inside a bundle, identified by
  `(namespace, machine_id, version)`.
- **Runtime** — one logical execution of a machine definition with configuration,
  variables, lifecycle status, and identity.
- **Root ownership aggregate** — one root runtime together with every retained
  lifecycle-bound component and owned spawned descendant.
- **State configuration** — the active leaf plus all active ancestors in one runtime.
- **Variable** — typed extended state declared on a state and scoped to that state and
  its descendants.
- **Envelope** — one immutable occurrence of an event, with identity, target, payload,
  and optional correlation.
- **Run-to-completion step** — atomic processing of one envelope for one target runtime
  from one stable aggregate state to the next.
- **Emission** — an immutable internal envelope or external output intent returned by
  the core after a successful step.
- **Queue plugin** — host infrastructure that chooses which envelope to present next
  and what to do with handled, unhandled, rejected, or faulting deliveries.
- **External peer** — anything outside the aggregate that communicates only through
  declared public events.

The core recognizes exactly four statechart relationships:

1. **Inline/nested state** — one hierarchy, configuration, and variable scope.
2. **Lifecycle-bound component** — a statically placed isolated runtime created and
   disposed with a containing `parallel` state.
3. **Owned spawned instance** — a dynamically created isolated runtime owned by the
   root aggregate.
4. **Independent external peer** — outside engine state and lifecycle.

No relationship permits direct cross-runtime variable access or a transition outside
the executing runtime's state boundary.

## 4. Bundle grammar

### 4.1 Minimal bundle

```yaml
format: 1
namespace: example.turnstile

events:
  coin:
    direction: input
    payload:
      amount: { type: int, required: true }
  push:
    direction: input

machines:
  - machine_id: turnstile
    root:
      type: composite
      initial: { transition_to: locked }
      states:
        locked:
          on_events:
            coin:
              guard: "event.payload.amount >= 100"
              transition_to: unlocked
        unlocked:
          on_events:
            push: { transition_to: locked }
```

See `examples/minimal.yaml` and `examples/full.yaml`.

### 4.2 Top-level fields

- `format` — required integer `1`.
- `namespace` — required dotted identifier.
- `events` — optional shared event declarations.
- `machines` — required non-empty ordered list of machine definitions.
- `meta` — optional opaque annotations ignored by core execution.

### 4.3 Machine fields

- `machine_id` — required and unique within the bundle.
- `version` — positive signed-64-bit integer, optional, default `1`.
- `languages` — optional `{guard, action}` language identifiers.
- `events` — optional private event declarations; every machine-local event is
  `internal`.
- `root` — required outermost state node.
- `meta` — optional opaque annotations.

All machine references resolve inside the same bundle. Package imports, dependency
constraints, visibility across packages, and version resolution are unsupported.

In format 1, `languages.guard` is exactly `cel` and `languages.action` is exactly
`determa`; those are also the defaults when omitted. A transition `lang`, when present,
is exactly `cel`. Other executable languages are unsupported in the portable core.

### 4.4 Event declarations

An event declaration is:

```yaml
payment_requested:
  direction: output
  payload:
    order_id: { type: string, required: true }

payment_succeeded:
  direction: input
  correlates_to: payment_requested
  payload:
    provider_id: { type: string, required: true }
```

`direction` is `internal`, `input`, or `output` and defaults to `internal`.

- Bundle `input` events may cross from a host into a root or spawned runtime.
- Bundle `output` events may leave the ownership aggregate as output intents.
- Bundle `internal` events are shared contracts but cannot cross the host boundary.
- Machine-local events are private and MUST be `internal`.

Payload fields use `string`, `int`, `float`, `bool`, `map`, or `list`. A field with
`required: true` must be present and cannot also declare `default`. An optional field
may declare a correctly typed literal `default`; an optional field without a default
remains absent when omitted. Extra fields are invalid.

Payload validation materializes defaults into the normalized immutable envelope before
any guard, handler action, capture, or delivery can observe it. For a host envelope,
normalization occurs after structural/type validation and before handler search. For an
author `send`, payload expressions are evaluated first and defaults are then
materialized before the immutable emission and its target are recorded. A declared
default is type-checked at bundle load. Rejected host input remains the caller's
unchanged original value.

Numeric payload values and defaults use the single portable normalization rule in
§5.2. In particular, a `float` field deliberately accepts either an integer or
floating-point numeric value and exposes one normalized CEL `double`.

`correlates_to` is valid only on a bundle `input` event and MUST name a bundle `output`
event. Success, rejection, failure, cancellation, or no-response outcomes from an
external effect become machine behavior only through later declared input events.

The names `env`, `done`, `determa.component_completed`,
`determa.component_failed`, and `determa.spawned_instance_failed` are reserved and
cannot be author declarations.

These names have closed built-in payload declarations available to the CEL checker:

```text
env:
  changed: {
    each_root_external_variable?: its_declared_type
  }

done:
  relationship: string
  state_path?: string
  owner_runtime_id?: string
  instance?: instance_reference
  instance_id?: string
  machine_id?: string
  machine_version?: int

determa.component_completed:
  component_id: string
  component_runtime_id: string

public_fault_record:
  runtime_id: string
  cause_id: string
  code: string
  step_sequence: string
  source_locator: string

determa.component_failed:
  component_id: string
  component_runtime_id: string
  fault: public_fault_record

determa.spawned_instance_failed:
  instance: instance_reference
  instance_id: string
  machine_id: string
  machine_version: int
  fault: public_fault_record
```

The `env.changed` record is specialized per target machine: every root external
variable is a declared optional field and no other field exists. Runtime validation
requires at least one field. The `done.relationship` value is exactly `parallel` or
`spawned_instance`. A parallel payload materializes exactly `relationship`,
`state_path`, and `owner_runtime_id`; the spawned-instance payload materializes exactly
`relationship`, `instance`, `instance_id`, `machine_id`, and `machine_version`. All
other optional fields remain absent. This single typed record lets a guard select the
relationship before an action reads a relationship-specific field.

All fields without `?` are required and present. Built-in payloads use the same
field-presence semantics as author payloads. In `public_fault_record`,
`step_sequence` is the `canonical_decimal` string projection of the aggregate's
unbounded mathematical counter; the retained logical-state fault record keeps the
counter as a mathematical integer.

### 4.5 Variables

Variables are declared inside state nodes:

```yaml
variables:
  order_id: { type: string, input: true }
  attempts: { type: int, init: 0 }
  provider_region: { type: string, external: true }
  worker:
    type: instance_reference
    machine_id: payment_worker
    nullable: true
    init: null
```

Variable fields:

- `type` — required scalar/container type or `instance_reference`.
- `init` — typed literal used on entry or as a missing creation-binding default.
- `input: true` — permits a typed creation binding.
- `external: true` — declares a host-source value copied into machine state.
- `nullable: true` — permits null for an `instance_reference`; it is invalid on every
  other type.
- `machine_id` — optional nominal constraint for `instance_reference`.

A declaration cannot be both `input` and `external`. Both flags are valid only on root
variables, which makes creation and refresh bindings unambiguous.

Variable names and declared payload-field names enter typed CEL records and MUST be
CEL-visible identifiers. They cannot be either Determa activation name `event` or
`owner`, any CEL keyword `false`, `in`, `null`, or `true`, or any CEL v0.25.2 reserved
identifier `as`, `break`, `const`, `continue`, `else`, `for`, `function`, `if`,
`import`, `let`, `loop`, `package`, `namespace`, `return`, `var`, `void`, or `while`.
The schema rejects these names wherever a variable or declared payload field is named,
including corresponding assign, refresh, send-payload, and binding keys. This
restriction does not apply to machine, state, component, event, or metadata keys
because those names do not themselves become CEL identifiers or typed record fields.

- An ordinary variable, with neither flag, MUST declare a correctly typed `init`.
- A root `input` or `external` variable without `init` requires a creation binding.
- A root `input` or `external` variable with `init` uses the supplied binding when
  present and otherwise uses `init`.
- Missing, extra, or wrongly typed host creation bindings reject `create` with
  `invalid_binding`. The equivalent component/spawn author-binding mismatch rejects the
  bundle at load with `invalid_binding`.

External variables are read-only to `assign`; they change only through a successful
`env`/`refresh` step. An `instance_reference` cannot be `input` or `external`, cannot be
constructed from an arbitrary string, and in format 1 MUST be declared
`nullable: true` with `init: null`. Non-null initial instance references are therefore
unsupported; only the engine writes a non-null value through `spawn.bind_to`.

At the logical-state boundary, a non-null `instance_reference` serializes as exactly:

```text
{
  root_instance_id: non_empty_string,
  instance_id: non_empty_string,
  machine_id: identifier,
  machine_version: positive_integer
}
```

Equality compares all four fields. A nullable reference may also be `null`. Assigning
any other map or string to an `instance_reference` is invalid.

Variable scope begins when the declaring state is entered and ends after its exit
action. Inner declarations shadow outer declarations. A transition action runs before
source exit (§6.4), so source-scoped variables remain available to it. Entry and exit
actions cannot access the current envelope; a transition action must copy required
event data into a variable whose scope survives the transition. Lifecycle entry and
exit actions may update any lexically visible variable that is still live at that
action. Later actions and outer exit actions observe that tentative value until its
declaring scope is actually destroyed.

### 4.6 State nodes

State types are `simple` (default), `composite`, `parallel`, and `final`.

- A `simple` state has no nested states or components.
- A `composite` state requires `initial` and non-empty `states`.
- A `parallel` state requires at least two `components`; it does not also declare
  nested `states`, `initial`, or history.
- A `final` state may declare only `meta`, `variables`, and `entry`.

Common state fields:

- `variables` — state-scoped declarations.
- `entry`, `exit` — ordered structured action lists.
- `on_events` — event name to transition or ordered transition list.
- `history` — `none`, `shallow`, or `deep`, valid only on composite states.
- `meta` — opaque annotation.

A choice pseudostate is represented by an ordered `choice` branch list. Its object may
contain only `choice` and optional `meta`; it cannot declare `type` or any active-state
field. A choice may appear only as a named child in an active state's `states` map and
must be reached by a compound transition. A bundle machine `root` and an inline
component `root` MUST be active states; they cannot be choice pseudostates.

State paths are relative to the machine root. The reserved literal path `root`
identifies the machine root itself; every other path is the dot-separated sequence of
child-state identifiers below it, without a `root.` prefix. A child state therefore
cannot be named `root`.

Native time events (`after`), state-level deferral, orthogonal regions with implicit
broadcast, submachine documents, and completion activities are not part of format 1.

### 4.7 Transitions

A transition may contain:

- `transition_to` — either a target state path or
  `{ history: path.to.composite }`; omit for an internal reaction.
- `guard` — CEL Boolean.
- `action` — ordered structured actions.
- `local: true` — for a non-root composite source targeting its strict descendant,
  preserve the source instead of applying the unmarked transition's source reset
  (§6.4).
- `lang` — optional expression-language override.

Absence of `transition_to` is the only legal spelling of an internal transition.
`internal` is not a format-1 field. A plain state path always performs normal target
entry. The explicit history form requests history restoration. Event transitions and
choice branches may use either target form; an initial transition MUST use a plain
state path. When `local` is present its value MUST be the literal `true`; `false` is
invalid rather than an alias for omission.

An ordered transition list selects the first branch whose guard is true. An
unguarded default MUST be last.

For an event transition, the **source state** is the state whose handler is selected,
even when dispatch began in a deeper active leaf. Its transition action executes once
with that source state's lexical variable scope while the complete pre-exit
configuration is still active. It cannot access a descendant variable that is outside
that lexical scope.

### 4.8 Structured actions

| action | shape | result |
|---|---|---|
| assign | `{ assign: { variable: CEL } }` | one typed write in the executing runtime |
| send | `{ send: { event, to? \| targets?, payload?, correlation_id? } }` | ordered immutable emissions |
| refresh | `{ refresh: { only?: [name, ...] } }` | adopt validated `env` values |
| spawn | `{ spawn: { machine_id, bindings?, bind_to? } }` | create an owned runtime |
| cancel | `{ cancel: { instance: CEL } }` | cancel an addressed owned runtime; null or non-targetable is a no-op |
| stop | `{ stop: {} }` | complete the executing runtime |

`stop` MUST be the final action in its list and is invalid in exit behavior. Once it
executes, any transition target is abandoned and deterministic runtime completion
replaces normal target entry.

`spawn` is invalid in exit behavior. Runtime termination performs owned-descendant
cleanup before author exit actions; permitting a later spawn would orphan the new
child.

Each `assign` action contains exactly one destination. Ordered action lists provide
sequential writes, and a later action observes every earlier tentative write in the
same RTC.

The remaining expression maps never use document member order. One `send` action takes
one state snapshot before evaluating anything, then uses this total order:

1. supplied payload expressions by ascending identifier UTF-8 byte order;
2. `correlation_id`, when present;
3. dynamic `{instance: CEL}` target expressions in target-list order (`to` is a
   one-element list);
4. payload default materialization and numeric normalization;
5. target resolution and eligibility checks in target-list order; and
6. only after every prior operation succeeds, identity/output-counter allocation and
   immutable emission construction in target-list order.

The first expression or eligibility check that fails determines the fault and exact
locator. No later operation runs; the action allocates no identity, counter, or partial
emission. Static targets do not add an expression-evaluation slot.

A component or spawn binding evaluates the `input` map first and the `external` map
second, each by ascending identifier UTF-8 byte order, against one owner-state snapshot
taken before any binding expression in that placement or action. Computed map members
cannot observe one another. The first expression that fails in this order determines
the `action_fault` and its exact pointer; no later expression is evaluated. Target-root
defaults are materialized only after every supplied expression succeeds.

A send target is one of:

```yaml
{ self: true }
{ owner: true }
{ component: component_id }
{ instance: CEL }
{ external: true }
```

`targets` is a non-empty ordered list and is mutually exclusive with `to`. Omitting
both means `self`. One target produces one independent emission. There is no implicit
broadcast.

Internal sends do not recursively dispatch. They are returned to the host in emission
order, and a queue plugin may later present them as new input envelopes.

An internal send to `self`, `owner`, `component`, or `instance` MUST name a bundle or
machine-local `internal` event. A send to `external` MUST name a bundle `output` event
and include `correlation_id`. Input events cannot be sent by author actions. These
event-direction and static target-shape rules are checked at load time.

There is one reserved-event exception: an owner may explicitly forward `env` to one
statically named component placement. That send MUST use exactly
`to: { component: component_id }`, MUST omit `targets` and `correlation_id`, and MUST
have exactly one payload member, `changed`. The `changed` expression MUST be a
non-empty CEL map literal whose keys are literal external-variable names declared by
the target component root. Its values are type-checked against those declarations and
normalized by §5.2. The committed emission carries the target component's specialized
fixed `env` payload from §4.4 and is later delivered explicitly in `internal` mode.
No other reserved event may be sent by author behavior. Dynamic, self, owner, spawned,
external, or multi-target `env` sends are invalid at load. This is explicit forwarding,
not broadcast, and does not permit direct host-to-component ingress.

Target existence is checked when the action executes. `{owner: true}` is valid syntax
in every machine because that machine may run as a contained runtime. If it executes in
an aggregate root, no owner exists and the accepted RTC step faults with
`invalid_instance_target`. An exit action runs after automatic component and
owned-instance cleanup; a send from that exit action to a component or instance already
disposed by the cleanup faults deterministically with `inactive_component_target` or
`invalid_instance_target`, respectively. Such sends are not cleanup no-ops.

## 5. Static validation and CEL

### 5.1 Load-time validation

After the §2 source checks and JSON Schema validation succeed, semantic validation
returns one exact load code. A rejection below that names a code in parentheses uses
that named code. An unavailable CEL-profile symbol or overload uses
`cel_profile_error`. Every other rejection in this section, including CEL
parse/name/type errors, uses the generic `semantic_validation` code.
`structural_validation` is conformance-harness notation for rejection by JSON Schema;
it is not an engine/spec error code.

A bundle is rejected before any runtime is created when it has:

- duplicate machine, component, state, variable, or event identities;
- unresolved machine, state, event, or variable references;
- unreachable states;
- an initial transition targeting anything outside its owning composite;
- a reachable composite without an initial transition;
- a guard or trigger on an initial transition;
- an `input` or `external` variable outside a machine root;
- a variable declaration that cannot obtain a value under §4.5;
- a fraction/exponent-form `double` source value where an `int` is required, including
  machine `version`, variable `init`, and payload `default`;
- an unguarded branch before a later branch;
- a cycle in state nesting or component placement;
- a cycle in the complete synchronous-initialization dependency graph. Its nodes are
  machine definitions and inline component-root definitions. It has an edge for every
  referenced or inline component placement, plus every `spawn` that can execute during
  initialization through an entry action, initial-transition action, or any reachable
  initialization choice branch. All possible guarded choice branches contribute
  edges. Event-handler spawns do not add an edge from the handler's machine, but every
  possible spawn target's own initialization graph must still be acyclic;
- an invalid public/private event direction or correlation;
- a send whose target, event direction, or correlation is inconsistent;
- `local: true` without a target, with a non-composite source, or with a target that is
  not a strict descendant of its source;
- `local: true` on a transition selected on the machine root
  (`root_local_transition`);
- a transition selected on the machine root whose target is the machine root, or any
  history target that resolves to the machine root (`root_reentry`);
- a history target naming a non-composite state, a composite whose `history` is `none`,
  or a composite that strictly contains the transition source;
- an `assign` or `refresh` in an originating event/initial/choice transition action
  whose destination variable belongs to a state exited by that selected outcome
  (`destroyed_variable_write`);
- a spawn `bind_to` in an originating event/initial/choice transition action whose
  destination reference belongs to a state exited by that selected outcome
  (`destroyed_reference_binding`);
- a component/spawn binding with a missing or extra name, or whose inferred expression
  type cannot satisfy the target root declaration (`invalid_binding`);
- invalid `instance_reference` constraints;
- an initial/choice cycle that cannot reach a stable state;
- a transition inside entry, exit, or initial behavior;
- a `spawn` action in exit behavior;
- a `stop` action in exit behavior;
- a `stop` action that is not last in its action list; or
- a CEL parsing, name-resolution, or type-checking failure.

For the destroyed-destination rules above, exits performed by root, component, or
spawned-runtime completion after reaching a final state or executing `stop` belong to
the same originating transition/action outcome. Lifecycle entry and exit action lists
are deliberately different: they may update still-live lexical variables for later
lifecycle actions or outer exits before actual destruction. A `stop` reached from an
entry action follows this lifecycle rule.

### 5.2 CEL environments and compile-time types

Every CEL expression MUST be parsed and type-checked at bundle load with the exact
variables, current event schema, owner binding environment, and expected result type
for its location.

The activation environment is closed:

- Bare identifiers are exactly the variables lexically visible from the expression's
  source state, with the nearest declaration winning under §4.5 shadowing. They expose
  the tentative values as of that action or guard's defined snapshot.
- `event` exists only in the two §5.3 locations. It is a record with exactly one field,
  `payload`, whose typed fields are the materialized immutable payload declaration.
  Event name, `event_id`, target, correlation id, and transport metadata are not
  author-visible.
- `owner` exists only while evaluating a component placement's `with` bindings. It has
  exactly one field, `variables`. That field is a typed record of the variables
  lexically visible from the containing parallel state after its variables and entry
  actions have run, with nearest-declaration shadowing and the one-snapshot behavior
  from §4.8.
- Component `with` expressions do not also receive bare owner-variable names or
  `event`. Spawn bindings use the ordinary bare lexical variables of the executing
  action and do not receive `owner`.

No other activation name or record field exists. In particular, engine state,
configuration, runtime identities, counters, host data, and plugin objects cannot leak
through a CEL library's ambient activation.

- A guard must infer `bool`.
- An `assign` expression must be assignable to the destination variable.
- Send payload expressions must satisfy the target event payload declaration.
- `correlation_id` must infer `string`.
- Component and spawn bindings must satisfy the target root input and external
  declarations.
- Instance targets and cancellation expressions must infer `instance_reference`.
- A dynamic value cannot flow into a concrete destination without an explicit checked
  conversion.

Portable numeric types are exact:

- `int` is a signed 64-bit mathematical integer. A literal or host value outside
  `-9223372036854775808` through `9223372036854775807` is invalid.
- `float` is a finite IEEE 754 binary64 value exposed to CEL as `double`. A document
  literal declared for a `float` may use integer, fractional, or exponent form and is
  converted from its exact mathematical lexical value. A runtime value supplied to a
  `float` destination may be a signed-64-bit CEL/host integer or a finite binary64
  value. Conversion uses round-to-nearest, ties-to-even; negative zero is normalized
  to positive zero. A literal that overflows binary64, NaN, and infinities are invalid.

This is the only implicit numeric widening: `int` never accepts a `float`. The rule is
applied before a value becomes observable for variable `init`, payload `default`, root
creation, component/spawn bindings, `env` changed values, author `assign`, and
host/authored payloads. Send and binding expressions may therefore infer `int` for a
declared `float` destination; the resulting value is normalized before it enters state
or an immutable envelope.

Within a `map` or `list`, a YAML/JSON integer-form numeric literal is a signed-64-bit
CEL `int`, while a fractional or exponent-form literal is a finite normalized CEL
`double`. Implementations MUST preserve that distinction while parsing. Every nested
numeric leaf must satisfy the domains above, but map/list members do not receive
destination-free integer-to-double coercion.

#### Portable CEL profile

Format 1 pins parsing, checking, and base evaluation to
[CEL specification v0.25.2](https://github.com/google/cel-spec/tree/v0.25.2), narrowed
by the deterministic profile below. An implementation's library version is irrelevant;
the accepted program and result MUST match this profile.

The profile contains only:

- `null`, `bool`, signed-64-bit `int`, binary64 `double`, Unicode `string`,
  `list(dyn)`, `map(string, dyn)`, typed event/owner records, and nominal
  `instance_reference`;
- literals, field/map selection, list/map indexing, `?:`, `!`, `&&`, `||`, `in`,
  same-type equality/comparison, checked numeric `+ - * / %`, string/list `+`,
  `size(value)`, `has(map.field)`, and `has(event.payload.declared_field)`;
- `double(int)`, `int(double)`, and `string(bool|int|double|string)` conversions; and
- equality/inequality between compatible `instance_reference` values or with null.

No other function, macro, receiver method, protobuf/object construction, `uint`, bytes,
timestamp, duration, optional type, regex, comprehension, iteration, random/time/I/O
function, host callback, or implementation extension is available. An unavailable
symbol or overload is `cel_profile_error` at bundle load. `instance_reference` cannot
be constructed, inspected by field, ordered, or converted to string.

Evaluation is exact:

- `&&` and `||` use CEL's commutative, error-absorbing result semantics. Either operand
  may be evaluated first because profile expressions are pure. For `&&`, `false`
  absorbs an error from the other operand; for `||`, `true` absorbs an error from the
  other operand. An error is returned when the other operand does not uniquely
  determine the Boolean result. Consequently `false && error` and `error && false`
  are `false`, while `true && error` and `error && true` are errors; `true || error`
  and `error || true` are `true`, while `false || error` and `error || false` are
  errors;
- `condition ? selected : unselected` evaluates the condition and exactly one selected
  branch; the unselected branch is not evaluated;
- signed-integer overflow is an evaluation error; integer division truncates toward
  zero, remainder has the dividend's sign, and division by zero is an error;
- double operations use binary64 round-to-nearest, ties-to-even, and any non-finite
  result is an evaluation error;
- mixed `int`/`double` operators are rejected at load; destination widening under the
  earlier rule occurs only after expression evaluation;
- `int(double)` truncates toward zero and errors outside signed-64-bit range;
  `double(int)` uses §5.2 rounding; `string` produces `true`/`false`, canonical decimal
  integer text, or the RFC 8785 finite-number spelling with negative zero already
  normalized;
- list equality is ordered and recursive; map equality ignores member order and
  compares the exact string-key/value set recursively; dynamic numeric members remain
  type-sensitive;
- a negative/out-of-range list index or missing map key is an evaluation error;
  `has(map.field)` tests exact string-key presence without reading an absent value;
  `has(event.payload.field)` is false only when that declared optional field has
  neither a supplied value nor a default, and true when it was supplied or materialized
  from a default. A required payload field is always present after envelope validation.
  Selecting an absent optional payload field without first taking a branch that proves
  its presence is an evaluation error. No other typed-record presence macro is
  available; and
- string size counts Unicode scalar values, string equality performs no
  normalization, and string relational operators compare lexicographically by Unicode
  scalar value.

An incompatible expression is an invalid document, not a runtime `type_fault`. A valid
bundle plus a valid input envelope MUST NOT discover an ordinary assignment or payload
type mismatch during execution.

Runtime CEL evaluation can still fail for value-dependent reasons such as division by
zero, invalid indexing, or an explicit conversion failure. A guard evaluation error is
`guard_fault`; another expression evaluation error is `action_fault`.

### 5.3 Lifecycle expression visibility

The author-visible `event` binding exists only while evaluating:

- an `on_events` guard selected for the envelope; and
- that directly selected event transition's actions.

It does not exist in:

- state entry actions;
- state exit actions;
- initial-transition actions;
- choice guards or actions;
- component `with` bindings;
- completion/cancellation cascades; or
- root creation.

The engine retains causal identity internally for deterministic tracing and emission
identity, but lifecycle CEL cannot inspect the triggering envelope or its payload.

## 6. Event and transition semantics

### 6.1 Input envelope

An envelope is:

```text
{
  event: event_name,
  event_id: non_empty_string,
  target:
    { root: {
        root_instance_id: non_empty_string,
        root_runtime_id: non_empty_string
    }}
    | { spawned_instance: instance_reference }
    | { component: {
        root_instance_id: non_empty_string,
        owner_runtime_id: non_empty_string,
        component_id: identifier,
        component_runtime_id: non_empty_string,
        activation_sequence: non_negative_integer
    }},
  payload: typed_map,
  correlation_id?: non_empty_string
}
```

This tagged union is the normalized immutable target. Author target shorthands from
§4.8 resolve to one union member when an emission is created; the resulting envelope
stores the complete member and never resolves it again. A component target therefore
names one activation incarnation, not merely a reusable `component_id`.

The caller owns the envelope. The core neither enqueues nor stores it. A processing
call carries the explicit delivery mode from §8. Before starting a step, the core
atomically validates that mode, event declaration, direction, payload, correlation,
and target eligibility. Rejection returns the prior state unchanged and allocates no
logical identity.

`input` mode MUST name a bundle `input` event and may supply only the `root` or
`spawned_instance` target member for a running runtime. Direct use of the `component`
member rejects with `invalid_instance_target`. If the declaration has `correlates_to`,
the envelope MUST carry a non-empty `correlation_id`. Reserved `env` is the exception
below.

`internal` mode MUST name a bundle or machine-local `internal` event, one fixed
reserved lifecycle event, or the statically targeted component `env` emission defined
by §4.8, and may carry any eligible target member. The core does not
authenticate how the caller obtained an envelope; the host/queue adapter is responsible
for using `internal` mode only for an immutable core emission it is delivering.
Using an input event in `internal` mode, an internal/reserved event in `input` mode, or
an output event in either mode rejects with `invalid_event`.

A completed or faulted root, and a faulted spawned runtime, reject delivery with
`invalid_instance_target`. A completed or faulted component rejects internal delivery
with `inactive_component_target`; it cannot accumulate work that will never run.
Rejection is atomic. Calling `dispatch` with null delivery is not a processing
operation; read-only inspection of a terminal aggregate returns its existing status
and no emissions.

Aggregate-root fault terminality overrides descendant status. Once the aggregate root
is faulted, every non-empty dispatch targeting the root, a component, or a spawned
descendant is rejected atomically with `invalid_instance_target`, even when the
descendant's retained diagnostic status was running. The aggregate, counters, and
emissions remain unchanged. A null-delivery read-only inspection returns that exact
faulted aggregate and no emissions.

Reserved `env` is the only undeclared host-input exception:

```text
{
  event: "env",
  event_id: non_empty_string,
  target:
    { root: {
        root_instance_id: non_empty_string,
        root_runtime_id: non_empty_string
    }}
    | { spawned_instance: instance_reference },
  payload: { changed: { external_name: typed_value, ... } }
}
```

It has no correlation id. `changed` is non-empty and every field must match a declared
root external variable on the targeted runtime. A `refresh` action is valid only in the
selected `env` handler and copies the requested changed values into those root
variables atomically. `refresh: {}` selects every field in the current `changed` map;
`refresh.only` selects exactly its named subset and every name MUST occur in that map.
If a selected `refresh.only` name is absent from the accepted `changed` record, the
action raises `action_fault` at the exact absolute pointer formed by appending
`/refresh/only/<index>` to the action's pointer in the validated bundle.
The ordinary RTC fault rule rolls back every write, emission, counter allocation, and
other tentative change from the step. This is not a pre-step rejection: the envelope
is structurally and semantically valid, and the failure depends on the selected
handler and action. The first absent item in ascending list-index order determines the
fault. An omitted `only` never has this failure because it selects exactly the names
present in `changed`.
Changed fields not selected by a committed `refresh` are not retained by the core.
There is no external-source map in logical state: the normalized `env` envelope is the
only source value visible during that RTC step.

For an aggregate root or spawned runtime, `env` arrives only through the `input` mode
exception above. For a component, it arrives only through the owner-to-static-component
forwarding form in §4.8 and subsequent explicit `internal` delivery. Direct host input
to a component remains invalid.

### 6.2 Run to completion

One `dispatch` call processes at most one accepted envelope for one runtime. The step is
non-reentrant and atomic. Emissions are accumulated but are invisible to every runtime
until the step commits and the host submits them to a queue or external adapter.

Other runtimes may be processed concurrently only when the host's persistence layout
provides serializable ownership of any aggregate state they might share. Two calls MUST
NOT concurrently mutate the same root ownership aggregate.

### 6.3 Hierarchical dispatch

The envelope is offered to the deepest active state. If no enabled handler is selected,
it is offered to that state's parent, recursively.

- A false guard does not consume the envelope; ancestor search continues.
- A guard evaluation error faults the step.
- The first true branch in an ordered list wins.
- If no state handles the envelope, the result disposition is `unhandled`.

The only exception is an unhandled `determa.component_failed` or
`determa.spawned_instance_failed` envelope, which faults the owner as specified in
§10.2.

Determa deliberately gives ordered guard branches declaration-order priority: the
first true branch wins, and guards need not be mutually exclusive. UML state-machine
models do not assign this priority to competing guarded transitions. Authors porting a
UML model MUST NOT assume guard-order independence. An unguarded default, when present,
MUST be last, which keeps the priority unambiguous. Without a default, all-false guards
continue ancestor search.

An unhandled result is not a core fault. The core stores nothing and changes no logical
state. The queue plugin decides whether to acknowledge, discard, retry, log, or retain
the envelope. High-volume irrelevant input, such as pointer movement, can therefore be
discarded without accumulating core state.

### 6.4 Transition execution order

Determa deliberately executes a selected external transition in this exact order:

1. evaluate the selected guard;
2. execute the originating transition actions in the source configuration and source
   variable scope, then completely resolve any targeted choice chain as described
   below;
3. exit the active source path from innermost to outermost, stopping below the
   transition boundary defined below;
4. enter the target path below that boundary, from outermost to innermost; and
5. restore explicitly targeted history or follow nested initial transitions until a
   stable leaf is active.

Because step 2 precedes exit, a write to a variable or `bind_to` reference destroyed by
step 3 would have no observable result. §5.1 therefore rejects such a transition at
load time. A write to an ancestor-scoped destination that survives the exit remains
valid. For a choice chain, every possible selected branch must satisfy this rule.
Entry/exit actions invoked by steps 3–5 instead use the lifecycle rule in §4.5.

This differs from canonical UML ordering, which treats the transition effect as behavior
of the edge after source exit and before target entry. Determa instead keeps the source
context intact while the transition action runs. Authors familiar with UML MUST rely on
the order above for Determa definitions.

A transition whose target is a choice forms one **compound transition**:

1. the event or initial transition that first targets a choice is the originating
   transition;
2. after its actions, evaluate that choice's branches in order and select the first
   true guard or final unguarded branch;
3. execute the selected branch actions immediately, before any state exit;
4. if its target is another choice, repeat steps 2–3; otherwise that target is the
   compound transition's final target; and
5. compute one transition boundary from the originating source and final target, then
   perform one exit/entry sequence.

Choice guards and actions use the originating transition's lexical variable scope after
all preceding actions in the chain. They never receive the `event` binding. Every
possible incoming origin/choice path MUST therefore parse, name-resolve, and type-check
in that origin's scope. A guard or action fault rolls back the whole RTC before any
state exit.

For an event transition, the originating source is the state whose handler was
selected. For an initial transition, it is the already-entered containing composite:
the initial and selected choice actions run in that composite's scope, the full chain
resolves before any descendant entry, and no state is exited. Choice-to-choice paths
never enter or exit the transient choice objects themselves.

Entry and exit actions belong to states. Entry initializes the state's variables, runs
its entry actions, then follows its initial transition unless explicit history
restoration supplies the descendant configuration. When a state exits, the engine first
performs the automatic owned-child cleanup defined by §7.2, then runs the state's exit
actions, then destroys its variables.

An internal transition has no `transition_to` and executes actions without exit, entry,
or initial descent. No state is exited or entered, including states between the active
leaf and an ancestor state whose handler was selected; the active configuration is
identical before and after.

For a transition with a target, the **transition boundary** is computed from the
resolved source/target relationship. The root is an invariant boundary: ordinary
transitions never exit or re-enter it.

- A plain self-transition on a non-root source uses the parent of the source as its
  boundary, so it exits and re-enters the source. A root self-transition is rejected
  at load time.
- When a composite source strictly contains the target, an unmarked transition also
  uses the parent of the source as its boundary, except that a root source uses the
  root itself. It exits and re-enters a non-root source, resetting that subtree
  including the source's variables and lifecycle actions. With a root source, only
  active descendants are exited and entered; root variables and lifecycle actions
  remain untouched.
- For that same strict-descendant relationship, `local: true` uses the source as its
  boundary. It exits and enters only descendants, leaving the source's variables and
  lifecycle actions untouched. A machine-root source cannot carry `local: true`;
  root invariance would make it a noncanonical no-op.
- When the target strictly contains the source, the target is the boundary. The active
  source path exits up to but excluding the target, and the target is not re-entered.
  External re-entry of a proper ancestor target is not expressible in format 1.
- For unrelated source and target states, their ordinary least common ancestor is the
  boundary.

For example, let composite `c` contain active leaf `a`, with the handler selected on
source `c` and target `c.a`:

```text
unmarked:    exitA, exitC, enterC, enterA
local: true: exitA, enterA
```

Local self-transitions, local transitions to ancestors, and local transitions between
unrelated states are deliberately unsupported.

### 6.5 Choice and history

A choice is transient and follows the compound-transition algorithm in §6.4. The first
true branch is selected and the final branch MUST be unguarded.

Each composite whose `history` is `shallow` or `deep` maintains an optional history
record. When an RTC exits that composite, the engine copies its pre-exit active
descendant configuration immediately before the first exit action in the composite's
subtree. The copy becomes the new history record only if the RTC commits. A transition
that passes through or changes descendants without exiting the composite does not
update its record.

Shallow history records only the active direct substate. Deep history records the full
active descendant configuration. A plain `transition_to: path.to.composite` always
restarts that composite through its `initial` transition, even when a history record
exists. `transition_to: { history: path.to.composite }` enters the composite and:

- restores the recorded direct substate for shallow history, then follows that
  substate's normal initial descent;
- restores the recorded descendant path for deep history; or
- follows the composite's `initial` transition when no record exists.

Entry actions run and state-scoped variables are initialized for every restored state,
outermost to innermost. History restores configuration, not destroyed state-scoped
variable values. A choice branch may select history using the same object form; an
initial transition cannot target history.

A transition from a non-root composite source to its own history is allowed without
`local: true`. It uses the plain self-transition boundary: the engine captures the
pre-exit configuration, exits and re-enters the composite, and restores the same
descendant configuration. Lifecycle actions rerun and state-scoped variables are
reinitialized even though the final active leaf is unchanged. A history target that
resolves to the machine root is rejected.

A history target may carry `local: true` when the resolved history composite is a
strict descendant of the composite source. The local boundary from §6.4 applies, then
history restoration occurs normally in step 5.

### 6.6 Stop interruption

`stop` is an immediate interruption point for the runtime executing it:

- in a transition or choice action, it abandons the transition/choice target and skips
  the remaining choice chain;
- in an entry action, it skips the rest of that runtime's entry/initial descent;
- in an initial-transition action, it abandons the initial target; and
- during component or spawned initialization, it completes that contained runtime,
  not its owner.

Actions and emissions performed before `stop` remain tentative results of the RTC.
When `stop` occurs in lifecycle entry behavior, preceding entry writes and later exit
writes to still-live variables use §4.5 and may feed subsequent exit behavior before
destruction.
The engine then performs that runtime's ordinary completion: cancel retained
descendants, dispose allocated-but-not-initialized component placements without running
their author behavior or emitting component notifications, exit the currently entered
partial configuration deepest-first, and finalize completion. Identity/counter
allocations made before `stop` remain consumed if the RTC commits.

If a parallel owner's entry action stops its runtime, component identities allocated
before entry are disposed and no component binding or initialization runs. If a
component stops during its own initialization, it emits its normal component-completed
notification and later placements continue in declaration order. If a spawned child
stops during its initialization, it emits normal spawned-instance `done`, is disposed,
and any `bind_to` value remains the resulting non-targetable nominal reference; later
owner actions continue.

Any cleanup or exit fault rolls the enclosing RTC back and uses the normal
fault-finalization rules. Thus retry starts from the same pre-step state, and no skipped
initialization or emission leaks from the failed attempt.

## 7. Components, spawning, and lifecycle

Every lifecycle cascade uses one recursive postorder algorithm:

1. At a runtime, direct retained component children are visited first in descending
   order of `(owning_state_document_pointer UTF-8 bytes,
   state_activation_sequence, component_declaration_index,
   component_activation_sequence)`.
2. Direct retained spawned children are then visited in ascending order of
   `(holder_rank, holder_declaration_pointer UTF-8 bytes,
   holder_state_activation_sequence, spawn_sequence)`. A bound child has
   `holder_rank = 0` and uses the exact RFC 6901 variable-declaration pointer plus the
   activation sequence of that declaring state. An unbound child has
   `holder_rank = 1`, empty holder pointer, and holder activation sequence zero.
3. Each selected child recursively visits its children by the same rules before that
   child is finalized. A running child then executes active exits innermost through its
   root; a completed child has no active configuration; a retained-faulted or
   root-frozen subtree skips all author exit behavior. The finalized subtree is
   disposed before the next sibling is visited.

The component tuple's declaration index is its zero-based index in `components`; its
state pointer/activation distinguishes repeated or nested placements. A runtime's
spawn sequence is unique and therefore breaks any remaining spawned-child tie. Already
disposed children are absent. Emissions append exactly when their exit action runs, so
the traversal above is also the total cascade-emission order.

State-scope cleanup selects only children bound to variable declarations whose
currently active state scopes are exiting; unbound children and children held by
surviving scopes are not selected. Runtime completion, `stop`, and root termination
select every direct component and spawned child, including unbound children. Explicit
`cancel` selects its addressed spawned subtree. Parallel-state exit selects every
placement of that parallel state. Natural component/spawned completion applies the
same traversal to that runtime's descendants. These selection rules plus the traversal
are reused without variation for explicit cancel, holder cleanup, component disposal,
natural completion, `stop`, and root cascade. A fault rolls the entire enclosing
cascade back atomically.

### 7.1 Lifecycle-bound components

A parallel state declares at least two isolated placements:

```yaml
processing:
  type: parallel
  components:
    - component_id: fulfillment
      machine_id: fulfillment
      with:
        input:
          order_id: "owner.variables.order_id"
    - component_id: accounting
      machine_id: accounting
      with:
        input:
          order_id: "owner.variables.order_id"
```

A placement declares exactly one of `machine_id` or inline `root`. `component_id` is
unique across the containing machine.

An inline `root` uses the containing machine's `(namespace, machine_id,
machine_version)` as its definition identity. Its exact component placement pointer is
the additional inline-definition discriminator used by §9. Nested inline placements use
their own full document pointers, so neither runtime nor effect identities can collide
with a sibling placement.

Entering a parallel state is part of the owner step:

1. allocate component runtime identities in declaration order and mark each placement
   pending-initialization;
2. initialize the parallel state's variables and run its entry actions;
3. evaluate the statically validated `with.input` and `with.external` expressions in
   the order and snapshot defined by §4.8, using owner variables only, then apply
   target-root `init` defaults;
4. create and initialize each component in declaration order; and
5. reach a stable configuration in every component before the owner step commits.

`pending-initialization` and `pending-completion` are tentative intra-RTC phases, not
logical-state statuses. They never appear in committed state, results, read-only
inspection, or the input to a later call. Stable committed root statuses are exactly
`running`, `completed`, and `faulted`; stable retained component statuses are exactly
`running`, `completed`, and `faulted`; stable retained spawned statuses are exactly
`running` and `faulted`, because completed spawned runtimes are disposed before commit.

The triggering event is unavailable to entry actions and `with`. A transition action
must first copy required payload into an owner variable.

Components have isolated configurations and variables. They do not share an event queue
because queues are outside core. An event reaches a component only through an explicit
emission targeting its nominal component runtime identity.

The `{component: component_id}` syntax resolves at emission time to the allocated
pending-initialization or running placement identity, including its activation
sequence. This permits the parallel owner's entry action to address identities allocated
in step 1. The immutable envelope stores the complete component target from §6.1.
Delayed delivery to a disposed placement therefore cannot accidentally reach a later
re-entry incarnation with the same `component_id`.

An emission to a pending-initialization placement is tentative and cannot be delivered
before the enclosing owner step commits. If initialization succeeds, later delivery may
target the running component. If the component initializes and immediately completes,
or initialization commits as an isolated component fault under §10.2, the earlier
emission remains in the committed owner result with its exact target, but later delivery
rejects it with `inactive_component_target`. If the enclosing owner step faults, both
the placement and tentative emission roll back.

A component reaching its root final state or executing `stop` becomes completed and
emits one `determa.component_completed` envelope to its owner using the fixed payload
from §4.4:

```text
{ component_id, component_runtime_id }
```

When every component placement is complete, the same successful step also emits the
parallel branch of the fixed reserved `done` payload from §4.4:

```text
{
  relationship: "parallel",
  state_path: dotted_identifier_from_root,
  owner_runtime_id: non_empty_string
}
```

Delivery and ordering relative to unrelated envelopes are queue-plugin behavior.
Within the committed core result, `determa.component_completed` precedes `done`.

For `determa.component_completed`, source is the component runtime and target is its
owner runtime. For the all-components-complete `done`, source and target are both the
owner runtime. Both retain the completing component step's cause.

Completed and faulted component runtimes remain inspectable until their parallel owner
exits, but are terminal and non-targetable. Running components accept internal
delivery. During one tentative owner RTC, an entry action may resolve and emit to an
already allocated pending-initialization identity as defined above; this is not
delivery to a committed pending runtime.

Exiting the parallel state synchronously cleans and disposes its retained component
runtimes with the canonical cascade above, atomically with the owner transition.

Reset-in-place, component history retention, shared variables, implicit broadcast, and
direct host-to-component ingress are unsupported.

### 7.2 Owned spawned instances

`spawn` creates an isolated runtime for a same-bundle machine:

```yaml
- spawn:
    machine_id: payment_worker
    bindings:
      input:
        order_id: "order_id"
      external:
        provider_region: "provider_region"
    bind_to: payment_worker
```

The action allocates a deterministic child identity, evaluates its statically validated
root input and external bindings in the order and snapshot defined by §4.8, applies
target-root `init` defaults, optionally stores its nominal `instance_reference` in
`bind_to`, and initializes the child to a stable configuration. A binding-expression
evaluation error is `action_fault`; missing/extra names and incompatible inferred types
were already rejected at load with `invalid_binding`. Creation and binding are atomic
with the parent step. `bind_to` MUST name a compatible nullable
`instance_reference` whose current value is null; otherwise the step faults with
`binding_not_empty`.

The declaring scope of the `instance_reference` used by `bind_to` defines the bound
child's maximum lifetime. Binding records that declaration as the child's lifetime
holder; copying or later replacing the nominal reference value neither transfers nor
erases that association. When the holder's state exits, every running or
retained-faulted associated child is synchronously cancelled and disposed before the
declaring state's exit actions run in the canonical cascade order above. The cleanup
is atomic with the owner transition.
Emissions from child exit actions are returned in that cancellation order as part of
the owner RTC step. A holder with no live or retained-faulted associated child requires
no cleanup. A cleanup failure rolls the owner RTC step back and finalizes
`cascade_fault` under §10.1. A root-scoped holder therefore lets its child survive every
ordinary transition; a state-scoped holder ties its child to that state's lifetime.

Ownership is not otherwise tied to the transition that spawned the child. An unbound
child or a child whose holding reference remains in scope is processed only when a
queue plugin later presents an envelope targeting it.

After its expression type-checks as `instance_reference`, `cancel` is always
well-formed. If the expression currently addresses a running or retained-faulted
directly or transitively owned instance, it synchronously cancels descendants with the
canonical cascade, runs remaining eligible exit actions, disposes logical child state,
and
invalidates the reference as a target. A retained reference remains serializable and
comparable but no longer addresses a live runtime. Faulted instances accept
cancellation only for this cleanup; they reject ordinary events and sends. Cancellation
of a retained-faulted instance follows the frozen-subtree rule in the canonical
cascade. Every other resolved value succeeds without effect as described below.

If the cancellation expression evaluates to null, or does not currently address a
running or retained-faulted directly or transitively owned instance, the action succeeds
as a no-op. The action itself changes no ownership or counters and produces no fault or
emission; the surrounding RTC step continues normally.

Natural child completion performs the same descendant cleanup and emits the
spawned-instance branch of the fixed reserved `done` payload from §4.4 to its immediate
owner:

```text
{
  relationship: "spawned_instance",
  instance: instance_reference,
  instance_id: non_empty_string,
  machine_id: identifier,
  machine_version: positive_integer
}
```

For this `done`, source is the completed child runtime and target is its immediate owner
runtime. It retains the child's completing cause.

The completed child subtree is then disposed and its `instance_reference` becomes
non-targetable. The completion envelope retains the nominal identity needed by the
owner. A faulted child subtree is instead retained for diagnostics and may be disposed
only by explicit owner cancellation or owner/root cleanup.

Remote provisioning is never core `spawn`. A machine requests it through an external
output intent and receives declared correlated input events from a host extension.

### 7.3 Runtime and aggregate-root completion

When any runtime reaches its root final state or executes `stop`, it synchronously
cancels all retained owned descendants with the canonical cascade, runs its active exit
actions deepest-first, and becomes completed before the RTC commits. Component and
spawned-runtime retention and notifications then follow §7.1 and §7.2.

The resulting emission order is exact:

1. author emissions produced before the completion trigger, including final-state
   entry actions or actions preceding `stop`;
2. descendant cancellation/cleanup emissions in the deterministic cascade order;
3. active-state exit-action emissions, innermost through the runtime root; and
4. after every exit succeeds, the runtime's reserved completion notification.

For a component, step 4 emits `determa.component_completed` and then, when it completes
the containing parallel placement set, the parallel `done`. For a spawned runtime,
step 4 emits spawned-instance `done`. The aggregate root has no reserved completion
notification. Any fault rolls back every emission in this sequence.

Completion exits the runtime root as well as its active descendants. After its root
exit action, every state-scoped variable is destroyed and the completed runtime has an
empty configuration and variable map. A retained completed component therefore exposes
identity, status, history, counters, and prior fault diagnostics but no active
configuration/variables. A completed spawned runtime is disposed as defined by §7.2.

For the aggregate root, the engine retains terminal identity/status, history, counters,
and fault-history diagnostics and returns `completed`; it retains no component or
spawned descendant. No new ordinary envelope may target it.

## 8. Pure foreground interface and logical state

Language APIs may use idiomatic names, but every implementation must provide behavior
equivalent to:

```text
create(bundle, machine_id, root_instance_id, creation_id, bindings)
  -> { status, state, emissions, fault, rejection }

dispatch(bundle, prior_state, delivery?)
  -> { status, disposition, state, emissions, fault, rejection }

delivery =
  { input: envelope }
  | { internal: envelope }
  | null
```

The two delivery members are a closed tagged union. `input` applies the public-ingress
rules in §6.1; `internal` applies the internal-delivery rules. A non-null delivery has
exactly one member. Null is the read-only inspection call.

`status` is `running`, `completed`, or `faulted` for an existing aggregate. A creation
rejected before an aggregate exists returns `status: rejected` and `state: null`.
Dispatch rejection or an unhandled envelope preserves the prior aggregate status.

Every named result field is present. `emissions` is the ordered list returned by this
call. `rejection` is null except on pre-step rejection, where it is exactly
`{ code: rejection_code }`. `fault` is:

- the aggregate root's committed fault record when the aggregate root is faulted;
- otherwise the target runtime fault newly committed by a `faulted` dispatch; or
- null.

A contained fault committed inside an otherwise successful owner RTC appears only in
the contained-runtime state and its reserved failure emission; it does not populate the
top-level `fault`. There is no plural `faults` result field.

Creation rejection codes are exactly `invalid_creation_request`,
`invalid_machine_target`, and `invalid_binding`. Dispatch rejection codes are exactly
`invalid_event`, `invalid_payload`, `invalid_correlation`,
`invalid_instance_target`, `inactive_component_target`, `invalid_prior_state`, and
`incompatible_bundle`. Bundle parsing, schema, and semantic load failures happen before
these calls and use §2/§5 codes. A rejection commits no fault record.

`disposition` is exactly:

- `handled` — the accepted envelope completed a successful RTC step;
- `unhandled` — no enabled handler existed;
- `rejected` — validation failed before an RTC step; or
- `faulted` — an engine fault occurred during the RTC step.

For the null-delivery read-only call defined by §6.1, `disposition` is null. It returns
the unchanged state, current `status` and `fault`, empty `emissions`, and null
`rejection`.

The root ownership aggregate contains:

- the root runtime;
- every retained component runtime;
- every non-disposed owned spawned descendant, including running, retained-faulted, or
  root-frozen diagnostic descendants;
- each runtime's definition identity, configuration, variables, history, lifecycle
  status, and fault records;
- ownership and `bind_to` lifetime-holder associations, plus placement, activation,
  state-entry, spawn, logical-step, and output identity counters; and
- the `validated_bundle_fingerprint` defined below.

It contains no event queue, deferred queue, dead-letter collection, timer, broker
acknowledgement, transport receipt, or plugin configuration.

Before creation, the engine creates one normalized bundle tree. Default
materialization is closed and context-sensitive:

| source context | omitted source member | normalized member |
|---|---|---|
| machine | `version` | `version: 1` |
| machine | `languages` or either child | complete `{ guard: cel, action: determa }` |
| bundle or machine event | `direction` | `direction: internal` |
| payload field | `required` | `required: false` |
| every non-reference variable | omitted `input` / `external` flag | insert that flag as `false` |
| active state | `type` | `type: simple` |
| composite state | `history` | `history: none` |
| event transition in `on_events` | `lang` | `lang: cel` |
| `send` action | both `to` and `targets` | `to: { self: true }` |

Before this table is applied, every typed literal is normalized under §5.2. Thus an
integer-form `init` or payload `default` for a declared `float` becomes a binary64
value in the normalized tree; destination-free numeric values such as `meta` leaves
retain their parsed `int`/`double` distinction.

An `instance_reference` must explicitly declare `nullable: true`; non-reference
variables cannot declare `nullable`, so no normalized `nullable: false` is inserted.
Initial transitions and choice branches have no `lang` member and use their fixed
language semantics without inserting one. An explicitly present value is retained
after §5.2 numeric normalization.

Every other optional member remains absent. In particular, absent `events`,
`variables`, `payload`, `entry`, `exit`, `on_events`, `states`, `components`, `meta`,
binding, correlation, action, guard, `init`, payload `default`, `local`, and
`correlates_to` members are not replaced with empty maps, empty lists, null, or false.
Choice pseudostates do not receive `type`. The normalizer recursively visits inline
component roots and every structured action location. JSON Schema `default`
annotations are informative only; this table is the normative algorithm.

The engine then computes:

```text
validated_bundle_fingerprint = hash([
  "determa-validated-bundle-fingerprint-1",
  typed_bundle_tree
])
```

`typed_bundle_tree` recursively encodes null as `["null"]`, Boolean as
`["boolean", value]`, string as `["string", value]`, integer as
`["integer", canonical_decimal(value)]`, binary64 as
`["float", sixteen_lowercase_hex_bits]`, list as `["list", encoded_items]`, and map as
`["map", [[key, encoded_value], ...]]` with entries sorted by key UTF-8 bytes. Binary64
bits use network byte order after negative-zero normalization. This typed projection
prevents JCS from collapsing `int`/`double` or rounding a signed-64-bit integer. It
includes `meta` and every other validated field.

Normative fingerprint vector for this default-materialized bundle:

```yaml
format: 1
namespace: example.turnstile
events:
  tick:
    payload:
      amount: { type: float, default: 1 }
meta:
  large_integer: 9007199254740993
  integer_one: 1
  floating_one: 1.0
machines:
  - machine_id: turnstile
    events:
      local_notice:
        payload:
          value: { type: int, required: true }
    root:
      type: composite
      variables:
        attempts: { type: int, init: 0 }
      initial: { transition_to: locked }
      states:
        locked:
          type: parallel
          components:
            - component_id: left
              root: {}
            - component_id: right
              root: {}
          on_events:
            tick:
              transition_to: unlocked
              action:
                - send:
                    event: local_notice
                    payload: { value: "1" }
        unlocked: {}
```

```text
JCS:
["determa-validated-bundle-fingerprint-1",["map",[["events",["map",[["tick",["map",[["direction",["string","internal"]],["payload",["map",[["amount",["map",[["default",["float","3ff0000000000000"]],["required",["boolean",false]],["type",["string","float"]]]]]]]]]]]]]],["format",["integer","1"]],["machines",["list",[["map",[["events",["map",[["local_notice",["map",[["direction",["string","internal"]],["payload",["map",[["value",["map",[["required",["boolean",true]],["type",["string","int"]]]]]]]]]]]]]],["languages",["map",[["action",["string","determa"]],["guard",["string","cel"]]]]],["machine_id",["string","turnstile"]],["root",["map",[["history",["string","none"]],["initial",["map",[["transition_to",["string","locked"]]]]],["states",["map",[["locked",["map",[["components",["list",[["map",[["component_id",["string","left"]],["root",["map",[["type",["string","simple"]]]]]]],["map",[["component_id",["string","right"]],["root",["map",[["type",["string","simple"]]]]]]]]]],["on_events",["map",[["tick",["map",[["action",["list",[["map",[["send",["map",[["event",["string","local_notice"]],["payload",["map",[["value",["string","1"]]]]],["to",["map",[["self",["boolean",true]]]]]]]]]]]]],["lang",["string","cel"]],["transition_to",["string","unlocked"]]]]]]]],["type",["string","parallel"]]]]],["unlocked",["map",[["type",["string","simple"]]]]]]]],["type",["string","composite"]],["variables",["map",[["attempts",["map",[["external",["boolean",false]],["init",["integer","0"]],["input",["boolean",false]],["type",["string","int"]]]]]]]]]]],["version",["integer","1"]]]]]]],["meta",["map",[["floating_one",["float","3ff0000000000000"]],["integer_one",["integer","1"]],["large_integer",["integer","9007199254740993"]]]]],["namespace",["string","example.turnstile"]]]]]

hash:
sha256:7e48ad82ea5305c24b7730f4fd24c36ec196a0875c982b85eba5b3a5ddcbb92f
```

Creation stores this fingerprint. Every ordinary dispatch, including read-only
inspection, first validates the abstract prior-state shape and all retained
definition/path references, then compares its stored fingerprint with the supplied
validated bundle. Malformed or internally inconsistent prior state rejects with
`invalid_prior_state`; a fingerprint mismatch rejects with `incompatible_bundle`.
Both return the exact prior state with no counter/state/emission change. This check
precedes envelope validation. A different document reusing the same
namespace/machine/version triple can therefore never reinterpret existing state.

The portable aggregate-state and explicit definition-migration operations in §16 are
separate operations around this dispatch boundary. A host may decode an aggregate
under its exact source definition and explicitly migrate it before dispatch. Ordinary
dispatch itself never chooses, discovers, or applies a migration.

The host may store one aggregate in one row/document or normalize it, provided every
dispatch sees serializable prior state and commits an observably equivalent result.
The core itself performs no persistence.

Creation initializes the aggregate and every root initial/entry action atomically.
Creation bindings contain separate `input` and `external` maps. Missing required,
extra, or wrongly typed values reject creation with `invalid_binding`, no state, and no
emissions. Omitted declarations use `init` only where §4.5 permits a default.

A value-dependent root initialization fault rolls author behavior back and commits a
terminal diagnostic aggregate containing only the validated-bundle fingerprint, root
runtime/definition/creation identity, aggregate counters, `faulted` status, and fault
record. Its configuration, variables, history, components, and owned spawned instances
are empty. Root-local spawn/state/component activation counters are reset to their
pre-author-initialization values: spawn sequence zero and empty state/component counter
maps. Creation used logical step zero, so `next_logical_step_sequence` is one;
`next_output_sequence` is zero because every author output rolled back. The result
contains no emission and null rejection. Contained component and spawned initialization
faults follow the mandatory isolated behavior in §10.2; they are not implementation
choices.

## 9. Deterministic identities and emissions

All identity hashes use:

```text
"sha256:" + lowercase_hex(SHA-256(UTF-8(JCS(value))))
```

where JCS is RFC 8785 canonical JSON.

Logical counters are unbounded non-negative mathematical integers. Hash operands encode
every counter as a canonical decimal JSON string: `0`, or a non-zero digit followed by
digits, with no sign or leading zero.

`root_instance_id` and `creation_id` are non-empty strings supplied by the caller.
`root_instance_id` identifies the aggregate. `creation_id` identifies one logical
creation request and MUST be reused when retrying that request.

The root runtime identity is:

```text
hash([
  "determa-root-runtime-identity-2",
  "1",
  validated_bundle_fingerprint,
  namespace,
  machine_id,
  canonical_decimal(machine_version),
  root_instance_id
])
```

Normative root-identity vector:

```text
JCS:
["determa-root-runtime-identity-2","1","sha256:7e48ad82ea5305c24b7730f4fd24c36ec196a0875c982b85eba5b3a5ddcbb92f","example.turnstile","turnstile","1","turnstile-42"]

hash:
sha256:72dca6d0b2b3690ae28bda2f17a461179b18fbf11daad7a12709d9384a500c64
```

Including the validated bundle fingerprint prevents a changed same-version definition
from reusing a prior root runtime or effect identity. Component and spawned identities
include an owner runtime identity, and every effect includes its emitting runtime
identity, so the distinction propagates through the complete ownership tree and
outbox.

A component runtime identity is:

```text
hash([
  "determa-component-runtime-identity-1",
  "1",
  root_instance_id,
  owner_runtime_id,
  component_definition_pointer,
  canonical_decimal(activation_sequence),
  namespace,
  machine_id,
  canonical_decimal(machine_version)
])
```

For a `machine_id` placement, the final three operands are the referenced machine's
definition identity. For an inline `root`, they are the containing machine's
definition identity and `component_definition_pointer` is the inline placement's full
RFC 6901 pointer. The pointer and `owner_runtime_id` distinguish nested and sibling
inline placements; `activation_sequence` distinguishes re-entry incarnations.

The first inline placement in the same normative bundle above is reached during root
initialization. With activation sequence zero, its normative identity vector is:

```text
JCS:
["determa-component-runtime-identity-1","1","turnstile-42","sha256:72dca6d0b2b3690ae28bda2f17a461179b18fbf11daad7a12709d9384a500c64","/machines/0/root/states/locked/components/0","0","example.turnstile","turnstile","1"]

hash:
sha256:43db74b6a8d6f31543f7d142fb5e25a49e33eb3bf548e7bfd20d59513778cbc3
```

A spawned `instance_id`, which is also its runtime identity, is:

```text
hash([
  "determa-spawned-runtime-identity-1",
  "1",
  root_instance_id,
  owner_runtime_id,
  spawn_action_pointer,
  canonical_decimal(spawn_sequence),
  namespace,
  machine_id,
  canonical_decimal(machine_version)
])
```

Pointers in these tuples are RFC 6901 JSON Pointers into the validated document.

Root creation is logical step sequence `0`. After successful creation, aggregate
`next_logical_step_sequence` is `1`; `next_output_sequence` is the number of external
intents emitted during creation. Every new runtime initializes `next_spawn_sequence`,
every component placement initializes `next_activation_sequence`, and every
runtime/state-path initializes `next_state_activation_sequence` to `0`.

Allocation always takes the current counter value and increments the counter before
using that value in the same atomic step:

- entering a state allocates its state activation sequence;
- creating a component placement allocates its activation sequence;
- executing `spawn` allocates the owner's spawn sequence; and
- appending an external output intent allocates the aggregate output sequence.

Rollback restores every tentative allocation. An accepted handled/faulting envelope
allocates the current `next_logical_step_sequence`; rejection and unhandled delivery
allocate none. All author behavior, initialization, lifecycle work, ownership changes,
and emissions in that RTC use the same step sequence. Fault finalization uses the
reserved value and advances it exactly once.

Every executing behavior has a deterministic cause:

- a delivered envelope's cause id is its `event_id`;
- root initialization derives the cause below with source and target equal to the root
  runtime, parent provenance `creation_id`, the machine root pointer, and ordinal `0`;
- component initialization uses the owner as source, component as target, the current
  cause as parent provenance, the component placement pointer, and its declaration
  index as ordinal; and
- spawned initialization uses the owner as source, child as target, the current cause
  as parent provenance, the spawn action pointer, and the allocated spawn sequence as
  ordinal.

Initialization causes are:

```text
cause_id = hash([
  "determa-cause-identity-1",
  "1",
  cause_kind,
  root_instance_id,
  source_runtime_id,
  target_runtime_id,
  parent_provenance,
  canonical_decimal(step_sequence),
  source_locator,
  canonical_decimal(ordinal)
])
```

`cause_kind` is exactly `root_initialization`, `component_initialization`, or
`spawned_initialization`. `source_locator` is the RFC 6901 pointer specified above.
Thus entry/initial actions can produce deterministic emissions even though their CEL
environment has no `event` binding.

Using the root vector above with `creation_id = "create-7"`, the normative root
initialization vector is:

```text
JCS:
["determa-cause-identity-1","1","root_initialization","turnstile-42","sha256:72dca6d0b2b3690ae28bda2f17a461179b18fbf11daad7a12709d9384a500c64","sha256:72dca6d0b2b3690ae28bda2f17a461179b18fbf11daad7a12709d9384a500c64","create-7","0","/machines/0/root","0"]

hash:
sha256:c9e8e89a01362f40e9a74c01392d09abe2323f31c8f14f22e05bfcaf6dfac0ab
```

An internal emission derives:

```text
event_id = hash([
  "determa-event-identity-1",
  "1",
  root_instance_id,
  source_runtime_id,
  target_runtime_id,
  cause_id,
  canonical_decimal(step_sequence),
  emission_locator,
  canonical_decimal(emission_ordinal)
])
```

Its immutable envelope `event_id` is exactly that derived value; there is no second
internal cause identifier. `emission_ordinal` is zero-based within the executing action
and follows declared target order. An author send's `emission_locator` is its RFC 6901
action pointer. Engine lifecycle emissions use an `emission_locator`
exactly equal to one of `system:component_completion`,
`system:spawned_completion`, `system:component_failure`, or
`system:spawned_failure`; the ordinal is zero unless that lifecycle operation emits
multiple envelopes, in which case it is their specified order. Distinct fan-out targets
therefore have distinct event ids.

An external output intent derives:

```text
effect_id = hash([
  "determa-effect-identity-1",
  "1",
  [namespace, machine_id, canonical_decimal(machine_version)],
  root_instance_id,
  emitting_runtime_id,
  cause_id,
  canonical_decimal(step_sequence),
  action_document_pointer,
  canonical_decimal(emission_index)
])
```

For effects emitted by an inline component, the definition triple is the containing
machine identity above. `emitting_runtime_id` already contains the full placement
pointer and activation sequence, so identical actions in sibling, nested, or later
inline placements cannot collide.

`emission_index` is the zero-based external-intent ordinal within that executing
action. The intent also carries the allocated aggregate-monotonic output `sequence`,
event name, typed payload, and correlation id. Retrying the same uncommitted prior
state and envelope reproduces the same state, emissions, ids, and order. Processing
several envelopes by repeated `dispatch` calls produces the same result as a host
convenience API that applies that same ordered envelope sequence atomically.

The queue or external adapter may use these identities for deduplication, but the core
does not require a delivery guarantee.

## 10. Faults and envelope disposition

### 10.1 Engine faults

Engine faults include:

- `guard_fault`;
- `action_fault`;
- `invalid_instance_target`;
- `inactive_component_target`;
- `binding_not_empty`;
- `contained_runtime_fault`;
- `invariant_fault`; and
- `cascade_fault`.

There is no runtime `type_fault`; statically checkable types are load-time validation.
`invalid_instance_target` and `inactive_component_target` may also be pre-step
rejection codes. They are engine faults only when an already accepted RTC action
attempts an invalid send target.

A `cancel` action whose expression evaluates to null, or does not currently address a
running or retained-faulted directly or transitively owned instance, is the successful
no-op defined by §7.2 and is never `invalid_instance_target`.

On an engine fault, the core:

1. rolls the RTC step back to its exact pre-step aggregate state, including variables,
   configuration, history, ownership changes, and tentative emissions;
2. commits one deterministic fault finalization using the reserved step sequence;
3. records the fault and marks the executing runtime faulted; and
4. returns no emission from the rolled-back author actions.

When the executing runtime is the aggregate root, that fault finalization is terminal
for the entire ownership aggregate. The engine preserves the rolled-back diagnostic
tree exactly: retained component and spawned descendants keep their pre-step
configuration, variables, individual status, and fault records, but are frozen and
non-targetable. It runs no descendant exit/cancellation behavior and emits no contained
failure notification for this freeze. The aggregate status is `faulted`; all later
processing and read-only behavior follows the aggregate-terminal rule in §6.1.

The committed fault record is:

```text
{
  runtime_id,
  cause_id,
  code,
  step_sequence,
  source_locator
}
```

`source_locator` uses a closed vocabulary. A value beginning `/` is the exact RFC 6901
pointer into the validated bundle; a value beginning `system:` is one of the fixed
locators below. Engines MUST use this mapping:

| fault code | exact `source_locator` |
|---|---|
| `guard_fault` | pointer to the failing event-transition or choice `guard` value |
| `action_fault` | pointer to the failing CEL expression value inside the action, entry, exit, initial transition, choice branch, component binding, or spawn binding; for an absent `refresh.only` field, the pointer to the first absent list item |
| `invalid_instance_target` | pointer to the executing send action's `to`/`targets` member or, for a dynamic instance expression, that exact expression value |
| `inactive_component_target` | pointer to the executing send action's `to`/`targets` member that names the component |
| `binding_not_empty` | pointer to the executing spawn action's `bind_to` value |
| `contained_runtime_fault` | exactly `system:unhandled_contained_failure` |
| `cascade_fault` | exactly `system:cascade_cleanup` |
| `invariant_fault` | exactly `system:invariant` |

For a `targets` list, the pointer includes the failing zero-based list index. A
contained failure notification embeds the §4.4 public projection of the child fault
record; if its delivery later faults the owner, the owner receives a separate retained
record with the system locator above. Rejected pre-step envelopes and rejected creation
have no committed fault record and therefore no `source_locator`.

The core does not remove, acknowledge, retry, retain, or dead-letter the input envelope.
The caller still owns the exact envelope and receives `disposition: faulted`. Its queue
plugin decides what to do with it.

### 10.2 Contained runtime faults

A component or spawned-runtime fault freezes that runtime and retained descendants,
then returns one deterministic internal failure emission to the immediate owner:

- `determa.component_failed` with:

  ```text
  {
    component_id: identifier,
    component_runtime_id: non_empty_string,
    fault: public_fault_record
  }
  ```

- `determa.spawned_instance_failed` with:

  ```text
  {
    instance: instance_reference,
    instance_id: non_empty_string,
    machine_id: identifier,
    machine_version: positive_integer,
    fault: public_fault_record
  }
  ```

The `fault` field is the fixed `public_fault_record` projection from §4.4. It copies
`runtime_id`, `cause_id`, `code`, and `source_locator` unchanged and encodes the
retained mathematical `step_sequence` as its `canonical_decimal` string. Each failed
runtime emits its notification once. Source is the failed runtime; target is its
immediate owner; the emission retains the faulting cause and uses the corresponding
system locator from §9.

Initialization faults are isolated contained-runtime faults with mandatory behavior:

- A component whose initialization faults rolls back only its tentative author
  initialization state and emissions, commits the diagnostic projection below as a
  retained-faulted placement, and contributes exactly one
  `determa.component_failed` emission. Later component placements continue
  initialization in declaration order.
- A spawned child whose initialization faults rolls back only its tentative author
  initialization state and emissions, commits the diagnostic projection below as a
  retained-faulted child, leaves `bind_to` set to that nominal child reference, and
  contributes exactly one `determa.spawned_instance_failed` emission. Later actions in
  the owner's ordered action list continue.

The retained diagnostic projection is exact: runtime and definition identity,
component-placement or spawned ownership/lifetime-holder identity, allocated component
activation or owner spawn sequence, `faulted` status, and the committed fault record.
Its configuration, variables, history, components, owned spawned descendants, and
author emissions are empty. Supplied root input/external values and root `init` values
are not retained. Its child-local next spawn sequence is zero and its state/component
activation-counter maps are empty; no author-initialization allocation survives.
The containing owner's already allocated placement activation or spawn counter remains
consumed, and a spawned parent's `bind_to` value remains set as specified above.

The failure emission occupies the point at which that contained initialization faults:
component failures follow placement declaration order; spawned failures follow their
spawn action's position relative to other owner emissions. Earlier tentative owner
emissions targeting a now-faulted pending component remain in the result as specified
by §7.1. No author emission from the failed initialization survives.

All of these retained faults, bindings, counters, later component/action work, and
failure emissions remain tentative until the enclosing owner RTC commits. If any later
work faults that owner step, the owner rollback removes the newly created contained
runtimes, bindings, contained fault records, and failure emissions and restores every
counter. If the owner step commits, the owner remains running and the retained-faulted
child is inspectable and cleanup-cancellable but cannot process ordinary delivery.

A reserved failure event that reaches its owner unhandled faults the owner with
`contained_runtime_fault`; its source locator is exactly
`system:unhandled_contained_failure` and its cause id is the failure envelope's
`event_id`. A queue plugin can discard or delay the notification; the core does not
claim otherwise.

The immediate owner may cancel a retained-faulted spawned child for cleanup. Ordinary
input cannot advance a faulted runtime.

### 10.3 Domain failures

Application outcomes such as `payment_rejected`, `email_failed`, or
`schedule_rejected` are ordinary declared events. They do not become engine faults
unless their own handling violates the engine contract.

### 10.4 Dead letters are not core state

The specification defines no `dead_letter`, `dead_letters`, or `dead_letter_policy`
field and no dead-letter storage shape.

A queue plugin may:

- discard unhandled or faulting envelopes without retaining anything;
- retain complete envelopes and fault metadata;
- retain metadata without payloads;
- retry before retention;
- forward to a broker-native dead-letter facility; or
- expose any other explicitly configured policy.

Its property names, configuration schema, capacity, retention, privacy, and operational
guarantees belong entirely to that plugin. Machine definitions cannot inspect or depend
on them.

## 11. Plugins and hosting

### 11.1 Queue plugins

A queue plugin is given ordered emissions after any committed core result and later
presents envelopes to `dispatch`. The core does not standardize a concrete plugin API,
but the host must preserve each presented envelope's immutable value and identity for
the duration of its RTC step.

Plugins may differ in:

- ordering and fan-out;
- in-memory or durable storage;
- delivery attempts and acknowledgements;
- duplicate delivery and deduplication;
- retry, delay, deferral, and dead-letter behavior;
- transactional integration with aggregate persistence; and
- capacity, overflow, backpressure, and availability.

Core determinism means that the same valid prior state and same envelope produce the
same result. It does not mean that different queue plugins produce the same delivery
trace.

A bundle whose correctness depends on deferral is not self-contained or portable in
format 1. Its behavior depends on host/plugin configuration outside the document, so
core conformance cannot guarantee equivalent behavior across hosts.

### 11.2 Timer extensions

Time is modeled through external event-producing extensions, never through core clock
state. A machine may emit a declared scheduling request and later receive a declared,
correlated elapsed, rejected, failed, or cancelled event.

The timer extension is a black box. It need not know which state or business process
uses the event. Its request payload, cancellation behavior, delivery reliability,
duplicate policy, clock source, persistence, and credentials are extension concerns.

Different timer extensions may provide best-effort in-memory behavior, durable
at-least-once delivery, database integration, or real-time-oriented scheduling. Core
Determa provides no guarantee that a scheduled event arrives, arrives once, arrives in
order, or arrives near its requested time.

Late or duplicate elapsed events are ordinary input. Machines and queue plugins handle
them through correlation, explicit state behavior, or plugin policy.

### 11.3 External effects

All remote I/O follows the same boundary:

1. a successful RTC returns a deterministic external intent;
2. the host persists/delivers it according to its plugin guarantees;
3. the external system eventually may produce a declared correlated input; and
4. that envelope is processed in a later independent RTC step.

No external success is inferred merely because an intent was emitted.

### 11.4 Hosting profiles

Informative examples of valid hosts include:

- foreground request/response processing with an in-memory queue;
- one-row database persistence with transactional inbox/outbox plugins;
- a durable background worker;
- a broker-backed distributed host;
- an embedded main-thread loop; and
- an MCP adapter exposing declared public inputs as tools.

Plugin names and configuration never appear in portable bundle grammar.

## 12. Inspection and visualization

Implementations SHOULD expose read-only inspection of:

- aggregate/root and runtime identities;
- active configurations and variables;
- component and ownership relationships;
- status and fault records;
- history; and
- deterministic emissions returned by the last call.

Queue contents, delivery attempts, dead letters, scheduled jobs, and broker
acknowledgements are inspected through their owning plugins, not the core engine.

Enabled-event inspection is deliberately undefined in format 1. A configuration alone
can reveal only structurally present handlers: whether a guarded handler is enabled is
a property of the configuration together with a candidate envelope and current
variables. Implementations MUST NOT present an implementation-specific enabled-event
shape as a portable format-1 contract.

Mermaid `stateDiagram-v2` is a useful default exporter:

| Determa | Mermaid |
|---|---|
| root composite | diagram root |
| composite state | `state S { ... }` |
| initial target | `[*] --> target` |
| final state | `state --> [*]` |
| transition | `source --> target: event [guard] / action` |
| internal reaction | annotation/note |
| parallel components | annotated placements or separate diagrams |

Mermaid renders entry/exit/action text but does not enforce Determa execution order.
Exported diagrams MUST therefore be treated as views of the normative bundle, not an
alternative executable definition.

## 13. Deliberately unsupported in format 1

The current alpha classifies the previously explored capability areas as follows. This
is a completeness boundary, not a compatibility promise for earlier drafts.

| capability | current format 1 rule |
|---|---|
| hierarchy and final states | retained with `root`, composite states, and final states |
| transitions and choices | retained; transition actions run before source exit |
| entry and exit | retained; the triggering `event` is not visible |
| shallow/deep history | retained through explicit history targets; plain composite targets restart |
| parallel behavior | changed to isolated lifecycle-bound components; no regions or implicit broadcast |
| variables and external refresh | retained as typed root inputs/external values plus `env`/`refresh` |
| actions and publication | retained as structured actions; publication is explicit `send` |
| shared contracts | represented by bundle public event declarations; separate named contracts are unsupported |
| timers | external scheduling/event extensions only |
| deferral and dead letters | queue-plugin policy only; no portable machine fields or core storage |
| owned spawning | retained for same-bundle machines with nominal `instance_reference` values |
| submachines and package imports | unsupported |
| definition migration/hot-swap | explicit portable aggregate migration under §16; never implicit in ordinary dispatch |
| observers and export | read-only inspection and visualization are recommended, not executable core behavior |
| snapshots | closed portable aggregate-state envelope and package under §16 |
| stores and CLI protocols | host/implementation concerns, not bundle grammar |

The pre-release format deliberately omits:

- native timers, clocks, `after`, sleeps, and time-triggered transitions;
- native queues, retries, acknowledgement, deferral, or dead letters;
- orthogonal regions with implicit event broadcast;
- shared mutable variables or shared queue state across runtimes;
- direct host-to-component delivery;
- cross-runtime transitions;
- remote or detached core `spawn`;
- package imports and dependency/version resolution;
- live definition replacement without the explicit §16 migration operation;
- identity rekeying during migration;
- destructive reset as a migration fallback;
- arbitrary executable migration code or author behavior during migration;
- standardized CLI/store JSON shapes;
- standardized enabled-event inspection;
- root engine-fault recovery/reset;
- reset or re-entry of the machine root by an ordinary transition;
- distributed transactions, exactly-once delivery, or hard real-time guarantees;
- local transitions whose target is not a strict descendant of their composite source;
- external re-entry of a proper ancestor target; and
- plugin discovery, installation, manifests, or standardized configuration fields.

These omissions are not reserved implementation hooks. A host may provide them only
outside core through events and plugins unless a later format revision defines otherwise.

## 14. Conformance plan

Format-1 conformance MUST be added before engine implementation or release. Cases should
be one behavior per fixture and include:

- document/schema positives and negatives;
- variable initialization/default requirements and creation/component/spawn binding
  rejection;
- payload-default materialization and optional-field absence;
- integer bounds and integer-to-binary64 normalization at every typed boundary;
- strict parsed-value and Unicode-scalar validation plus portable numeric, Boolean,
  and null source-token resolution;
- CEL profile name/type checking, event visibility, numeric faults, selected conditional
  branches, commutative error absorption for `&&`/`||`, and Unicode string ordering;
- one-snapshot send-expression evaluation and deterministic payload/correlation/target
  fault precedence;
- leaf-to-ancestor dispatch and false-guard fallback;
- internal, self, local, and external descendant-reset transition traces;
- proper-ancestor transition bounds and the absence of external ancestor re-entry;
- schema rejection of non-canonical internal/local transition shapes;
- least-common-ancestor exit/entry paths;
- transition-action-before-exit ordering;
- load-time rejection of transition writes to destinations that the transition exits;
- root-boundary preservation plus root self/history rejection;
- root-local rejection and terminal aggregate-root fault behavior across descendants;
- destroyed `refresh` rejection, missing-`refresh.only` rollback, and owned-child
  cancellation on reference scope exit;
- initial descent and ordered choice;
- compound event/initial/choice-chain actions resolved before one lifecycle transition;
- explicit history resume/restart, self-history lifecycle replay, first-entry fallback,
  capture timing, shallow/deep restoration, local history targeting, and variable
  reinitialization;
- immutable envelope validation and each disposition;
- explicit owner-to-component `env` forwarding, typed component refresh, and rejection
  of every broader reserved-event send form;
- no recursive delivery of internal sends;
- exact immutable root/spawned/component targets and stale component-incarnation
  rejection;
- component creation/routing/completion/disposal, including pending-initialization
  addressing and inline identity vectors;
- spawn, nominal reference, completion, cancel and cascade, including no-op
  cancellation of null and disposed references;
- completion and `stop` output ordering across author behavior, descendant cleanup,
  active exits, and reserved owner notifications;
- isolated component/spawn initialization faults and enclosing-owner rollback;
- root execution of `{owner: true}`, exit-time spawn rejection, and deterministic
  exit-time sends to disposed targets;
- omitted `refresh.only` and absence of retained external-source maps;
- coherent validated-bundle-fingerprint, root-runtime, initialization-cause,
  event/effect, and inline-component identity vectors across languages;
- prior-state shape and supplied-bundle compatibility rejection, including changed
  metadata and same-version definitions;
- full RTC rollback and contained-runtime failure propagation;
- exact document/system fault locators;
- completed runtime empty configuration/variables and retained terminal diagnostics;
- queue-independent behavior for a fixed delivery trace; and
- explicit absence of timer, queue, and dead-letter fields from core state.

Persistence and migration conformance additionally requires:

- canonical aggregate encoding, decoding, digest verification, and byte-stable
  round trips, including exact no-trailing-newline RFC 8785 byte vectors;
- strict rejection of unknown artifact fields, formats, schema versions, invalid
  typed values, invalid relations, and inconsistent source definitions;
- exact root, four-field spawned-reference, and five-field component target shapes,
  with rejection of every extra/missing format-1 target member;
- complete faulted-aggregate round trips including fault `step_sequence` and historical
  definition anchor;
- content-addressed definition and descriptor resolution, including missing,
  untrusted, and hash-mismatched artifacts;
- package attachment equivalence with the definition-registry contract;
- unchanged-definition resume through encode/decode;
- aggregate-shape-compatible migration with no logical-state transform;
- exact per-descriptor `migration_applied` audit records and empty equal-fingerprint
  route no-op results;
- total active-state, variable, history, component, owned-runtime, lifetime-holder,
  counter, and fault-anchor transforms;
- repeated-runtime/activation transform scoping with no cross-runtime value mixing;
- explicit deleted-state quarantine with no name, ancestor, initial, history, or reset
  guess;
- exact pinned multi-hop routes, adjacency checks, and cycle/alternate-route rejection;
- immutable runtime, target, nominal-reference, activation, spawn, logical-step, and
  output identities across migration;
- retry-identical success or failure, complete rollback, and no counter consumption;
- migration followed by handled, unhandled, rejected, and faulted dispatch in one
  host transaction;
- terminal completed/faulted maintenance migration without reactivation or emission;
- exact terminal maintenance/policy failures and definition/descriptor authorization
  failures; and
- descriptor-declared and cumulative resource-limit accounting.

Transaction crash points, inbox idempotency, outbox uniqueness, quarantine storage,
and local-cache behavior belong to a separately declared persistence hosting profile.

Golden-trace cases SHOULD make every action emit a trace token so ordering is directly
reviewable.

## 15. Future example repositories

After conformance and engine support, a separate examples repository should validate:

- local foreground processing;
- database/ACID embedding with queue plugins;
- hibernation and later host ingress;
- isolated parallel components;
- owned spawning and cancellation;
- remote orchestration through effects;
- best-effort and durable timer extensions;
- broker retry/dead-letter policies;
- package reuse after import semantics exist;
- MCP exposure; and
- real-time-oriented hosting;
- portable aggregate-state round trips; and
- one-row and normalized database persistence with lazy migration, transactional
  inbox/outbox/audit, rollback injection, and quarantine recovery.

Those examples are empirical design validation. They are not part of this
specification-only change.

## 16. Portable persistence and definition migration

### 16.1 Independent artifact identities

Machine documents remain numeric `format: 1`. Persistence introduces three independent
closed JSON artifacts:

| artifact | exact format field | exact schema-version field |
|---|---|---|
| aggregate-state envelope | `aggregate_state_format: "determa.aggregate_state"` | `aggregate_state_schema_version: 1` |
| migration descriptor | `migration_descriptor_format: "determa.aggregate_migration"` | `migration_descriptor_schema_version: 1` |
| transport package | `aggregate_state_package_format: "determa.aggregate_state_package"` | `aggregate_state_package_schema_version: 1` |

These wire schema versions, machine format, repository/package SemVer, launcher
SemVer, and author-controlled machine `version` are independent version domains.
Unknown artifact formats or schema versions are rejected before semantic validation;
there is no nearest-version parsing, implicit conversion, or best-effort field
retention.

The artifacts MUST be JSON encoded as strict UTF-8. Their parsers apply the §2
source-level duplicate-name, acyclic JSON-value, Unicode-scalar, Boolean, null, and
finite-number requirements before the applicable schema. YAML is not a portable
encoding for these artifacts. Every schema is closed: an unknown member is invalid.
The exact structural schemas are:

- `schema/aggregate-state.schema.json`;
- `schema/migration-descriptor.schema.json`; and
- `schema/aggregate-state-package.schema.json`.

Structural validity is necessary but not sufficient. The semantic invariants in this
section are mandatory even where JSON Schema cannot express ordering, cross-reference,
digest, type, or totality constraints.

The portable core operations are behaviorally equivalent to:

```text
encode_aggregate(source_bundle, abstract_aggregate) -> canonical_json_bytes
decode_aggregate(canonical_or_whitespace_json_bytes, definition_resolver)
  -> abstract_aggregate
migrate_aggregate(
  aggregate_envelope,
  target_validated_bundle_fingerprint,
  ordered_descriptor_digests,
  artifact_resolver,
  resource_limits,
  maintenance_mode
) -> { aggregate_envelope, audit_records } | migration_failure
```

Language APIs may use idiomatic names. These are pure operations; they do not define a
database API, registry transport, queue, transaction manager, or bulk migration job.

### 16.2 Canonical values and aggregate encoding

Artifact-owned counters and machine versions use canonical decimal strings: `0`, or a
non-zero digit followed by zero or more digits. Signed typed integer values use `0` or
an optional `-` followed by a non-zero digit and zero or more digits. Bounds that are
semantic rather than structural are checked after schema validation.

The one exception is `target_identity`: it embeds the exact normalized format-1 §6.1
target object so an already-created target is never rewritten by persistence or
migration. Its `machine_version` and component `activation_sequence` therefore retain
the format-1 positive/non-negative JSON integer representation. Origin, current
relation, and counter records use the artifact-owned decimal strings and may carry the
additional migration data that is deliberately absent from the target.

Every stored Determa value uses exactly one typed projection:

```text
["null"]
["boolean", boolean]
["string", unicode_scalar_string]
["integer", signed_64_bit_canonical_decimal]
["float", sixteen_lowercase_binary64_hex_bits]
["list", [typed_value, ...]]
["map", [[string_key, typed_value], ...]]
```

Map entries are strictly increasing by key UTF-8 bytes and contain no duplicate key.
Negative binary64 zero is encoded as positive zero. Non-finite binary64 values are
invalid. Lists preserve order. This is the same value distinction used by the §8
validated-bundle fingerprint; a declared `instance_reference` value is encoded through
its exact §4.5 map projection and is retyped from its declaration on decode.

All arrays representing sets or maps have one canonical order:

- `runtimes`: `runtime_id` UTF-8 bytes;
- active leaf pointers, history pointers, and definition-pointer counter domains:
  pointer UTF-8 bytes;
- state activations: pointer, then numeric activation sequence;
- variables: declaration pointer, then numeric declaring-state activation sequence.

Any duplicate canonical key or noncanonical order is `invalid_aggregate_state`; a
decoder never silently sorts an accepted envelope. Serialization emits RFC 8785 JCS
bytes of the complete envelope with no byte-order mark, leading/trailing whitespace,
or trailing newline. A parser may accept insignificant JSON whitespace and then verify
that the semantic data is canonical.

The human-readable `aggregate-state.json`, `migrated-aggregate-state.json`, and
`faulted-aggregate-state.json` fixtures are pretty representations, not canonical
serialized bytes. `aggregate-state.canonical.json` and
`migrated-aggregate-state.canonical.json` are the exact byte goldens; their complete
file bytes MUST equal RFC 8785 serialization of the corresponding pretty fixture.

The digest is:

```text
aggregate_state_digest = hash([
  "determa-aggregate-state-digest-1",
  envelope_without_aggregate_state_digest
])
```

`hash` is the §9 SHA-256/JCS construction. A mismatch is
`aggregate_state_digest_mismatch`. The digest does not include database metadata,
quarantine state, queue state, or package attachments.

### 16.3 Complete root ownership aggregate

One envelope represents exactly one §3 root ownership aggregate. It contains:

- the current validated-bundle fingerprint, namespace, root machine identity, machine
  format, root/creation/runtime identities, and migration sequence;
- aggregate next logical-step and output sequences;
- every retained root, component, and owned spawned runtime;
- each runtime's immutable identity origin and immutable target identity;
- each runtime's current definition binding and current relationship;
- lifecycle status, active leaves and state activations, live variables, history,
  spawn/state/component counters, lifetime-holder association, and retained fault;
  and
- no plugin-owned queue, deferred event, timer, broker receipt, acknowledgement,
  dead-letter, credential, or transport configuration.

The root runtime occurs exactly once and matches every top-level root identity field.
Every other runtime has exactly one retained owner. The ownership graph is acyclic and
reachable from the root. Runtime identifiers are unique. Component placement and
owned-child relation data agree with the immutable target identity and with the
current target definition. Every active pointer, variable declaration, history slot,
component placement, spawn action, and counter domain resolves against the runtime's
current definition.

The envelope uses declaration pointers plus declaring-state activation sequences for
live variables, so shadowed names remain lossless. History records contain null or the
exact recorded target-pointer set. A wire fault record is exactly the five committed
format-1 fields `runtime_id`, `cause_id`, `code`, `step_sequence`, and
`source_locator`, plus required `definition_fingerprint`. `step_sequence` uses the
artifact canonical-decimal projection. The definition fingerprint anchors the
historical locator; migration never reinterprets that locator against a later
definition. `examples/persistence/faulted-aggregate-state.json` is the normative
faulted round-trip vector.

Encoding first validates the implementation's abstract aggregate under the supplied
source bundle. Decoding verifies structure, canonical form, digest, definition
availability and fingerprint, all relationships, all typed values, and the complete
§8 abstract-state invariants before returning state. An implementation-specific
dictionary, object graph, compiled machine, callback, or pointer is never portable
state.

### 16.4 Immutable identity and mutable definition binding

Migration separates three concepts for every existing runtime:

1. `identity_origin`: the definition, owner, placement/action pointer, and allocation
   sequence from which the runtime identity was originally derived;
2. `target_identity`: the immutable value accepted by already-created envelopes and
   nominal references; and
3. `current_definition` and `relation`: the definition and current placement/action
   against which future behavior resolves.

`target_identity` is exactly one normalized §6.1 target:

```text
{ root: { root_instance_id, root_runtime_id } }
{ spawned_instance: {
    root_instance_id, instance_id, machine_id, machine_version
} }
{ component: {
    root_instance_id, owner_runtime_id, component_id,
    component_runtime_id, activation_sequence
} }
```

It has no `kind`, namespace, owner/spawn metadata inside a spawned reference, or
definition/placement pointer inside a component target. Those facts belong to
`identity_origin` or `relation`. The four-field spawned reference is exactly §4.5, and
the component target is exactly §6.1.

`runtime_id`, `identity_origin`, and the complete normalized `target_identity` bytes
are invariant across every version-1 descriptor. Root identity, component runtime
identity, component activation sequence, spawned instance reference, spawned instance
id, spawn sequence, and existing lifetime-holder activation identity are never
rederived.

For a migrated component, author syntax using the target definition's current
`component_id` resolves through the current relation to its preserved target identity.
For a migrated spawned runtime, its existing `instance_reference` remains byte-for-byte
stable while `current_definition` changes. New components and spawned instances use
the target definition normally. Identity rekeying, external-reference rewriting, and
detached child migration are unsupported in schema version 1.

### 16.5 Content-addressed definition registry

An aggregate references its current definition by the exact §8
`validated_bundle_fingerprint`. A conforming resolver stores the canonical typed
normalized bundle tree once under that key:

- put-if-absent is idempotent;
- the same key with different canonical bytes is an integrity failure;
- bytes are rehashed and semantically revalidated before admission to a trusted local
  cache;
- source and target definitions remain available while any aggregate, descriptor, or
  audit record references them; and
- garbage collection is reference-aware, never age-only.

Content addressing proves integrity, not authority. A deployment separately
allowlists or verifies a signed release manifest containing trusted definition,
descriptor, and route digests. Signature algorithms, key management, and registry
transport belong to the host.

An ordinary aggregate row does not embed its definition. This avoids copying old
definitions into every dormant row while still permitting lazy migration. Retaining
old normalized declarative definitions centrally does not retain old host executable
logic.

### 16.6 Aggregate-shape fingerprint

The aggregate-shape fingerprint proves only that existing logical state can be bound
to another definition without transformation. It does not claim behavioral
equivalence.

Starting from the §8 normalized bundle, implementations construct this exact plain
JSON projection before applying the §8 typed-value projection:

```text
state_bearing_tree = {
  format: 1,
  namespace,
  machines: [machine_projection, ...]
}

machine_projection = {
  machine_id,
  version,
  root: state_projection
}

state_projection = {
  definition_pointer,
  type,
  history?,
  variables?: [variable_projection, ...],
  states?: [state_projection, ...],
  components?: [component_projection, ...],
  spawn_sites?: [spawn_site_projection, ...]
}

variable_projection = {
  declaration_pointer,
  type,
  nullable,
  input,
  external,
  machine_id?
}

component_projection = {
  declaration_pointer,
  declaration_index,
  component_id,
  machine_id?
  inline_root?
}

spawn_site_projection = {
  action_pointer,
  machine_id,
  holder_variable_declaration_pointer
}
```

Machines retain bundle array order. Child states are sorted by state identifier UTF-8
bytes. Variables are sorted by declaration pointer UTF-8 bytes. Components retain
declaration order. Spawn sites are sorted by action-pointer UTF-8 bytes after
recursively visiting entry, exit, handler, choice, and nested action lists.

Every state has `definition_pointer` and normalized `type`. `history` is included only
for a composite state and contains its normalized mode. Empty `variables`, `states`,
`components`, and `spawn_sites` arrays are omitted. Variable `nullable` is `true` only
for an `instance_reference` declaration and otherwise `false`; normalized `input` and
`external` are always included. Variable `machine_id` is included only when declared.
A component includes exactly one of `machine_id` or recursive `inline_root`.
`declaration_index` is its zero-based array index. A spawn site's holder pointer is the
resolved `bind_to` declaration pointer or null.

This recursive tree therefore contains every state, placement, spawn, holder, and
declaration-pointer domain from which a retained path, relationship, or counter key can
be drawn. Object keys are encoded with the §8 typed-tree map ordering. Metadata, event
declarations and payload defaults, guards, ordinary action expressions, transition
targets, entry/exit behavior other than spawn-site shape, and component `with`
expressions are excluded because they cannot make an existing logical-state field
structurally invalid.

```text
aggregate_shape_fingerprint = hash([
  "determa-aggregate-shape-fingerprint-1",
  typed_state_bearing_tree
])
```

A `compatible` descriptor is valid only when independently recomputed source and
target shape fingerprints are equal and every mapping array is empty. It changes only
the aggregate and runtime current definition references and increments
`migration_sequence`; every other field is byte-for-byte preserved before digest
recomputation. A machine-version change, path change, rename, or any other
state-bearing projection difference requires `transform` mode.

### 16.7 Immutable declarative migration descriptors

A migration descriptor names exactly one source and one target machine format,
validated-bundle fingerprint, and independently recomputed aggregate-shape
fingerprint. Schema version 1 requires both machine formats to be numeric `1`. Its
digest is:

```text
migration_descriptor_digest = hash([
  "determa-migration-descriptor-1",
  descriptor_without_migration_descriptor_digest
])
```

Changing any descriptor member creates a different descriptor. A digest match does
not make it trusted. The deployment must authorize the exact digest.

`transform` descriptors contain closed rules for machine/root bindings, active-state
materialization, variables, history, components, owned runtimes, lifetime holders, and
counter domains. Descriptors are immutable pure data. They cannot execute Python,
Rust, JavaScript, WASM, shell code, author actions, entry/exit behavior, transitions,
choice selection, component/spawn initialization, host callbacks, plugins, network or
filesystem I/O, clocks, randomness, environment reads, credentials, or secrets.
Migration itself emits no author, lifecycle, internal, or external event.

Variable `transform` and `initialize` rules use a closed migration CEL profile. It is
the §5.2 portable profile restricted to null, Boolean, signed-64-bit integer,
binary64, string, list, and string-keyed map values and their already enumerated pure
operators/functions. It excludes `event`, `owner`, runtime inspection,
`instance_reference`, `has(event...)`, comprehension, iteration, and every extension.
For a transform, the descriptor's `source_declaration_pointers` order binds exact
symbols `source_0`, `source_1`, and so on. An initialize expression has no symbols.
Each expression is parsed and statically checked against source declaration types and
the single target declaration type before migration. Identity and nominal-reference
mappings are descriptor operations, never CEL values.

### 16.8 Exact route and migration algorithm

A route is the exact ordered array of trusted descriptor digests supplied by deployment
configuration or a trusted release manifest. The engine never searches a registry
graph or selects a shortest, newest, cheapest, or otherwise preferred path.

Before transformation:

- the first descriptor source equals the stored aggregate fingerprint;
- each descriptor target equals the next descriptor source;
- the final target equals the requested target fingerprint;
- no descriptor digest repeats and no source/target cycle occurs;
- every definition and descriptor is present, hash-valid, semantically valid, trusted,
  and within declared resource requirements; and
- the route's first source still equals the locked aggregate when execution begins.

An empty route with equal stored/requested fingerprints succeeds as a strict no-op: it
returns the exact input aggregate envelope bytes and `audit_records: []`, performs no
artifact lookup beyond the already required source-definition integrity and
authorization checks, does not recompute the digest, and does not require terminal
maintenance mode. An empty route with unequal fingerprints fails with
`migration_route_missing`. Multiple available routes are irrelevant; only the pinned
ordered array is evaluated. All intermediate states remain in memory and only the
final aggregate is committed.

For each descriptor, migration:

1. validates the complete source candidate against the exact source definition;
2. reserves no logical-step, output, spawn, state, or component sequence;
3. applies each mapping to an isolated candidate;
4. increments `migration_sequence` exactly once;
5. validates every field and relationship against the target definition;
6. computes the target canonical envelope and digest; and
7. either makes that candidate the next source or discards it completely.

The same canonical source bytes, exact route, trusted artifacts, and limits reproduce
the same success bytes and audit records or the same deterministic failure.

### 16.9 Total transform matrix

The descriptor validator and migration operation jointly enforce:

| aggregate field | required result |
|---|---|
| wire format/schema | exact supported version |
| current definition | exact source match and target assignment |
| root/creation identity | preserve |
| runtime id and identity origin | preserve |
| immutable root/component/spawn target | preserve |
| runtime current machine/root binding | preserve in compatible mode or map exactly once |
| lifecycle status | preserve |
| active leaves | every source leaf maps exactly once to a complete valid target leaf set |
| active ancestors/activations | map by state pointer with coherent ancestry and preserved activation values |
| live variables | each source is copied, transformed, or explicitly dropped; every required target live declaration receives exactly one typed value |
| shadowed variables | match by declaration pointer and activation, never bare name |
| history | each source slot maps or explicitly drops; each new target slot is explicitly null or mapped |
| component placement | every retained placement maps to one compatible target placement |
| owned spawned runtime | every retained child maps to one target current definition/action binding |
| lifetime holder | maps to one surviving compatible declaration with the same active holder activation |
| nominal references | preserve exact immutable value and validate holder association |
| fault diagnostics | preserve record and source-definition anchor |
| aggregate logical/output counters | preserve and never decrease/reset |
| runtime spawn counter | preserve |
| state/component counter domains | one-to-one map, explicit zero initialization, or explicit maximum merge |
| active/relationship allocations | preserve; mapped next counter remains strictly greater |
| migration sequence | increment once per successful descriptor |

After every descriptor, no source field or required target field may remain
unaccounted for. Duplicate source consumption, duplicate target production, ambiguous
mapping, invalid target type, incompatible relation, stale current pointer, and
counter inconsistency are `migration_totality_failure`.

Mapping rules are definition rules, not single aggregate occurrences. A rule applies
independently to every retained runtime whose current source binding resolves the
rule's source machine/root. State, history, component, owned-runtime, holder, and
counter mappings operate on each occurrence in that runtime while preserving that
occurrence's runtime and activation identity.

Variable occurrence identity is exactly:

```text
(runtime_id, variable_declaration_pointer, declaring_state_activation_sequence)
```

For each target runtime whose mapped active configuration makes a target declaration
live, one `copy`, `transform`, or `initialize` rule must produce that target occurrence.
A `copy` consumes the unique live source occurrence with the mapped declaration pointer
in the same source runtime. A `transform` resolves every
`source_declaration_pointers[i]` to the unique live occurrence in that same runtime and
binds only its value as `source_i`. An `initialize` has no source occurrence. A `drop`
consumes each matching live source occurrence independently.

All inputs come from one immutable pre-descriptor snapshot. Rules cannot observe
another rule's output. Rules that consume one source occurrence twice, produce one
target occurrence twice, reference declarations outside the applicable source/target
runtime bindings, or could combine values from different runtimes are
`invalid_migration_descriptor`. If a statically valid required source occurrence is
not live for an applicable target occurrence, is multiply live, or a required target
occurrence remains unproduced, the result is `migration_totality_failure`. Runtime
iteration uses canonical `runtime_id` order only for resource accounting; because
occurrences are isolated and rule domains cannot overlap, result bytes never depend on
host map or iteration order.

When an active state is deleted or incompatible, version 1 permits only:

- an exact source-leaf to target-leaf mapping;
- an explicit ancestor mapping accompanied by the complete resulting target leaf set;
- a declared target initial/history selection only when it resolves without any guard
  or author action and the descriptor still provides the final leaf set and every new
  live value; or
- deterministic failure and quarantine.

The engine never guesses by equal name, nearest surviving ancestor, initial state,
history, or root reset. Removed variables require an explicit destructive `drop` with
an operator-facing reason. Type changes require a statically checked transform.
Removed live components, incompatible owned runtimes, missing holders, or ambiguous
identity mappings fail in schema version 1 rather than being silently disposed.

### 16.10 Terminal aggregates

Completed and faulted aggregates remain terminal. An ordinary non-null dispatch is
rejected under the stored source definition before automatic host migration begins.
Null inspection may continue using the source definition without migration.

`maintenance_mode` is a required Boolean migration-request member. Omission or a
non-Boolean value is `invalid_migration_request`. An explicit migration may advance a
terminal aggregate only when `maintenance_mode` is true and every descriptor's matching
terminal policy is `preserve`. A non-empty route against a terminal aggregate with
`maintenance_mode: false` is `terminal_migration_requires_maintenance`; a matching
policy of `reject` is `terminal_migration_rejected`. These failures preserve the exact
source envelope and produce no audit record. A completed migration preserves root
identity, completed status, counters, history, and diagnostics and creates no runtime
or emission. A faulted migration preserves the root fault including
`step_sequence`, the frozen diagnostic tree, every retained child status/value,
counters, and historical fault-definition anchors; it cannot reactivate any runtime.

If no allowed route exists, retaining the terminal source aggregate is valid while its
source definition remains registered. A host requiring a uniform target may quarantine
it. Version 1 has no recovery, restart, or destructive reset policy.

### 16.11 Lazy transactional host ordering

A persistence host supporting lazy migration follows this ordering:

1. Resolve and cryptographically verify the target definition, exact route, all
   descriptors, trust metadata, and resource requirements into a local immutable cache
   before taking the aggregate lock.
2. Begin a serializable transaction or acquire an observably equivalent aggregate
   compare-and-swap guard.
3. Lock/read the aggregate and the inbox receipt for the presented envelope.
4. If that idempotency key is already committed, return its recorded outcome without
   migration or dispatch.
5. Parse and verify the aggregate, resolve its source definition from the local cache,
   validate source state, and recheck route source.
6. Apply the complete route to an in-memory copy, validating every intermediate.
7. If the resulting aggregate is running and a delivery was supplied, invoke ordinary
   target-definition dispatch exactly once.
8. Build deterministic migration audit records and normal core result records.
9. Atomically replace aggregate bytes, record inbox disposition, insert ordered
   internal/external emissions in the outbox under their existing unique identities,
   and append audit rows.
10. Commit once, then acknowledge broker ingress or dispatch outbox work.

`handled`, `unhandled`, `rejected`, and `faulted` are core outcomes, not storage
failures. If the host contract commits an outcome, migration and that outcome commit
together. In particular, target-definition fault finalization cannot commit while its
preceding migration rolls back.

Definition/descriptor resolution, signature verification, and remote registry calls
MUST NOT occur inside the database transaction. A transaction conflict retries from
the newly committed row. Parent and owned-child state remains one aggregate transaction
boundary.

### 16.12 Failure, rollback, quarantine, and audit

The closed deterministic migration failure codes are:

- `invalid_aggregate_state`;
- `invalid_aggregate_state_package`;
- `aggregate_state_digest_mismatch`;
- `invalid_migration_request`;
- `source_definition_unavailable`;
- `target_definition_unavailable`;
- `definition_untrusted`;
- `definition_fingerprint_mismatch`;
- `migration_descriptor_untrusted`;
- `invalid_migration_descriptor`;
- `migration_route_missing`;
- `migration_route_mismatch`;
- `migration_transform_fault`;
- `migration_totality_failure`;
- `target_state_validation_failure`;
- `migration_resource_limit_exceeded`;
- `terminal_migration_requires_maintenance`; and
- `terminal_migration_rejected`.

The pure failure value is exactly `{ code: migration_failure_code }`; it has no
aggregate candidate or audit-record member. The caller retains the exact supplied
envelope. A successful result is exactly `{ aggregate_envelope, audit_records }`.

Artifact format/schema rejection occurs before this operation and uses
`unsupported_aggregate_state_format`,
`unsupported_aggregate_state_schema_version`,
`unsupported_migration_descriptor_format`,
`unsupported_migration_descriptor_schema_version`,
`unsupported_aggregate_state_package_format`, or
`unsupported_aggregate_state_package_schema_version`.
After a recognized package format/version, structural, attachment-uniqueness, or
cross-reference failure is `invalid_aggregate_state_package`. A recognized aggregate
or descriptor that fails its closed schema is respectively `invalid_aggregate_state`
or `invalid_migration_descriptor`.

`source_definition_unavailable` means the definition named by the stored aggregate
cannot be resolved. `target_definition_unavailable` means the requested target or an
intermediate non-source definition cannot be resolved. A definition whose bytes
reproduce its digest but whose digest is absent from the deployment allowlist/signed
manifest fails with `definition_untrusted`; it is never treated as unavailable or
implicitly trusted. A descriptor has the parallel
`migration_descriptor_untrusted` result.

Registry/cache unavailability, transaction conflict, and temporary storage failure are
transient host failures. Digest mismatch, unauthorized/invalid artifacts, invalid
request/route, transform/type/totality/target validation failure, terminal-policy
failure, and deterministic resource-limit failure are permanent for the same state,
request, and route.

Any descriptor failure discards every intermediate candidate. It writes no aggregate,
inbox success, outbox intent, successful audit, logical counter, or migration sequence.
For a permanent failure, a host atomically retains the exact original aggregate bytes,
records a quarantine marker and failure audit, and keeps the triggering inbox item
blocked. Quarantine is host metadata, not an aggregate lifecycle status and not a core
dead-letter collection. Resolution installs a new trusted route, restores an artifact,
or explicitly releases the blocked item.

Each successful descriptor returns exactly one closed audit record in route order:

```text
{
  migration_audit_record_schema_version: 1,
  root_instance_id: non_empty_string,
  root_runtime_id: non_empty_string,
  migration_sequence: canonical_decimal,
  source_validated_bundle_fingerprint: sha256_string,
  target_validated_bundle_fingerprint: sha256_string,
  migration_descriptor_digest: sha256_string,
  source_aggregate_state_digest: sha256_string,
  target_aggregate_state_digest: sha256_string,
  result_code: "migration_applied"
}
```

No other member is present. `migration_sequence` is the post-descriptor sequence.
Source/target state digests are the exact candidate digests immediately before and
after that descriptor. `examples/persistence/compatible-migration-audit.json` is the
normative one-hop result. Conformance compares the complete ordered record list.
Host observation time, worker identity, database transaction id, and operator metadata
may be stored alongside but outside this deterministic record. Success and quarantine
audit rows are append-only and transactional with the state they describe.

### 16.13 Package transport

A package contains exactly one aggregate, zero or more normalized definitions, zero or
more descriptors, and the exact route digest array. It is a transfer/archive artifact,
not the ordinary row format.

Every normalized definition attachment must reproduce its declared
validated-bundle fingerprint. Every descriptor must reproduce its declared descriptor
digest. Duplicate attachment digests are invalid. The route obeys §16.8 and every
artifact needed to decode the aggregate and execute that route must be available either
in the package or the receiving trusted registry. Package attachments seed the same
put-if-absent resolver contract; they never override a registry entry and are excluded
from the aggregate-state digest.

`examples/persistence/aggregate-state-package.json` is a normative one-hop vector. Its
source and target bundle fingerprints, shared shape fingerprint, aggregate digest,
descriptor digest, and canonical attachment trees are fixed by the accompanying
fixtures.

### 16.14 Security and resource limits

Digest verification is mandatory at every package/registry/cache boundary. Deployment
pins an allowed target and route, preventing silent downgrade or alternate-path
selection. Definitions and descriptors are immutable after trust admission. Source
definitions and historical fault definitions are retained while referenced.

Implementations expose configurable limits, but core conformance defines a minimum
supported floor for aggregate, definition, descriptor and transformed-output bytes;
JSON nesting; runtimes; active states; variables; map/list members; string bytes;
migration-chain length; descriptor rules; migration CEL expression length, AST nodes
and evaluation steps; and cumulative chain work.

The four descriptor `resource_requirements` members are non-negative
canonical-decimal upper bounds for one descriptor application:

- `maximum_transformed_output_bytes`: cumulative UTF-8 byte length of RFC 8785
  serialization of every typed value produced by `transform` and `initialize` rule
  occurrences; copied values and the surrounding aggregate envelope are not counted;
- `maximum_cel_expression_length`: cumulative UTF-8 source bytes of all distinct CEL
  expressions declared by the descriptor, counted once each;
- `maximum_cel_ast_nodes`: cumulative checked CEL abstract-syntax nodes across those
  distinct expressions, counted once each; and
- `maximum_cel_evaluation_steps`: cumulative evaluator steps across every expression
  occurrence, including repeated applicable runtimes/activations.

Static expression length/node requirements are checked before evaluation. Dynamic
evaluation-step and transformed-output counters start at zero for each descriptor,
advance in canonical runtime/rule occurrence order, and may not exceed either the
descriptor declaration or the implementation's configured limit. A descriptor that
understates actual use fails with `migration_resource_limit_exceeded`; the field is a
limit, not permission to truncate work.

A `compatible` descriptor has no CEL expressions and produces no transformed/initialized
typed values, so all four requirements MUST be `"0"`. Structural definition-binding
updates and final aggregate serialization are deliberately not transformed-output
bytes. A transform descriptor with only pointer/identity mappings may also use zero.
Cycles are forbidden. Exceeding any deterministic per-descriptor or cumulative route
limit fails closed with `migration_resource_limit_exceeded`.

This section does not standardize a production database schema, object-relational
mapper, registry transport, queue plugin, distributed transaction, exactly-once
external delivery, package import, or bulk row rewrite. A later runnable database
example belongs in the separate examples repository after conformance and both engines
implement this contract.
