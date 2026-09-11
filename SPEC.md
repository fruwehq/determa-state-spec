# Determa State — specification

Status: **pre-release alpha**. Normative unless a section says informative.
Document format: **1**.
Spec version: **0.2.0** (see `VERSION`; synchronized across the Determa State
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
7. deterministic identities, lifecycle, rollback, and fault results; and
8. runtime-local ready and deferred mailboxes for lossless continuation.

The core deliberately does **not** provide:

- an external transport queue, delivery worker, scheduler, or background thread;
- dead-letter storage or a dead-letter policy;
- a clock, timer, delay, sleep, or time-event implementation;
- transport, broker, retry, acknowledgement, or delivery guarantees;
- a state store, transaction manager, credentials, or external I/O; or
- plugin discovery, installation, configuration schemas, or package resolution.

Those are host or plugin responsibilities (§11). The core can admit an envelope to an
isolated runtime mailbox or receive one envelope for immediate foreground processing,
processes at most one targeted RTC step at a time, and returns state plus ordered
emissions. It never calls an external queue, timer, broker, database, or remote service
from a guard or action.

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

Determa State repository/package releases 0.0.1 through 0.0.6 emitted a legacy machine
grammar and snapshot format. Those artifacts are frozen legacy artifacts and MUST NOT
be migrated into format 1. Release 0.0.7 introduced format 1; format-1 definitions
produced by 0.0.7 remain ordinary format-1 inputs and are accepted or rejected by the
current §2 source checks, schema, and §5 semantic validation. These historical release
numbers describe the support policy only. Repository/package SemVer remains distinct
from the machine `format` discriminator and is never inferred from document shape.

A machine loader applies the format check above without release-provenance or shape
detection. A definition produced by releases 0.0.1 through 0.0.6 with a missing
discriminator or a discriminator other than the YAML/JSON integer `1` fails
`unsupported_format`. If a legacy-shaped document explicitly carries `format: 1`, the
discriminator succeeds and the loader proceeds to the current schema; the legacy
structure MUST be rejected by JSON Schema (`structural_validation` in §5.1
conformance-harness notation) before §5 semantic validation. The loader MUST NOT
identify a legacy document by its other fields, supply omitted fields, select a legacy
parser, or silently reinterpret it as format 1.

There is no 0.0.1-through-0.0.6 definition converter or legacy-snapshot import in this
specification. Users migrating from those releases MUST author a new format-1 bundle
and MUST NOT carry a legacy snapshot forward. The definition migration rules in §16
apply only between recognized format-1 bundles and their format-1 portable artifacts.

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
- `deferred_events` — optional non-empty unique list of declared events that this state
  defers under §6.7 when no enabled handler exists in the active hierarchy.
- `deferred_event_capacity` — optional non-negative signed-64-bit capacity for the
  runtime-local deferred mailbox; valid only on a machine root or inline-component
  root. Omission means logically unbounded.
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

Native time events (`after`), orthogonal regions with implicit broadcast, submachine
documents, and completion activities are not part of format 1.

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

Internal sends do not recursively dispatch. Version-1 direct dispatch returns them to
the host in emission order. Version-2 processing appends them to exact target runtime
ready mailboxes in that same order as part of the producing RTC commit.

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
- a `deferred_events` name that is undeclared, is an output event, or is one of the
  reserved lifecycle/control events `done`, `determa.component_completed`,
  `determa.component_failed`, or `determa.spawned_instance_failed`, or a
  `deferred_event_capacity` outside a runtime root;
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

A queue-bearing version-2 mailbox envelope additionally stores required `cause_id` and
`source`. Host input uses `cause_id: event_id` and `source: { host: true }`. Internal
emissions preserve their existing deterministic cause and use
`source: { runtime: target_identity_of_emitter }`; system emissions use their exact
`system:` source locator. These fields make the already defined logical provenance
portable; they do not change version-1 envelope bytes.

Direct aggregate-state version-1 `dispatch` never transfers envelope ownership to the
core because that artifact has no mailbox. It validates and classifies the exact
caller-owned envelope for one immediate call. If classification is `deferred`, it
returns the byte-for-byte prior aggregate, no emissions, and the caller retains the
exact envelope for later resubmission. Version 1 performs no automatic recall and makes
no portable ordering or durability claim for caller-retained deferred work.

Portable automatic deferral therefore requires queue-bearing aggregate/checkpoint
version 2. The caller owns a version-2 envelope until mailbox admission commits. Before
that boundary, the core atomically validates the delivery mode, event declaration,
direction, payload, correlation, exact target incarnation, and target eligibility.
Rejection returns the prior state unchanged, allocates no logical identity, and leaves
the envelope caller-owned. After committed admission the envelope exists in exactly one
engine-owned lifecycle location: one runtime's ready mailbox, that same runtime's
deferred mailbox, or a terminal/disposal receipt under §17.15.

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

One direct `dispatch` examines at most one caller-owned envelope and one mailbox `step`
processes at most one accepted envelope, in either case for one explicitly addressed
runtime. The step is non-reentrant and atomic. Internal emissions append to exact target
runtime ready mailboxes only if the operation uses queue-bearing aggregate-state version
2; direct aggregate-state version 1 dispatch continues to return them to its caller.
External emissions remain output intents. No operation chooses another runtime, drains
an aggregate, or introduces a round-robin scheduler.

For a version-2 operation, a successful internal `send` action resolves and validates
its exact target incarnation, allocates the next acceptance and queue sequences, and
tentatively appends the complete entry to that target's ready mailbox at that action's
position in emission order. The entry is visible to later lifecycle cleanup in the same
RTC but is never dispatched recursively. The enclosing RTC commit makes the append
durable; rollback removes it and restores both counters.

If the target is already completed, faulted, pending disposal, or absent when the send
action executes, the sender faults with the existing exact target code. If the target
remains retained-faulted after a later fault in the same RTC, the entry remains in its
frozen ready mailbox. If later same-RTC cancellation, natural completion, or aggregate
completion disposes the target, cleanup removes the entry and records
one core `lifecycle_disposition` under §8. The producing emission becomes an
`internal_disposed` reference to that record rather than an `internal_mailbox`
reference. A committed result therefore contains either one mailbox entry or one
disposition for the emitted envelope, never both and never neither.

The same-RTC target outcome is closed:

| target state at enclosing RTC commit | emitted-envelope result |
|---|---|
| running or pending initialization | one ready-mailbox entry plus `internal_mailbox` reference |
| retained faulted/frozen without disposal | one frozen ready-mailbox entry plus `internal_mailbox` reference |
| successfully cancelled, completed, or aggregate-disposed | no mailbox entry; one lifecycle disposition plus `internal_disposed` reference |
| lifecycle cleanup or any enclosing action faults | complete enclosing RTC rollback; no entry, disposition, emission, or counter allocation commits |

No transition between these rows may create a second copy or an unreferenced committed
envelope.

Other runtimes may be processed concurrently only when the host's persistence layout
provides serializable ownership of any aggregate state they might share. Two calls MUST
NOT concurrently mutate the same root ownership aggregate.

### 6.3 Hierarchical dispatch

The envelope is resolved one state level at a time from the deepest active state to the
runtime root. At each level the engine first evaluates that state's declared handler,
if any, using its ordered branches. An enabled branch handles the event. If no branch at
that level is enabled and that same state declares the event in `deferred_events`, the
event is deferred immediately. Only when neither result applies does resolution continue
to the parent.

- A false guard does not consume the envelope. After every branch at that level is
  false, same-state deferral is checked before ancestor search.
- A guard evaluation error faults the step.
- The first true branch in an ordered list wins.
- If a state has no enabled branch and defers the event, resolution stops with
  `deferred`; no ancestor handler is evaluated.
- If the root has neither an enabled branch nor a matching deferral, the disposition is
  `unhandled`.

The only exception is an unhandled `determa.component_failed` or
`determa.spawned_instance_failed` envelope, which faults the owner as specified in
§10.2.

Determa deliberately gives ordered guard branches declaration-order priority: the
first true branch wins, and guards need not be mutually exclusive. UML state-machine
models do not assign this priority to competing guarded transitions. Authors porting a
UML model MUST NOT assume guard-order independence. An unguarded default, when present,
MUST be last, which keeps the priority unambiguous. Without a default, all-false guards
continue ancestor search.

The closed per-level precedence table is:

| current state level | handler result at this level | defers at this level | result |
|---|---|---|---|
| any | first unguarded or true branch | absent or present | `handled`; same-state handler wins |
| any | every declared branch guard is false | present | `deferred` at this level |
| any | no handler declaration | present | `deferred` at this level |
| non-root | every branch false or no handler | absent | continue to parent |
| root | every branch false or no handler | absent | `unhandled` |
| any | an evaluated guard faults | either | atomic engine fault; deferral is not consulted |

Thus a child handler overrides a parent deferral because the child handles before the
parent is reached. A child deferral overrides a parent handler because resolution stops
at the child. At one state, an enabled handler overrides that state's deferral, while an
all-false handler plus that state's deferral defers. The rule considers only the
addressed runtime's ordinary state hierarchy; components and owned spawned runtimes have
isolated configurations and mailboxes. Explicit fan-out creates independent envelopes
that are resolved independently.

An unhandled result is not a core fault. Under version 2 the accepted envelope is
consumed into its terminal receipt and is not retained in logical state. Under direct
version-1 dispatch the core retains no copy and the caller continues to own the supplied
envelope under the pre-existing delivery boundary. A host may separately audit or
dead-letter that terminal disposition, but no transport plugin may reinterpret it as
machine deferral.

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

### 6.7 Deferred mailboxes and automatic recall

The mailbox and automatic-recall rules in this subsection apply only to queue-bearing
aggregate-state version 2. Direct version-1 `dispatch` has the caller-owned `deferred`
result defined in §6.1 and does not retain or recall the envelope.

In version 2 each runtime has one FIFO ready mailbox and one FIFO deferred mailbox. They
are isolated from every other runtime even when the runtimes share one ownership
aggregate. A `deferred` result atomically removes the selected ready entry, increments
its `deferral_count`, allocates a new `queue_sequence`, and appends it to that runtime's
deferred tail. The immutable `acceptance_sequence` and the complete normalized envelope,
including `event_id`, `cause_id`, source, exact target incarnation, payload, and optional
`correlation_id`, do not change. Deferral never creates an emission or a second event.

After every successful `handled` RTC step has completed all transition actions, choice
resolution, exit/entry behavior, initial descent, completion behavior, and lifecycle
cleanup, the engine performs one bounded structural recall phase against the resulting
stable configuration. A `deferred` classification is not a handled RTC step and does not
immediately recall the entry it just deferred. The phase freezes the deferred entries
present at its start and examines each exactly once in existing `queue_sequence` order.
It evaluates no guard, action, or other expression.

For each frozen entry, structural recall walks the active states deepest-to-root without
evaluating guards and applies this closed rule at each level:

| current level structure | recall action |
|---|---|
| handler declaration present, with or without same-state deferral | move to ready tail; stop structural walk |
| no handler declaration and matching deferral present | remain deferred; stop structural walk |
| neither declaration at a non-root level | continue to parent |
| neither declaration at the root | move to ready tail as no longer deferred |

Every moved entry receives a fresh `queue_sequence` and appends to the ready tail in the
same relative order. Existing ready entries and entries emitted by the just-completed
RTC remain ahead of recalled entries. Entries not in the frozen snapshot are not
examined. Recall does not dispatch recursively, consume a logical-step sequence, or
fault the successful RTC merely because a deferred guard would fault if evaluated.

When a recalled entry later reaches the ready head, the engine performs the complete
§6.3 level-by-level evaluation. An enabled handler wins over a deferral on that same
state. An all-false handler with a same-state deferral, or a deeper deferral reached
before an ancestor handler, moves the entry back to the deferred tail. A guard fault
faults that selected event's own step. A later successful handled RTC may make it
structurally recall-eligible again. This repeated movement is bounded to one examination
per recall phase and preserves envelope and acceptance identity while each queue
placement receives a new scheduling identity.

`deferred_event_capacity` limits the number of entries in that runtime's deferred
mailbox after the attempted append. Omission is logically unbounded; `0` forbids every
deferral. An overflow rolls back the attempted RTC/classification, consumes the causal
entry into a terminal `faulted` receipt with code `deferred_event_capacity_exceeded`,
and applies the normal root or contained-runtime fault rule. The event is never dropped,
left at the ready head for an infinite retry, or delegated to a plugin overflow policy.

Deferred events do not expire. A successful runtime cancellation, natural completion,
or aggregate completion disposes every remaining ready and deferred entry in canonical
queue order and creates one terminal `disposed` receipt per entry with respectively
`runtime_cancelled`, `runtime_completed`, or `aggregate_completed`. Those receipts are
part of the same atomic lifecycle commit. A cleanup fault rolls back all disposal
records and mailbox removal; fault-frozen diagnostic runtimes retain their mailboxes
unchanged until a later successful owner cancellation disposes them. Reserved completion
and contained-failure events cannot be declared deferred, so machine behavior cannot
hold them indefinitely.

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

Components have isolated configurations, variables, and ready/deferred mailboxes. An
event reaches a component only through an explicit emission targeting its nominal
component runtime identity. No parent, sibling, or owned child participates in its
deferral decision.

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

Under aggregate-state version 2, both notifications append to the owner ready mailbox in
the committed emission order, after work already there; under version 1 direct dispatch
they remain returned emissions. In either representation,
`determa.component_completed` precedes `done`.

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
child or a child whose holding reference remains in scope is processed only when an
explicit envelope targets it and the host invokes direct dispatch or a mailbox `step`
for that exact runtime.

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

create_v2(bundle, machine_id, root_instance_id, creation_id, bindings)
  -> { status, state, emissions, lifecycle_dispositions, fault, rejection }

dispatch(bundle, prior_state, delivery?)
  -> { status, disposition, state, emissions, fault, rejection }

admit(bundle, prior_state_v2, ordered_deliveries)
  -> { status, accepted, state, rejection }

step(bundle, prior_state_v2, target_runtime_id)
  -> { status, disposition, state, emissions, lifecycle_dispositions, fault, rejection }

delivery =
  { input: envelope }
  | { internal: envelope }
  | null
```

The two delivery members are a closed tagged union. `input` applies the public-ingress
rules in §6.1; `internal` applies the internal-delivery rules. A non-null delivery has
exactly one member. Null is the read-only inspection call.

`admit` validates its complete ordered batch before mutation, then appends each envelope
to its exact target runtime's ready tail in caller order. It allocates immutable
aggregate-wide `acceptance_sequence` and mutable `queue_sequence` values independently.
If any member is invalid, the entire batch is rejected byte-for-byte. `step` names one
exact running runtime and processes only its ready head; an empty ready mailbox returns
`not_runnable` without mutation. Neither operation selects or advances another runtime.

Before mailbox selection, `step` resolves its target with this closed rule:

| target resolution | result |
|---|---|
| exact retained component runtime whose status is `completed` or `faulted`, or that is otherwise inactive pending owner disposal | reject with `inactive_component_target` |
| absent or disposed runtime identity, or a root/spawned machine runtime that is completed, faulted, or otherwise non-targetable | reject with `invalid_instance_target` |
| exact running runtime with an empty ready mailbox | `not_runnable` with null `rejection` |
| exact running runtime with a ready head | perform the one atomic mailbox step |

The first two rows are pre-step rejection outcomes: they preserve the prior aggregate
byte-for-byte, allocate nothing, and return no emissions or lifecycle dispositions. A
retained inactive component receives the component-specific code because its exact
activation identity is still present for diagnosis. Once that component has been
disposed and is no longer retained, the same stale identity falls into the absent-runtime
row and uses `invalid_instance_target`. Root and spawned runtimes never use
`inactive_component_target`.

`create` retains its existing aggregate-state version-1 result. `create_v2` is the only
fresh version-2 creation selection; hosts MUST choose it explicitly rather than infer it
from later persistence. Before author initialization it sets aggregate next acceptance
and queue sequences to zero and creates every runtime with empty ready/deferred
mailboxes. Each initialization internal emission then allocates from those counters in
emission order and starts with `deferral_count: "0"`; absent such emissions both next
counters remain zero. Creation-time target disposal follows the same disposition rule
as a version-2 `step`.

`status` is `running`, `completed`, or `faulted` for an existing aggregate. A creation
rejected before an aggregate exists returns `status: rejected` and `state: null`.
Dispatch rejection or an unhandled envelope preserves the prior aggregate status.

Every named result field is present. For version-1 direct dispatch, `emissions` is the
ordered immutable emission list returned by the call. For version-2 operations it
contains full external intents plus `internal_mailbox` or `internal_disposed` references
only; an internal envelope's deliverable copy exists solely in its target mailbox or is
accounted by the referenced lifecycle disposition. `rejection` is null
except on pre-step rejection, where it is exactly
`{ code: rejection_code }`. `fault` is:

- the aggregate root's committed fault record when the aggregate root is faulted;
- otherwise the target runtime fault newly committed by a `faulted` dispatch; or
- null.

A contained fault committed inside an otherwise successful owner RTC appears only in
the contained-runtime state and its reserved failure emission; it does not populate the
top-level `fault`. There is no plural `faults` result field.

Every `create_v2` and `step` result also contains `lifecycle_dispositions`, including
the empty list; rejected creation returns the empty list. One entry contains exactly `event_id`, `request_digest`,
`acceptance_sequence`, `final_queue_sequence`, `target_runtime_id`, and lifecycle
`reason`. It accounts for every ready/deferred entry removed by successful lifecycle
cleanup, including internal work emitted earlier in the same RTC. Entries use lifecycle
cleanup runtime order, then ready entries followed by deferred entries in queue order.
The closed structural schema for `step` is `schema/core-step-result-v2.schema.json`;
semantic status/disposition/fault/rejection relationships remain mandatory. The
creation result retains the creation-specific status/state shape above. Direct
version-1 results have no lifecycle-disposition field and retain their existing shape.

Creation rejection codes are exactly `invalid_creation_request`,
`invalid_machine_target`, and `invalid_binding`. Dispatch rejection codes are exactly
`invalid_event`, `invalid_payload`, `invalid_correlation`,
`invalid_instance_target`, `inactive_component_target`, `invalid_prior_state`, and
`incompatible_bundle`. The closed version-2 `step` pre-step rejection subset is exactly
`invalid_instance_target`, `inactive_component_target`, `invalid_prior_state`, and
`incompatible_bundle`; admission has the larger closed set in §17.15. Bundle parsing,
schema, and semantic load failures happen before these calls and use §2/§5 codes. A
rejection commits no fault record.

`disposition` is exactly:

- `handled` — the accepted envelope completed a successful RTC step;
- `deferred` — no handler at the resolving state level was enabled and that state
  deferred the event;
- `unhandled` — no enabled handler or active deferral declaration existed;
- `not_runnable` — a mailbox `step` named a running runtime with an empty ready mailbox;
- `rejected` — validation failed before an RTC step; or
- `faulted` — an engine fault occurred during the RTC step.

Deferral ownership is total across the two operation profiles:

| operation and artifact | classification | committed ownership/result |
|---|---|---|
| direct `dispatch`, aggregate-state version 1 | `deferred` | prior aggregate unchanged; exact envelope remains caller-owned |
| mailbox `step`, aggregate-state version 2 | `deferred` | selected ready entry moves to that runtime's deferred mailbox |
| version-2 structural recall | eligible | deferred entry moves once to that runtime's ready tail |
| version-2 structural recall | ineligible | deferred entry remains in its existing position |

Only the version-2 rows provide portable automatic retention and recall. A version-1
caller that discards a `deferred` result discards its own envelope; the core has not
claimed acceptance or retained a hidden copy.

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
- for the queue-bearing abstract aggregate, every runtime's isolated ready and deferred
  mailboxes plus aggregate acceptance and queue-placement counters; and
- the `validated_bundle_fingerprint` defined below.

It contains no external broker backlog, dead-letter collection, timer, broker
acknowledgement token, credential, transport receipt, or plugin configuration.
Aggregate-state schema version 1 can encode only an abstract aggregate whose runtime
mailboxes are empty; schema version 2 is required whenever accepted work is ready or
deferred. This restriction does not reinterpret any version-1 byte.

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
- `deferred_event_capacity_exceeded`;
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
| `deferred_event_capacity_exceeded` | exactly `system:deferred_event_capacity` |
| `contained_runtime_fault` | exactly `system:unhandled_contained_failure` |
| `cascade_fault` | exactly `system:cascade_cleanup` |
| `invariant_fault` | exactly `system:invariant` |

For a `targets` list, the pointer includes the failing zero-based list index. A
contained failure notification embeds the §4.4 public projection of the child fault
record; if its delivery later faults the owner, the owner receives a separate retained
record with the system locator above. Rejected pre-step envelopes and rejected creation
have no committed fault record and therefore no `source_locator`.

For version-2 deferred-capacity overflow, selection reserves the current aggregate
`next_logical_step_sequence` before classification. The failed deferred append and any
tentative queue-sequence allocation roll back completely. Fault finalization consumes
the selected ready entry, uses its immutable envelope `cause_id`, records code
`deferred_event_capacity_exceeded`, the reserved step sequence, and locator
`system:deferred_event_capacity`, then advances `next_logical_step_sequence` exactly
once. `next_queue_sequence` is unchanged because no new queue placement committed. No
author action or external intent ran. The checkpoint host creates the selected event's
terminal `faulted` receipt in that same commit; root or contained-fault propagation then
follows the ordinary rules.

For direct version-1 dispatch, the caller still owns a faulting envelope and receives
`disposition: faulted`, preserving the established version-1 boundary. For a previously
accepted version-2 mailbox entry, fault finalization removes that entry and creates its
terminal `faulted` receipt in the same commit. Other ready/deferred entries remain in
their exact locations; a root fault freezes them, while a contained fault freezes only
that runtime subtree as §6.7 defines. A transport plugin cannot retry or discard an
engine-owned mailbox entry independently.

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

A transport or audit plugin may:

- discard unhandled or faulting envelopes without retaining anything;
- retain complete envelopes and fault metadata;
- retain metadata without payloads;
- retry before retention;
- forward to a broker-native dead-letter facility; or
- expose any other explicitly configured policy.

Its property names, configuration schema, retention, privacy, and operational guarantees
belong entirely to that plugin. These policies apply only after terminal machine
disposition or before Determa acceptance; they cannot replace, reorder, expire, or cap a
runtime's normative ready/deferred mailboxes.

## 11. Plugins and hosting

This section defines the core boundary. The optional portable execution-checkpoint
hosting contract, adapter registration behavior, and durability capabilities are
defined in §17. Neither section defines a cross-language plugin ABI.

### 11.1 Transport queue plugins

A transport queue plugin owns external backlog until a host commits admission into a
Determa aggregate. It may later receive external output intents. The core does not
standardize a concrete plugin API, but acceptance must preserve each envelope's
immutable value and identity and transfer ownership exactly once.

Plugins may differ in:

- ordering and fan-out;
- in-memory or durable storage;
- delivery attempts and acknowledgements;
- duplicate delivery and deduplication;
- retry, delay, and dead-letter behavior outside the accepted machine mailbox;
- transactional integration with aggregate persistence; and
- capacity, overflow, backpressure, and availability.

Core determinism means that the same valid prior state and same accepted sequence
produce the same mailbox state and result. Different transport plugins may produce
different admission traces, but after acceptance they cannot alter §6.7 deferral,
recall, ordering, capacity, or disposition semantics.

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

Runtime-local ready/deferred mailbox contents and their portable ordering identities are
core state and SHOULD be inspectable. External broker backlog, delivery attempts, dead
letters, scheduled jobs, and broker acknowledgements remain plugin-owned.

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
| deferral and dead letters | UML-style runtime-local deferral is portable; dead-letter storage remains host policy |
| owned spawning | retained for same-bundle machines with nominal `instance_reference` values |
| submachines and package imports | unsupported |
| definition migration/hot-swap | explicit portable aggregate migration under §16; never implicit in ordinary dispatch |
| observers and export | read-only inspection and visualization are recommended, not executable core behavior |
| snapshots | closed portable aggregate-state envelope and package under §16 |
| stores and CLI protocols | host/implementation concerns, not bundle grammar |

The pre-release format deliberately omits:

- native timers, clocks, `after`, sleeps, and time-triggered transitions;
- external transport queues, retries, acknowledgement, or dead-letter storage;
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
- child-handler-over-parent-deferral, child-deferral-over-parent-handler, and same-state
  handler-versus-deferral precedence, including true, false, all-false, and faulting
  guards;
- repeated deferral, selective FIFO recall after stable RTC, reclassification at the
  ready head, capacity overflow, and exact envelope-identity preservation;
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
- runtime-local mailbox isolation across root, component, and spawned targets; and
- explicit absence of timers, external transport queues, and dead-letter fields from
  core state.

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
- queue-bearing aggregate/checkpoint version-2 round trips, version-1 immutability,
  event acceptance/terminal receipt separation, and ready/deferred migration totality;
- retry-identical success or failure, complete rollback, and no counter consumption;
- migration followed by handled, unhandled, rejected, and faulted dispatch in one
  host transaction;
- terminal completed/faulted maintenance migration without reactivation or emission;
- exact terminal maintenance/policy failures and definition/descriptor authorization
  failures; and
- descriptor-declared and cumulative resource-limit accounting.

Execution-checkpoint profile conformance additionally requires:

- exact checkpoint and envelope digests plus byte-stable canonical round trips;
- version-1-to-version-2 conversion with wrapped legacy receipts, converted internal
  origins, distinct digest domains, zero-based mailbox counters, and no stale digest
  equality claim;
- same-RTC internal-send retention, frozen-target retention, successful lifecycle
  disposal, and rollback cases with exactly one mailbox/disposition result;
- duplicate event ids within one batch, equal/conflicting terminal replay after root
  completion/fault/tombstone, event-identity tombstone compaction, and dependency-closed
  pruning;
- exact deferred-capacity fault locator, allocation rollback, causal consumption, and
  root/contained finalization;
- reusable queue migration disposal selectors, arbitrary migration reasons, reduced
  capacity totality, and retained-faulted mailbox preservation without recall;
- closed-schema and semantic rejection for identity, root, ordering, counter, outcome,
  revision, and digest inconsistencies;
- durable host-input acceptance, unified host/internal sequence ordering, pending
  same-content replay, pending/committed disjointness, and every unequal-content
  conflict;
- every closed pre-acceptance failure, including replay-before-tombstone ordering and
  proof that no failed acceptance mutates or acknowledges;
- creation, handled/unhandled/rejected/faulted delivery, applied/no-op maintenance, and
  tombstone idempotency receipts, including every otherwise-case;
- explicit receipt-versus-§8-result boundaries and optional same-transaction
  application-response replay;
- permanent/bounded retention, irreversible pruning history, terminal
  checkpoint/tombstone retention in both modes, dependency-closed pruning, restore
  completeness, and root-identity no-reuse;
- rejection of physical checkpoint/root-marker deletion in schema version 1,
  including bounded mode and backup/restore;
- exact accepted/committed revision equations for delayed, foreground, creation, and
  internally emitted deliveries, including every impossible ordering;
- atomic aggregate/pending-delivery/receipt/outbox/audit/revision replacement at every
  injected pre-commit crash point;
- post-commit/pre-acknowledgement replay without redispatch or duplicate insertion;
- embedded foreground accept/process and delayed processing with equal committed
  results;
- all pending outbox states (`not_attempted`, `retryable_failure`, `ambiguous`) and all
  terminal outcomes (`confirmed`, `permanently_rejected`, `operator_cancelled`,
  `discarded`, `dead_lettered`), equal-state update replay, compact effect tombstones,
  and silent-deletion rejection;
- one-winner concurrent revision updates without lost writes;
- built-in and synthetic third-party registration through one public route;
- registry-free direct injection and mandatory registry use for every offered
  scheme/identifier resolution;
- deterministic unknown, duplicate, invalid-configuration, and capability-mismatch
  execution-store failures; and
- truthful store-capability versus composed-host-profile negotiation, including
  durable-profile rejection of memory and rejection of store-only broker claims.

Quarantine storage and local immutable-cache mechanics remain profile-owned host
details outside the checkpoint artifact.

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

Machine documents remain numeric `format: 1`. Persistence introduces four independent
closed JSON artifacts:

| artifact | exact format field | exact schema-version field |
|---|---|---|
| aggregate-state envelope | `aggregate_state_format: "determa.aggregate_state"` | `aggregate_state_schema_version: 1` |
| migration descriptor | `migration_descriptor_format: "determa.aggregate_migration"` | `migration_descriptor_schema_version: 1` |
| transport package | `aggregate_state_package_format: "determa.aggregate_state_package"` | `aggregate_state_package_schema_version: 1` |
| execution checkpoint | `execution_checkpoint_format: "determa.execution_checkpoint"` | `execution_checkpoint_schema_version: 1` |

Queue-bearing continuation adds aggregate-state, migration-descriptor,
aggregate-state-package, and execution-checkpoint schema version `2` under the same
respective format discriminators. Version 1 of every artifact above remains immutable;
version 2 is selected explicitly and is never inferred from fields.

These wire schema versions, machine format, repository/package SemVer, launcher
SemVer, and author-controlled machine `version` are independent version domains.
Unknown artifact formats or schema versions are rejected before semantic validation;
there is no nearest-version parsing, implicit conversion, or best-effort field
retention.

In particular, snapshots produced by releases 0.0.1 through 0.0.6 are not modern
portable artifacts. The caller or host MUST select one modern decoder before decoding;
the selected decoder MUST NOT probe other artifact kinds or infer one from field shape.
A legacy snapshot, including one with the selected decoder's discriminator absent,
MUST be rejected before schema or semantic validation with the decoder's exact closed
format code:

- aggregate-state decoding uses `unsupported_aggregate_state_format` (§16.12);
- migration-descriptor decoding uses `unsupported_migration_descriptor_format`
  (§16.12);
- aggregate-state-package decoding, when that decoder was selected, uses
  `unsupported_aggregate_state_package_format` (§16.12); and
- execution-checkpoint restoration uses `unsupported_execution_checkpoint_format`
  (§17.1).

A decoder MUST NOT fall through to another decoder, treat a legacy snapshot as a
format-1 artifact, or attempt a migration.

The artifacts MUST be JSON encoded as strict UTF-8. Their parsers apply the §2
source-level duplicate-name, acyclic JSON-value, Unicode-scalar, Boolean, null, and
finite-number requirements before the applicable schema. YAML is not a portable
encoding for these artifacts. Every schema is closed: an unknown member is invalid.
The exact structural schemas are:

- `schema/aggregate-state.schema.json`;
- `schema/migration-descriptor.schema.json`; and
- `schema/aggregate-state-package.schema.json`; and
- `schema/execution-checkpoint.schema.json`.

The corresponding version-2 schemas are `schema/aggregate-state-v2.schema.json`,
`schema/migration-descriptor-v2.schema.json`,
`schema/aggregate-state-package-v2.schema.json`, and
`schema/execution-checkpoint-v2.schema.json`. Queue-bearing core results use
`schema/core-step-result-v2.schema.json`.

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

`target_identity` embeds the exact normalized format-1 §6.1 mathematical target value
but uses artifact-owned decimal-string projections for its integer-valued members.
A spawned target's `machine_version` is a positive signed-64-bit canonical decimal
string. A component target's `activation_sequence` is an unbounded non-negative
canonical decimal string. Numeric JSON forms are invalid even when their values would
be exactly representable. Decoding reconstructs the mathematical integers before
target equality, reference equality, routing, or dispatch; the decimal-string
projection is not a change to format-1 identity semantics. Origin, current relation,
and counter records use the same artifact-owned decimal strings and may carry the
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
the component target is exactly §6.1. On the wire, spawned `machine_version` and
component `activation_sequence` use the §16.2 decimal-string projections. A decoder
MUST reconstruct their mathematical integer values before comparing them with
in-memory references or targets and before using them for routing or dispatch.

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
- `migration_resource_limit_exceeded`;
- `terminal_migration_requires_maintenance`; and
- `terminal_migration_rejected`.

The pure failure value is exactly `{ code: migration_failure_code }`; it has no
aggregate candidate or audit-record member. The caller retains the exact supplied
envelope. A successful result is exactly `{ aggregate_envelope, audit_records }`.

Failure to produce a complete state valid under the target definition, including an
invalid target type, relationship, pointer, configuration, or required value, is
`migration_totality_failure`. A CEL evaluation or other runtime failure while
executing a statically valid transform is `migration_transform_fault`. There is no
separate target-state-validation result code.

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
migration-chain length; descriptor rules; and migration CEL expression length, AST
nodes, and evaluation steps.

Migration-chain length is exactly the number of descriptor digests in the requested
route. Before applying any descriptor, the operation compares that count with the
implementation's configured migration-chain-length limit. Exceeding it fails with
`migration_resource_limit_exceeded`. The empty route has length zero.

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
Cycles are forbidden. Each descriptor is checked independently against its declared
four bounds and the corresponding configured per-descriptor limits. A route is bounded
by its exact descriptor count plus those independent per-descriptor checks; schema
version 1 defines no mixed-unit cumulative-chain-work counter and does not sum resource
dimensions across descriptors. Exceeding the chain-length limit or any deterministic
per-descriptor limit fails closed with `migration_resource_limit_exceeded`.

This section does not standardize a production database schema, object-relational
mapper, registry transport, queue plugin, distributed transaction, exactly-once
external delivery, package import, or bulk row rewrite. A later runnable database
example belongs in the separate examples repository after conformance and both engines
implement this contract.

### 16.15 Queue-bearing artifact version 2

Aggregate-state schema version 2 preserves every version-1 field and adds exactly
aggregate `next_acceptance_sequence` and `next_queue_sequence`, plus each runtime's
`ready_mailbox` and `deferred_mailbox`. Each mailbox entry contains immutable
`acceptance_sequence`, current `queue_sequence`, delivery mode, complete normalized
envelope, envelope digest, and `deferral_count`. Entries in each mailbox are strictly
increasing by mathematical `queue_sequence`; acceptance sequences and event ids are
unique across all runtime mailboxes. Both next counters exceed every allocated value and
are never reduced or reused. One entry occurs in exactly one mailbox.

Each mailbox entry's envelope digest is:

```text
envelope_digest = hash([
  "determa-inbox-envelope-digest-2",
  "2",
  root_instance_id,
  delivery_mode,
  envelope
])
```

The digest is also the acceptance receipt's `request_digest` and the eventual terminal
event receipt's `request_digest`.

Version-2 aggregate serialization uses the §16.2 canonical rules and:

```text
aggregate_state_digest = hash([
  "determa-aggregate-state-digest-2",
  envelope_without_aggregate_state_digest
])
```

The complete logical checkpoint is the configuration, variables, identities,
ready/deferred entries, counters, receipts, and participating output state. A host may
store those records in one document or physically separate tables/objects, but one read
must reconstruct one schema-valid revision and one commit must replace it atomically or
with observably equivalent serializable compare-and-swap behavior. An enum or relational
projection that cannot preserve or exactly reconstruct every required member MUST reject
before any application row, mailbox, receipt, or outbox mutation; it never drops an
unrepresented field. Ephemeral in-memory ownership is conforming when no durability is
claimed.

Aggregate-state version 1 decodes exactly as before and has no mailboxes. Explicit
upgrade `1 -> 2` is permitted only by copying every version-1 logical field, inserting
zero next acceptance/queue counters and empty mailboxes, setting schema version 2, and
recomputing the version-2 digest. Downgrade `2 -> 1` is permitted only when both counters
are zero and every mailbox is empty; otherwise it fails `migration_totality_failure`.
No decoder performs either conversion implicitly.

Migration-descriptor schema version 2 contains one complete version-1 base descriptor,
the fixed `queued_event_default: preserve_if_compatible`, and reusable
`queued_event_rules`. A rule is an explicit disposal override keyed by exact source
`machine_id`, event name, and delivery mode and carries `action: dispose` plus a
non-empty operator reason. Duplicate selectors are invalid even when their reasons are
equal. The descriptor is definition-pair-specific but backlog-independent; it MUST NOT
list event ids, runtime ids, acceptance sequences, or queue sequences.

For every source mailbox entry, one matching disposal override removes it. Otherwise
the fixed default preserves the exact envelope, acceptance identity, current queue
identity, location, and deferral count only when the mapped target incarnation still
exists and the target definition accepts the exact event direction, correlation
contract, and normalized payload without coercion. No match plus any incompatibility is
`migration_totality_failure`; the source remains byte-for-byte unchanged. A disposal
creates terminal outcome `migration_disposed` with the rule's exact reason and this
descriptor's digest. It does not use lifecycle-only `disposed` reasons.

The version-2 descriptor digest is independently recomputed as:

```text
migration_descriptor_digest = hash([
  "determa-migration-descriptor-2",
  descriptor_without_migration_descriptor_digest
])
```

After every successful descriptor, preserved deferred entries use the bounded structural
eligibility table in §6.7 against the mapped stable configuration. No guard or action is
evaluated. Entries not structurally eligible remain in relative order; eligible entries
move to the ready tail in prior deferred order with fresh queue sequences. Changing only
a deferral declaration is therefore explicit and deterministic, not silent event loss.
Migration executes no handler action and emits no event.
After disposal and structural recall, each runtime's resulting deferred occupancy MUST
be no greater than its mapped root `deferred_event_capacity` when one is declared.
Lowering capacity below retained occupancy without enough explicit disposal rules fails
the whole migration unchanged; migration never evicts an entry merely to meet capacity.

Mailboxes in a retained-faulted runtime or frozen descendant remain part of the source
and use the same preservation/disposal rules, but structural recall is skipped for that
subtree because it is non-runnable. Preserved entries remain frozen and serializable
until a successful owner cleanup or root tombstone disposes them. Migration cannot
revive, retarget, or process them.
Package schema version 2 carries exactly one aggregate-state version-2 envelope and only
version-2 migration descriptors. Version mixing inside one package is invalid.

## 17. Portable execution checkpoints and hosting adapters

### 17.1 Scope and compatibility

Sections 17.2 through 17.14 define execution-checkpoint schema version 1, which wraps but
does not alter the original pure §8 `create` and `dispatch` operations. Under that
immutable version the core processes one explicitly supplied delivery, owns no queue or
database, and aggregate-state version 1 excludes queues, inboxes, outboxes, timers, and
plugin configuration. Section 17.15 separately defines queue-bearing checkpoint schema
version 2; no version-1 field or byte receives a new meaning.

A checkpoint-schema-version-1 host MUST reject every new delivery before acceptance
with host result `checkpoint_upgrade_required` whenever the selected validated bundle
contains any `deferred_events` declaration. The check allocates no delivery sequence,
creates no receipt, calls no core operation, changes no checkpoint byte, and leaves external
ownership unchanged. This host result is outside the immutable version-1 checkpoint
schema and is not an engine rejection code. The host MUST upgrade the complete
checkpoint under §17.15 before accepting such work. Version-1 processing remains
unchanged for bundles containing no `deferred_events` declaration.

An execution checkpoint is the portable durable-host state for exactly one root
ownership aggregate transaction boundary. It combines the current aggregate-state
envelope or terminal tombstone with accepted deliveries, durable operation receipts,
pending, terminal, and compact outbox work, and migration audit. A host can accept input
now and process it later without allowing the accepted envelope to exist only in
memory. A host MAY embed this contract directly in an application process; no daemon,
socket, broker, database server, background thread, or subprocess plugin protocol is
required.

Every `ExecutionStore` is one logical store scope. The host assigns it one opaque scope
identity and binds it to exactly one owning party or deployment trust domain. Mutually
untrusted parties or deployment trust domains MUST NOT share a logical store scope.
Multiple authenticated users or service principals MAY operate within one such trust
domain; principal authentication, authorization, approvals, and audit remain host
policy and do not require a separate store per principal.

Within the execution-checkpoint profile, root, creation, operation, event, and effect
identities; checkpoint revisions and digests; receipts and tombstones; and replay,
conflict, no-reuse, and deduplication guarantees are unique or evaluated only within
the selected logical store scope. Equal portable identities and bytes MAY coexist in
independent scopes. A portable identity or digest is not globally unique and is not
evidence that an artifact belongs to a host scope.

A physical backend MAY contain multiple logical store scopes only when a mandatory
external isolation key participates in every lookup, mutation, uniqueness constraint,
transaction or compare-and-swap guard, lock, replay check, receipt, tombstone, and
outbox operation. Before any such operation, the host MUST select and authorize exactly
one scope. Missing, ambiguous, mismatched, or unauthorized selection MUST fail closed
without a core call, checkpoint mutation, outbox delivery, broker acknowledgement,
fallback, probing, or access to another scope.

The scope identity, ownership binding, principal policy, and physical isolation key are
host metadata outside portable Determa State bytes and semantics. They MUST NOT be added
to a machine document, aggregate state, migration descriptor, aggregate-state package,
execution checkpoint, event, effect intent, or portable digest input. Machine
namespaces and portable identities do not select or authorize a scope. The portable
engine, bundle, checkpoint, and hash bytes remain tenant-agnostic; credentials,
endpoints, tenant identifiers, and SaaS policy fields remain host configuration.

The schemas remain unchanged because scope selection is deliberately external host
metadata. The execution-checkpoint conformance profile MUST add host-adapter cases
proving that equal portable identities coexist independently in two logical scopes;
that equal portable `effect_id` values route, retry, reconcile, and deduplicate
independently in those scopes; and that missing, ambiguous, mismatched, or unauthorized
scope selection makes no core `create`, `dispatch`, or migration call, leaves checkpoint
bytes unchanged, and performs no outbox mutation, delivery, or broker acknowledgement.
Those artifacts are follow-up work and are not added here.

The checkpoint artifact is strict UTF-8 JSON and obeys the parsing and closed-schema
rules of §16.1. Unknown formats and versions fail respectively with
`unsupported_execution_checkpoint_format` and
`unsupported_execution_checkpoint_schema_version` before semantic validation. A
recognized artifact that fails its schema or the invariants below is
`invalid_execution_checkpoint`; a valid structure with the wrong digest is
`execution_checkpoint_digest_mismatch`.

The closed checkpoint-host conflict codes are `event_id_conflict`,
`creation_id_conflict`, `operation_id_conflict`, `effect_id_conflict`, and
`checkpoint_revision_conflict`. They are deterministic non-core failures and preserve
the supplied committed checkpoint byte-for-byte.

### 17.2 Closed checkpoint artifact

The complete schema is `schema/execution-checkpoint.schema.json`. A checkpoint has
exactly:

```text
{
  execution_checkpoint_format: "determa.execution_checkpoint",
  execution_checkpoint_schema_version: 1,
  root_instance_id: non_empty_string,
  revision: canonical_decimal,
  root_record:
    { status: "retained", aggregate_state }
    | root_tombstone,
  replay_retention: permanent_or_bounded_retention,
  next_delivery_sequence: canonical_decimal,
  pending_deliveries: [pending_delivery, ...],
  next_operation_receipt_sequence: canonical_decimal,
  operation_receipts: [operation_receipt, ...],
  pending_outbox_intents: [pending_outbox_intent, ...],
  next_outbox_terminal_sequence: canonical_decimal,
  terminal_outbox_records: [terminal_outbox_record, ...],
  outbox_effect_tombstones: [outbox_effect_tombstone, ...],
  migration_audit_records: [migration_audit_record, ...],
  execution_checkpoint_digest: sha256_string
}
```

For a retained root, `root_instance_id` MUST equal the aggregate envelope's root
identity and the aggregate MUST pass all §16 validation, including definition
resolution and its own digest. A tombstone obeys §17.8.
`revision` identifies the committed checkpoint generation. The first successfully
created checkpoint has revision `"0"`. Every later transaction that changes any
checkpoint member replaces it with revision `canonical_decimal(previous + 1)` exactly
once. A read, idempotent replay, failed transaction, or compare-and-swap conflict
changes neither bytes nor revision.

The checkpoint digest is:

```text
execution_checkpoint_digest = hash([
  "determa-execution-checkpoint-digest-1",
  checkpoint_without_execution_checkpoint_digest
])
```

using §9 JCS and SHA-256. Semantic validation occurs after format/version recognition
and structural validation, verifies the embedded aggregate first, then verifies the
checkpoint digest and the remaining cross-field invariants. The pretty and canonical
normative representations are
`examples/persistence/execution-checkpoint.json` and
`examples/persistence/execution-checkpoint.canonical.json`. The canonical file is the
exact RFC 8785 byte sequence with no trailing newline.

Operational leases, locks, credentials, connection details, broker acknowledgement
tokens, wall-clock attempt timestamps, worker identities, and arbitrary application
rows are not checkpoint members. A compatible execution store MAY store those
separately and MAY include application rows, including an application response cache,
in the same native transaction.

### 17.3 Durable operation receipts and replay

An operation receipt is a durable host-layer result. It is deliberately not the
original §8 result: it contains the committed status/disposition/fault or migration
result, the resulting aggregate-state digest, and exact references to work inserted
into the pending-delivery and outbox collections. It does not contain the historical
aggregate bytes or duplicate complete emissions.

The historical core `state` is represented only by
`resulting_aggregate_state_digest`. Once a later commit replaces that aggregate, the
old state cannot be reconstructed from the receipt. An internal envelope or external
intent remains complete while pending; after an internal delivery is consumed, only
its delivery receipt and digest remain. A strict outbox retains the full intent in
either pending or terminal form (§17.6).

Applications that need to replay an HTTP body, domain projection, historical aggregate,
or other response beyond the portable receipt MUST store that application response as
application data in the same shared transaction. Its format is application-owned and
is not a checkpoint member.

Receipts allocate zero-based `receipt_sequence` values from
`next_operation_receipt_sequence` and occur in commit order. The creation receipt is
always sequence `"0"`, is always the first retained receipt, and is never pruned while
the checkpoint exists, including in bounded mode. Each emission reference is in
original core emission order and has a contiguous zero-based `emission_index`:

```text
{ kind: "internal_delivery", emission_index, event_id, delivery_sequence }
| { kind: "external_outbox", emission_index, effect_id }
```

An internal reference identifies the exact pending-delivery allocation created in the
same commit. An external reference identifies the exact full intent inserted into the
pending outbox in the same commit. Later consumption or terminalization does not alter
the producing receipt.

A delivery receipt is:

```text
{
  operation_kind: "delivery",
  receipt_sequence,
  event_id,
  request_digest,
  accepted_delivery_sequence,
  accepted_revision,
  delivery_mode,
  origin,
  committed_revision,
  resulting_aggregate_state_digest,
  outcome: { status, disposition, fault, rejection },
  emission_references
}
```

`outcome.disposition` is exactly `handled`, `unhandled`, `rejected`, or `faulted`.
Unhandled has running status, null fault/rejection, and no emissions. Rejected has no
emissions and carries the exact §8 rejection. Its fault is the aggregate root fault
exactly when status is `faulted`, as required by §8; rejected status `running` or
`completed` has null fault. Faulted disposition has the exact committed target fault
and null rejection. Parse failure, execution-store failure, transaction conflict, lost
connection, resource exhaustion before a core result, and every other infrastructure
failure produce no receipt.

On duplicate processing of a committed delivery identity:

- an equal request digest returns exactly
  `{ result: "committed", receipt: delivery_receipt }` without migration, `dispatch`,
  new work, revision change, or broker acknowledgement-side mutation;
- a different digest fails with `event_id_conflict` and preserves the checkpoint
  byte-for-byte.

This host response is the same for the first committed processing result and its
duplicates. It is not represented as a §8 result and does not imply that historical
state or complete consumed internal emissions are available.

Creation has one mandatory durable receipt:

```text
{
  operation_kind: "creation",
  receipt_sequence: "0",
  creation_id,
  request_digest,
  committed_revision: "0",
  resulting_aggregate_state_digest,
  status,
  fault,
  emission_references
}
```

The creation request digest is:

```text
hash([
  "determa-creation-request-digest-1",
  "1",
  validated_bundle_fingerprint,
  namespace,
  machine_id,
  canonical_decimal(machine_version),
  root_instance_id,
  creation_id,
  normalized_typed_binding_map
])
```

A successful `create`, including a committed faulted initialization, atomically writes
revision `"0"`, the retained aggregate, creation receipt, and all referenced pending
deliveries/outbox intents. Retrying the same root and creation id with the same digest
returns `{ result: "committed", receipt: creation_receipt }` without calling `create`.
Any different creation id or request digest for an existing root identity is
`creation_id_conflict`. The rule applies equally after terminal tombstoning; the root
identity is not recreated.

A creation rejected before an aggregate exists does not create an execution checkpoint
or reserve the root identity. Such a rejection is outside checkpoint replay. An
application requiring replay of rejected creation requests MUST commit its own request
record and response under an application-owned identity. This is safe because no
Determa aggregate, emission, or checkpoint mutation was committed.

### 17.4 Unified pending deliveries

`pending_deliveries` is one ordered durable collection for host input accepted for
later processing and for committed core internal emissions. One item is:

```text
{
  delivery_sequence,
  accepted_revision,
  delivery_mode: "input" | "internal",
  origin:
    { kind: "host_input" }
    | {
        kind: "internal_emission",
        producing_receipt_sequence,
        emission_index
      },
  envelope: portable_presented_envelope,
  envelope_digest
}
```

The portable presented envelope uses the §6.1 target and §16.2 typed-value projection.
Checkpoint acceptance validates its closed wire shape, non-empty event id, root
membership, digest, and input/internal origin consistency. Event declaration,
direction, payload defaults/types, correlation, and current target eligibility remain
the core `dispatch` decision and can produce a committed rejected receipt later.

The digest is:

```text
envelope_digest = hash([
  "determa-inbox-envelope-digest-1",
  "1",
  root_instance_id,
  delivery_mode,
  envelope
])
```

The pending record's `envelope_digest` and its eventual delivery receipt's
`request_digest` are exactly this same value.

Host input MUST have `delivery_mode: "input"` and `{ kind: "host_input" }`. A core
internal emission MUST have `delivery_mode: "internal"` and an
`internal_emission` origin. No other combination is valid.

Before accepting host input, the host returns exactly one closed result:

```text
{ result: "pending", event_id, delivery_sequence, accepted_revision }
| { result: "committed", receipt: delivery_receipt }
| {
    result: "not_accepted",
    failure: {
      code:
        "malformed_delivery"
        | "wrong_root"
        | "invalid_delivery_mode"
        | "invalid_delivery_origin"
        | "delivery_digest_mismatch"
        | "event_id_conflict"
        | "tombstoned_root"
    }
  }
```

No other pre-acceptance failure code or member is present. `malformed_delivery` means
the supplied value cannot provide the closed envelope fields and identity required to
perform acceptance. `wrong_root` means its supplied root identity does not name this
checkpoint. `invalid_delivery_mode` and `invalid_delivery_origin` cover their
respective closed unions and inconsistent pairing. `delivery_digest_mismatch` means a
caller-supplied digest does not equal the canonical digest above.
`event_id_conflict` means the event id matches a pending or retained committed
identity but its canonical digest differs. `tombstoned_root` means the identity names
this checkpoint but no equal pending or committed replay exists and its root record is
a tombstone.

The host performs only the parsing necessary to extract a candidate root identity,
event id, mode, and canonical digest, then applies this order:

1. malformed values fail `malformed_delivery`;
2. a root identity unequal to the checkpoint fails `wrong_root`;
3. when event id and digest are available, check equal/conflicting pending and retained
   receipt identities under the rules below, returning `event_id_conflict` through the
   closed `not_accepted` result when unequal;
4. if no replay applies and the root is tombstoned, fail `tombstoned_root`;
5. validate mode, origin, and any supplied digest, returning their exact failure; and
6. validate and commit acceptance.

Step 3 deliberately precedes tombstone rejection, so an equal delivery committed
before tombstoning still replays its receipt. A conflicting retained identity still
returns `event_id_conflict`. A pending delivery cannot coexist with a tombstone, but
the ordering remains normative for restored/candidate validation and future artifact
versions.

Every `not_accepted` result and every conflict creates no pending record or receipt,
does not call `dispatch`, changes no counter, revision, digest, application row, or
checkpoint byte, does not acknowledge broker ingress, and MUST NOT be reported as
accepted. Broker-owned ingress remains broker-owned.

Acceptance allocates the current `next_delivery_sequence`, increments it, appends the
full item, increments checkpoint revision once, sets `accepted_revision` to that new
revision, computes the new checkpoint digest, and commits atomically. The caller
receives exactly:

```text
{ result: "pending", event_id, delivery_sequence, accepted_revision }
```

Once that commit succeeds, the envelope is accepted by the host and exists durably in
the checkpoint. Before it succeeds, it is not accepted. A process crash cannot leave
an accepted host input only in memory.

Pending event ids are unique and are disjoint from every retained delivery-receipt
event id. When the same identity is presented:

- if pending with the same digest, return its exact pending result without mutation;
- if pending with a different digest, fail `event_id_conflict`;
- if committed with the same digest, return the exact committed delivery receipt under
  §17.3; or
- if committed with a different digest, fail `event_id_conflict`.

Every failure preserves the checkpoint byte-for-byte. Under bounded retention, an id
whose receipt was pruned is outside the declared replay horizon (§17.8).

Internal emissions allocate delivery sequences and insert complete pending items in the
same transaction as their producing operation. Their origin names that operation's
`receipt_sequence` and the emission's zero-based index. The producing receipt MUST have
at that index the exact `internal_delivery` reference with equal event id and delivery
sequence. This bidirectional link is immutable.

Processing a pending item is one atomic transition:

1. select the exact item under the host's declared queue policy;
2. invoke `dispatch` once with its recorded delivery mode and envelope;
3. remove the pending item;
4. append its delivery receipt, preserving delivery sequence, accepted revision,
   mode, origin, and digest;
5. replace the aggregate and append every newly produced pending delivery, outbox
   intent, and migration audit record;
6. increment revision once and commit.

Handled, unhandled, rejected, and faulted outcomes consume the pending item. A failure
before commit leaves it unchanged and produces no receipt. The processed event id
therefore exists in exactly one of the pending set or committed delivery-receipt set,
never both or neither after a successful processing mutation and while it remains
inside the declared replay horizon. Bounded receipt pruning may later remove the
committed identity exactly as §17.8 defines.

For delayed processing, the delivery receipt preserves the pending record's exact
`accepted_revision`; its `committed_revision` is the processing transaction's new
checkpoint revision and is strictly greater than `accepted_revision`. For foreground
accept-and-process below, both values equal the one resulting revision.

Every internal emission inserted by a delivery receipt has
`accepted_revision` equal to that producing receipt's `committed_revision`. Every
internal emission inserted by the creation receipt has `accepted_revision: "0"`.
Consequently, every pending delivery satisfies
`accepted_revision <= checkpoint.revision`; every delivery receipt satisfies
`accepted_revision <= committed_revision <= checkpoint.revision`; and every retained
receipt's `committed_revision` values are strictly increasing in receipt order.
Creation is the sole receipt at revision `"0"`. Equality between accepted and committed
revision is valid only for foreground input processing. Any other ordering, or an
internal origin whose accepted revision differs from its retained producer's committed
revision, is `invalid_execution_checkpoint`.

An embedded foreground host MAY accept and process one new host input in the same
transaction. It still allocates a delivery sequence, applies pending dedupe, and writes
the delivery receipt; the intermediate pending record need not be externally committed.
`accepted_revision` and `committed_revision` are both the single resulting revision.
Failure rolls back both acceptance and processing, so the host never reports the input
as accepted. The successful caller receives the committed host receipt, not a pending
result or reconstructed §8 result.

An external broker message is broker-owned and unaccepted by Determa until the host
commits either its pending record or its synchronous processing receipt. Broker
redelivery before that commit is not checkpoint duplication. Broker acknowledgement
occurs only after commit; loss after commit is handled by pending/receipt replay.

The checkpoint contract does not require first-in-first-out selection. A host claiming
durable first-in-first-out delivery MUST select eligible items in ascending
`delivery_sequence` across both host and internal origins. Other deterministic or
broker-directed policies MUST declare their queue profile. Sequence values are unique,
strictly increasing in the stored array, less than `next_delivery_sequence`, and never
reused. Removing an item may leave a gap and never renumbers later work.

### 17.5 Maintenance-migration operations

A migration with no delivery MUST carry a non-empty host-supplied `operation_id`.
The writer also supplies the exact checkpoint revision and digest it read, as required
by §17.9. The operation id is unique among retained native maintenance-migration
receipts for the root.
Its request digest is:

```text
hash([
  "determa-maintenance-migration-request-digest-1",
  "1",
  root_instance_id,
  operation_id,
  source_aggregate_state_digest,
  target_validated_bundle_fingerprint,
  exact_ordered_migration_descriptor_digest_route,
  maintenance_mode
])
```

A successful operation appends:

```text
{
  operation_kind: "maintenance_migration",
  receipt_sequence,
  operation_id,
  request_digest,
  committed_revision,
  source_aggregate_state_digest,
  target_validated_bundle_fingerprint,
  resulting_aggregate_state_digest,
  migration_sequences,
  result_code: "migration_applied" | "migration_no_operation"
}
```

It atomically replaces the aggregate, appends the §16 audit records, writes the
receipt at the current `next_operation_receipt_sequence`, increments that counter,
increments checkpoint revision exactly once, recomputes the checkpoint digest, and
commits. The receipt's `committed_revision` is that new revision. Its source and
resulting aggregate digests are respectively the exact pre-transaction aggregate and
committed aggregate digests. Its `target_validated_bundle_fingerprint` is the exact
target definition fingerprint used in the request digest. The receipt retains that
definition identity even after root tombstoning removes the aggregate.

For a non-empty route of length N, `result_code` is `migration_applied`, exactly N
§16 audit records are appended in descriptor-route order, and `migration_sequences`
is exactly their ordered sequence list. The values are strictly increasing and
contiguous: the first is the source aggregate's `migration_sequence + 1`, and the last
is the resulting aggregate's `migration_sequence`. A one-hop route therefore has one
audit record and one sequence; a multi-hop route has one of each per descriptor.

For an empty route, source and target definition identities are equal, the aggregate
bytes and aggregate digest are unchanged, no migration audit record is appended,
`result_code` is `migration_no_operation`, and `migration_sequences` is empty. The
checkpoint transaction still allocates its receipt and increments checkpoint revision
once so response loss cannot make the operation ambiguous. Its retained
`target_validated_bundle_fingerprint` equals the unchanged source aggregate definition
fingerprint. A loader verifies the canonical request digest from the receipt's root,
operation, source aggregate digest, target definition fingerprint, empty descriptor
route, and Boolean maintenance mode even when the root record is a tombstone. Because
the receipt does not repeat `maintenance_mode`, its digest is valid only when it equals
the canonical construction for one of the two exact Boolean values; replay still
compares the caller's complete request digest byte-for-byte.

After loading and validating the checkpoint, the host checks retained operation
identity before applying the caller's stale-writer guard. Presenting the same operation
id and digest returns exactly
`{ result: "committed", receipt: maintenance_migration_receipt }` without rerunning
migration or changing revision, even when the replay carries the revision and digest
from its original request. Reuse with a different digest is `operation_id_conflict`
and preserves the checkpoint. If no retained replay/conflict applies, a mismatched
expected revision or checkpoint digest is `checkpoint_revision_conflict`. A
deterministic migration failure produces no successful receipt or candidate aggregate;
quarantine/failure audit remains host metadata under §16.12. Retrying that failure is
safe because no migration state committed.

An implementation claiming this checkpoint profile MUST NOT expose an unkeyed
maintenance-migration commit path. Operator tools MAY generate an operation id, but
must display and reuse it when retrying after an unknown response.

### 17.6 Durable outbox lifecycle

Every committed external emission enters `pending_outbox_intents` in the same
transaction as its source operation. One entry retains the complete §9 intent:

```text
{
  intent: {
    effect_id,
    sequence,
    event,
    payload,
    correlation_id
  },
  state_revision,
  delivery_state:
    { status: "not_attempted" }
    | { status: "retryable_failure", reason_code }
    | { status: "ambiguous", reason_code }
}
```

`state_revision` is the checkpoint revision that inserted or most recently changed the
pending state. Initial insertion uses the producing operation receipt's
`committed_revision`.

`retryable_failure` means the destination did not confirm acceptance and policy permits
another attempt. `ambiguous` means acceptance may have occurred but no durable
confirmation was obtained; it MUST be retried with the same `effect_id` or resolved by
an operator/destination-specific reconciliation. Both remain pending with the full
intent. Attempt timestamps, connection errors, and credentials are operational data
outside the portable artifact; `reason_code` is a stable host-defined identifier.

A terminal transition atomically removes the pending entry and appends:

```text
{
  terminal_sequence,
  intent: complete_original_intent,
  committed_revision,
  outcome:
    { status: "confirmed" }
    | { status: "permanently_rejected", reason_code }
    | { status: "operator_cancelled", reason_code }
    | { status: "discarded", reason_code }
    | { status: "dead_lettered", reason_code }
}
```

After `not_attempted`, the seven closed attempted-delivery states have exact meanings:

- `confirmed` — the destination supplied the adapter's configured durable acceptance
  confirmation;
- `retryable_failure` — no confirmation, retry remains permitted, still pending;
- `ambiguous` — confirmation is unknown, still pending;
- `permanently_rejected` — the destination definitively refused the intent and retry
  is forbidden;
- `operator_cancelled` — an authorized operator deliberately ended delivery;
- `discarded` — declared policy deliberately ended delivery without destination
  acceptance; and
- `dead_lettered` — declared policy moved responsibility to a durable terminal
  dead-letter record.

`confirmed`, `permanently_rejected`, `operator_cancelled`, `discarded`, and
`dead_lettered` are terminal and retain the complete intent. Terminal records allocate
zero-based `terminal_sequence` values from `next_outbox_terminal_sequence` and remain
ordered by that sequence. An `effect_id` occurs exactly once across pending and
terminal full/compact outbox sets. The producing operation receipt references that
same id.

The pending-to-pending or pending-to-terminal update increments checkpoint revision
once and is atomic. `state_revision` or terminal `committed_revision` is set to that
new revision. A failed update leaves the prior state unchanged. Reuse of an existing
effect id with unequal intent is `effect_id_conflict`; equal insertion from a replayed
source operation is a no-op because the source receipt prevents redispatch.

A pending-state update request names the effect id, desired closed pending state, and
the checkpoint revision/digest read by the writer. If the desired state exactly equals
the stored state, the host returns
`{ result: "committed", record: pending_outbox_intent }` with its existing
`state_revision`; it does not apply the stale-writer check, mutate, or increment
revision. This is the response both after the first committed update and after response
loss. A genuinely different pending state requires a successful revision/digest
compare-and-swap, changes state once, sets `state_revision` to the new checkpoint
revision, and returns the same committed shape. Repeated equal
`retryable_failure`/`ambiguous` reports therefore cannot consume revisions
indefinitely.

Once terminal, the record is immutable. Repeating the same terminal outcome returns
the exact terminal record without mutation; requesting a different terminal outcome
fails `effect_id_conflict`.

The complete intent may be compacted only to:

```text
{
  terminal_sequence,
  effect_id,
  intent_digest,
  committed_revision,
  outcome
}
```

where:

```text
intent_digest = hash([
  "determa-outbox-intent-digest-1",
  "1",
  root_instance_id,
  complete_original_intent
])
```

This `outbox_effect_tombstone` preserves the exact effect identity, original terminal
sequence, terminal policy evidence, and enough immutable content evidence to replay an
equal insertion or reject unequal content. Compacting a full terminal record replaces
it atomically with the tombstone, preserves its `committed_revision`, and increments
checkpoint revision once. Retrying equal compaction returns the existing tombstone
without mutation. An effect id occurs in exactly one of pending, full terminal, or
effect-tombstone storage.

A full terminal record or compact effect tombstone may be deleted only when no retained
operation receipt references its effect id. The producing receipt must first be pruned
under the dependency-safe §17.8 rules. Otherwise silent deletion is
`invalid_execution_checkpoint`. This makes weak cleanup compatible with receipt
linkage without requiring every host to retain complete historical payloads.

Under the strict durable-outbox host profile, an intent that is not confirmed MUST
remain pending or move atomically to one retained terminal outcome. Silent deletion,
compaction to an effect tombstone, retention expiry of terminal records, and
best-effort fire-and-forget are forbidden. A host permitting any of those weaker
policies MUST declare a weaker outbox policy and MUST NOT claim the strict
durable-outbox profile. A compact-retention profile retains effect tombstones while
referencing receipts exist. A bounded cleanup profile may delete tombstones only after
those receipts are dependency-safely pruned. A policy deleting either side earlier is
not a valid checkpoint profile.

Delivery begins only after the checkpoint transaction commits. An intent remains in
pending, full terminal, or compact tombstone form until its permitted atomic lifecycle
update commits. A crash after remote acceptance but before confirmed terminalization
leaves it ambiguous/pending and may deliver it again. External delivery is therefore
at least once. It is effectively once only when the destination treats `effect_id` as
an idempotency key. Determa does not claim distributed ACID, universal external
exactly-once delivery, or proof of remote business success. Remote outcomes become
aggregate facts only through later declared input envelopes.

### 17.7 Migration audit and canonical ordering

`migration_audit_records` contains the exact successful §16.12 records in commit
order. Every record belongs to this root, records are strictly increasing by
`migration_sequence`. Permanent replay retains the complete successful history.
Bounded replay may remove audit records only in the same dependency-safe transaction
that prunes every receipt referencing them (§17.8). Failed or quarantined migration
metadata remains host-owned because it does not describe a committed aggregate
replacement.

All checkpoint arrays have one canonical semantic order:

- pending deliveries by mathematical `delivery_sequence`;
- operation receipts by mathematical `receipt_sequence`;
- pending outbox intents by mathematical intent `sequence`;
- terminal outbox records by mathematical `terminal_sequence`;
- outbox effect tombstones by mathematical `terminal_sequence`; and
- migration audit records by mathematical `migration_sequence`.

Every counter is strictly greater than each retained allocation in its domain and is
never reduced or reused. Gaps caused by consumption or bounded receipt pruning are
valid. Receipt emission indexes are contiguous from zero. Delivery event ids are
unique within pending deliveries and within retained delivery receipts, and those two
sets are disjoint. Maintenance operation ids are unique. Effect ids and terminal
sequences are unique and disjoint across pending intents, full terminal records, and
effect tombstones as applicable.

Every internal-emission origin resolves to one earlier producing receipt and one
matching internal-delivery emission reference, subject only to the bounded-pruning
rules in §17.8. Every retained delivery receipt points back to its original delivery
sequence and origin. Every external reference resolves to one pending intent, full
terminal record, or compact effect tombstone. Every retained migration receipt
retains the exact target definition fingerprint used by its canonical request digest
and, when its exact ordered audit records remain retained, the final audit target
fingerprint equals the receipt target fingerprint. No audit is required for an empty
migration or for a migration receipt whose audit is already attested as pruned. For an
empty migration, the receipt's target fingerprint remains authoritative for
request-digest validation when the aggregate is absent. Root ids,
aggregate/tombstone identity, receipt identities, targets, revisions, digests,
sequences, statuses, and union otherwise-cases MUST all be consistent.

A duplicate identity/sequence, noncanonical order, cross-set overlap, dangling or
unequal linkage, record for another root, envelope/intent digest conflict, invalid
retention transition, impossible revision/outcome union, or counter not greater than
retained allocations is `invalid_execution_checkpoint`.

### 17.8 Replay retention and root lifecycle

`replay_retention` is exactly one of:

```text
{
  mode: "permanent",
  permanent_replay_eligible: true,
  pruned_through_receipt_sequence: null,
  policy_identifier: null
}
```

or:

```text
{
  mode: "bounded",
  permanent_replay_eligible: false,
  pruned_through_receipt_sequence: positive_canonical_decimal | null,
  policy_identifier: non_empty_string
}
```

Permanent mode retains every operation receipt for the complete root-identity
lifetime. It never prunes creation, delivery, or maintenance receipts. Bounded mode
names the deployed policy and records the greatest non-creation receipt sequence
covered by completed pruning. Creation receipt sequence `"0"` remains first and is
retained unchanged for as long as the checkpoint exists, so creation retry and
conflict evidence never disappears merely because delivery history is bounded.

A non-null bounded cutoff `C` has this exact meaning:

- creation receipt `"0"` is retained;
- every non-creation receipt with sequence `1 <= sequence <= C` has been pruned;
- every retained non-creation receipt has sequence greater than `C`;
- no pending internal delivery has an origin producer in `1..C`;
- no retained committed internal-delivery receipt has an origin producer in `1..C`;
- every retained internal origin resolves transitively through retained producers back
  to creation or to a producer above `C`; and
- audit records referenced only by pruned maintenance receipts and terminal
  full/compact effect records whose producing receipts were also pruned may be removed
  in the same transaction.

Pruning may advance from the prior cutoff to candidate `C` only when removing the
entire non-creation interval through `C` satisfies all of those conditions. If a
pending or retained committed internal delivery depends directly or transitively on a
producer in that interval, the cutoff stops before the earliest required producer.
Pruning a consumer while retaining its producer is allowed: the producer's immutable
emission reference becomes historical evidence that the consumer existed, and the
cutoff attests that its committed receipt was removed. No retained origin may ever
reference a pruned producer.

The pruning transaction removes the dependency-closed interval, its safely removable
audit/effect dependants, increments revision once, and never removes pending
deliveries or pending outbox work. An equal request for the already recorded cutoff is
an idempotent read with no revision change. A lower cutoff, a skipped receipt in
`1..C`, or a cutoff crossing any retained-origin dependency is
`invalid_execution_checkpoint`. No rule requires a receipt or audit record already
attested as pruned by the cutoff.

The transition from permanent to bounded is allowed and irreversible.
`permanent_replay_eligible` becomes false in that same commit and can never become true
for this root identity, even if the current retained set later happens to contain all
new receipts. A checkpoint restored from bounded mode or with a non-null pruning cutoff
MUST remain bounded. This recorded history prevents prior cleanup from being hidden by
later configuration.

Checkpoint schema version 1 retains root identity evidence in both permanent and
bounded modes. A completed or faulted aggregate MUST remain as a retained terminal
aggregate, or may be replaced only after all pending deliveries and pending outbox
intents are resolved by a root tombstone:

```text
{
  status: "tombstone",
  root_runtime_id,
  creation_id,
  terminal_status: "completed" | "faulted",
  final_aggregate_state_digest,
  tombstone_operation_id
}
```

Tombstoning is one compare-and-swap mutation with a stable operation id, preserves
all receipts retained under the selected mode, terminal outbox records/effect
tombstones, and retained migration audit, and increments revision once. It does not
permit dispatch, migration, new pending work, or aggregate reconstruction. The first
commit and an equal retry return exactly
`{ result: "tombstoned", tombstone: root_tombstone }`; the retry does not change
revision. Another operation id fails with `operation_id_conflict`. A running aggregate
cannot be tombstoned in schema version 1.

In both retention modes, `root_instance_id` is never reused after creation, including
after completion, fault, application deletion, backup, restore, or tombstone
compaction. Physical deletion of the checkpoint or its root identity marker is
unsupported in checkpoint schema version 1. Bounded mode may prune only the
dependency-safe receipt/audit/effect history defined above; it never prunes creation
receipt `"0"`, the retained terminal aggregate/root tombstone, or root identity.

A complete backup MUST retain every checkpoint, including every bounded or permanent
terminal checkpoint/root tombstone. A restore that omits one loses root identity,
creation-conflict, and no-reuse evidence and is not a conforming restore of this
checkpoint profile. Bounded restores retain their recorded horizon and cannot be
upgraded to permanent replay. Permanent restores may advertise permanent replay only
when the collection is complete.

### 17.9 Transaction and concurrency ordering

A durable host processes one presented delivery in this exact order:

1. Select and authorize exactly one §17.1 logical store scope, then resolve, verify,
   authorize, and locally cache all required definitions, migration descriptors, route
   metadata, adapter configuration, and capability declarations.
2. Begin one transaction with exclusive ownership of the root checkpoint, or an
   observably equivalent compare-and-swap guard over its exact revision and digest.
3. Read and validate the checkpoint, pending identity, and retained operation receipts.
4. On an equal pending or committed identity, return its pending result or receipt
   without redispatch. On a digest conflict, fail without mutation.
5. If migration is requested, apply the complete §16 route to an in-memory copy.
6. Invoke `dispatch` exactly once against that candidate, or `create` exactly once for
   an absent root creation.
7. Build the candidate root record, delivery/operation receipt, pending deliveries,
   pending/terminal/tombstoned outbox records, migration audit records, counters, next
   revision, retention state, and digest.
8. Atomically commit the complete candidate plus any application rows participating
   through a shared native transaction.
9. Only after commit, acknowledge broker ingress and begin pending outbox delivery.

The transaction includes migration and dispatch even when dispatch is unhandled,
rejected, or faulted. A failure before step 8 preserves the prior checkpoint
byte-for-byte. A crash after step 8 but before ingress acknowledgement causes
redelivery to return the recorded receipt without migration, dispatch, duplicate
pending delivery, or duplicate outbox insertion.

Every writer supplies the revision and digest it read. A stale writer fails with
`checkpoint_revision_conflict`; it MUST NOT overwrite, merge, or append to the newer
checkpoint. It restarts from the committed checkpoint and re-evaluates pending or
receipt replay.
A `durable_concurrent` execution store MUST provide serializable behavior or this exact
compare-and-swap result. Lost updates are nonconformant.

### 17.10 Execution-store registration and resolution

This specification defines adapter behavior, not a language API, binary interface,
wire protocol, database schema, or cross-language dynamic-loading mechanism.
Applications SHOULD be able to inject an execution-store object directly without a
registry, URI, discovery, or command-line interface. If an implementation offers any
adapter identifier, URI, or scheme resolution, it MUST expose and use one public
registry whose registration operation associates:

- one lowercase adapter identifier and URI scheme matching
  `[a-z][a-z0-9+.-]*`;
- one factory;
- configuration validation;
- capability evaluation for the resulting configured instance;
- health operations; and
- optional adapter-storage schema migration operations.

URI parsing extracts the scheme generically and asks that registry to resolve it. Core,
host, and command-line code MUST NOT branch on a particular adapter identifier.
Registration of an already registered identifier fails with
`duplicate_adapter_registration`; later registration never overrides the first.
Resolution of an absent identifier fails with `unknown_adapter`. Invalid
configuration fails with `invalid_adapter_configuration`. Unsatisfied requested
capabilities fail with `adapter_capability_mismatch` before any root is created,
loaded, or processed.

Bundled and third-party execution stores use the same public registration operation,
validation, resolution, and error behavior. In particular, ordinary bundled identifiers
`memory`, `file`, `sqlite`, and `postgresql` receive no private switch branch,
precedence, override right, discovery path, or implicit capability elevation. An
implementation need not bundle all four. If it bundles one, that registration is
observationally indistinguishable from a third-party registration except for who
invoked the public operation. Automatic bundled inclusion is an ordinary startup call
to that same operation, not pre-population of a privileged internal map.

Automatic loading of arbitrary installed code is forbidden. Discovery is explicit or
restricted by a host allowlist because factory loading executes code. A Python host
MAY opt into package-entry-point discovery. A Rust host MAY link crates that explicitly
register trait-object factories. Both MUST preserve direct object injection. A stock
binary exposes only registrations it explicitly includes or explicitly discovers.
A language-neutral subprocess or socket protocol is future work and is not required
for embedded hosts.

### 17.11 Execution-store capabilities and composed host profiles

Execution-store capabilities describe only the configured store instance, not its
name, implementation family, ingress adapter, broker, or outbox worker. The standard
store capability names and guarantees are:

| capability | required behavior |
|---|---|
| `ephemeral` | process-loss and restart-loss are permitted; no durability claim |
| `restart_persistent` | committed bytes survive an ordinary clean restart; crash atomicity is not implied |
| `durable_single_writer` | one logical writer receives atomic, restart-safe checkpoint replacement |
| `durable_concurrent` | concurrent writers receive serializable or exact revision/digest compare-and-swap behavior |
| `shared_application_transaction` | application rows and checkpoint replacement can participate in one native atomic transaction |
| `permanent_receipt_retention` | operation receipts are physically retained for the root lifetime |
| `root_identity_retention` | every created root remains represented by a checkpoint or root tombstone and cannot be reused |
| `permanent_outbox_terminal_retention` | full terminal outbox records are physically retained |
| `compact_effect_identity_retention` | compact effect tombstones are retained while any retained receipt references them |

Hosts declare every required store capability before processing. Capability evaluation
occurs after configuration validation because durability and retention may depend on
transaction isolation, filesystem synchronization, journal mode, connection topology,
or cleanup settings. There is no implicit capability inheritance: a store advertises
every capability it proves.

`memory` MUST advertise only `ephemeral` from this standard set. A `file` adapter MAY
advertise `restart_persistent`, but MUST NOT claim either durable capability merely
because files survive normal restart. A configured `sqlite` adapter MAY advertise
`durable_single_writer` only when its transaction and synchronization behavior proves
that guarantee. A configured `postgresql` adapter MAY advertise
`durable_concurrent` and `shared_application_transaction` only when its isolation,
locking/revision checks, and transaction API prove them. No identifier alone proves a
capability.

Host profiles describe a composition of an execution store with ingress handling,
queue policy, an outbox worker, destination semantics, and application transactions.
They are not execution-store capabilities. Every durable checkpoint host profile below
requires `root_identity_retention`; physical checkpoint/root-marker deletion is not a
weaker schema-version-1 profile:

- `durable_embedded_processing` requires a durable store and the §17.4 atomic
  accept/process rules; no broker is required.
- `exactly_once_committed_processing` additionally requires permanent checkpoint
  retention mode and `permanent_receipt_retention`.
- `broker_integrated` requires a durable store, an ingress adapter that acknowledges
  only after checkpoint commit, durable redelivery behavior, and an outbox worker; a
  store alone can never claim it.
- `strict_durable_outbox` requires the §17.6 total lifecycle,
  `permanent_outbox_terminal_retention`, and an outbox worker that never silently
  deletes unresolved work.
- `compact_durable_outbox` permits full terminal intents to become §17.6 effect
  tombstones, requires `compact_effect_identity_retention`, and forbids deleting a
  tombstone while a retained receipt references it.
- `shared_application_transaction` is available to a host only when the store exposes
  that store capability and the application actually uses one native transaction.

A bounded checkpoint cannot satisfy `exactly_once_committed_processing`. Selecting
`memory` for any durable host profile fails before processing rather than silently
weakening the requested profile. A host MUST validate the complete composed profile,
not infer it from the storage scheme.

### 17.12 Exact guarantee boundary

For every identity still covered by the checkpoint's replay-retention guarantee, the
checkpoint contract provides exactly-once **committed processing** within one selected
§17.1 logical store scope:

- at most one aggregate replacement is committed for that identity;
- every retry with equal content returns the first durable host receipt;
- pending deliveries, outbox records, migration audit, application rows included in a
  shared transaction, and revision change commit with that receipt; and
- an uncommitted attempt has no durable effect.

This guarantee does not mean exactly-once network receipt, broker delivery, remote side
effect, or globally distributed transaction. Broker delivery may repeat before
acknowledgement. External effects are at least once and require destination
idempotency for effectively-once behavior. Hosts MUST state their selected durability,
retention, queue-ordering, broker, and external-idempotency profiles without attributing
stronger guarantees to the pure core.

Every store operation and every outbox routing, retry, reconciliation, and idempotency
record MUST retain the selected logical store scope. External effect idempotency MUST
use the host-owned scope identity together with the portable `effect_id`; `effect_id`
alone is not globally unique. This host metadata MUST NOT alter the portable intent.

### 17.13 Cluster checkpoint composition

One execution checkpoint never represents a complete deployment or an independently
committed owned child: every owned child remains inside its root aggregate. A complete
deployment backup is a consistent collection containing:

- one valid checkpoint for every created root identity, with either a retained
  aggregate or root tombstone in its `root_record`;
- every content-addressed normalized definition referenced by those aggregates and
  fault anchors;
- every trusted migration descriptor and route needed by deployment recovery policy;
- adapter metadata needed to restore pending broker ownership without treating an
  uncommitted message as accepted;
- relevant application data; and
- a manifest that identifies the exact member bytes and the consistency point chosen
  by the host.

The checkpoint schema does not define that cluster manifest or require a global
transaction across unrelated roots. A cluster backup is valid only if the storage and
broker-specific procedure supplies an application-appropriate consistency point and
does not omit committed checkpoints/tombstones, operation receipts, pending deliveries,
pending/terminal/tombstoned outbox records, retention history, application response
data needed by its API, or referenced trusted artifacts. A restore that omits any
created root's checkpoint/tombstone is nonconforming in either retention mode. A
restored deployment may claim permanent replay only when the collection is complete
and every checkpoint remains permanently eligible. Restoring one checkpoint requires
no other root checkpoint, but application-level cross-root invariants may require
coordinated backup and restore.

This section defines only checkpoint-collection completeness. Backup and restore
operation protocols, cloning, transfer or rebinding, multi-scope archives, relocation,
and fencing are not defined by this profile. Portable checkpoint bytes, identities, or
digests MUST NOT authorize access to or movement across logical store scopes. A future
host-profile contract is required before any such operation can claim conformance.

### 17.14 Future timer durability

Format 1 introduces no timer semantics and checkpoint schema version 1 has no timer
member. A host therefore MUST NOT advertise accepted timer work as covered by a durable
checkpoint while retaining that work only in process memory. A future timer contract
that participates in durable processing MUST add a versioned checkpoint representation
for accepted scheduling requests, deadlines, cancellation state, and deterministic
delivery identity, or use an external durable service whose accepted ownership and
recovery boundary is stated explicitly.

Adding non-empty timer state to this artifact requires a later checkpoint schema
version or a separately identified durable timer artifact. It does not silently add a
field to schema version 1 and does not change aggregate-state schema version 1.

### 17.15 Queue-bearing checkpoint version 2

Execution-checkpoint schema version 2 is the single durable continuation boundary for
an aggregate-state version-2 envelope. Its closed schema is
`schema/execution-checkpoint-v2.schema.json`. It preserves version-1 root, replay,
outbox, audit, and revision concepts, but removes `next_delivery_sequence` and
`pending_deliveries`: accepted envelopes already exist exactly once in the embedded
aggregate's runtime-local mailboxes. Reconstructing a second host-pending copy is
`invalid_execution_checkpoint`.

The version-2 digest is:

```text
execution_checkpoint_digest = hash([
  "determa-execution-checkpoint-digest-2",
  checkpoint_without_execution_checkpoint_digest
])
```

A fresh version-2 checkpoint is created only from a successful `create_v2` result. Its
checkpoint `revision` and creation receipt `committed_revision` are `"0"`, and the
creation receipt sequence is `"0"`. `next_operation_receipt_sequence` begins at `"1"`;
creation-time lifecycle terminal receipts allocate from it in their defined order.
`event_identity_tombstones` is empty. Outbox collections are empty unless initialization
emitted external intents. The embedded aggregate carries the exact post-initialization
counters from §8: acceptance and queue counters began at zero, initialization emissions
allocated them in order, and every retained emitted entry has `deferral_count: "0"`.
No other creation path, inferred upgrade, or default counter state is conforming.

Admission is one atomic checkpoint mutation. A host first normalizes the complete
ordered batch, including exact source, cause, target, payload, correlation, and the
version-2 envelope digest from §16.3. It then applies this closed order before allocating
anything:

1. malformed batch or member: `malformed_delivery`;
2. a root unequal to this checkpoint: `wrong_root`;
3. the same `event_id` occurring more than once in this batch, whether equal or
   conflicting: `duplicate_event_id_in_batch` for the complete batch;
4. compare each event identity with every retained ready/deferred entry, acceptance
   receipt, terminal receipt, and event-identity tombstone as defined below;
5. if any non-replay member targets a completed or faulted root:
   `terminal_root`; if the root is a tombstone: `tombstoned_root`;
6. invalid mode, source, target, correlation, payload, or supplied digest: its existing
   exact validation code; otherwise accept the complete batch.

The closed version-2 admission failure codes are exactly `malformed_delivery`,
`wrong_root`, `duplicate_event_id_in_batch`, `event_id_conflict`, `terminal_root`,
`tombstoned_root`, `invalid_delivery_mode`, `invalid_delivery_source`,
`invalid_instance_target`, `inactive_component_target`, `invalid_event`,
`invalid_payload`, `invalid_correlation`, and `delivery_digest_mismatch`. No other
failure code is conforming. An all-replay batch
returns its retained evidence without mutation. A mixed replay/new batch allocates only
the new members, commits once, and returns evidence in caller order; any failure in any
member rejects the whole batch.

Every failure rejects the complete batch, preserves every byte and counter, calls no
core step, and leaves external ownership unchanged. For success, the host allocates one
immutable `acceptance_sequence` and initial `queue_sequence` per entry in caller order,
appends each to its exact target runtime ready tail, appends one `acceptance` receipt per
host event, increments checkpoint revision once, recomputes the aggregate and checkpoint
digests, and commits. Only after commit may a broker adapter acknowledge transfer of
ownership.

The acceptance receipt is durable proof of admission, not proof of processing. It names
`event_id`, version-2 request digest, acceptance sequence, accepted revision, and
delivery mode. An existing identity is equal only when its retained digest in the same
digest domain equals the candidate digest. Equal replay returns the original acceptance
and, when present, terminal evidence without mutation. A conflicting digest returns
`event_id_conflict`, including after processing, root completion, root fault, or root
tombstoning. These replay/conflict checks precede terminal-root rejection. Internal
emissions append directly to target ready mailboxes in their producing RTC commit and
are referenced by the producing receipt; they never pass through a second
`pending_deliveries` collection.

Processing requires an explicit target runtime id. It selects only that runtime's ready
head and performs one §6 RTC/classification step. A `deferred` result commits the
ready-to-deferred move and revision but creates no terminal receipt. A later recall move
also creates no receipt. A handled, unhandled, faulted, or lifecycle-disposed event is
removed from its final mailbox and receives exactly one `event_terminal` receipt with
its event id, request digest, acceptance sequence, final queue sequence, committed
revision, resulting aggregate digest, terminal outcome, and emission references. The
acceptance receipt may coexist with its terminal receipt because they attest different
facts; the full envelope never coexists in two lifecycle locations.

Terminal event outcomes are exactly `handled`, `unhandled`, `faulted`, `disposed`, and
`migration_disposed`. Lifecycle `disposed` has exactly one reason:
`runtime_cancelled`, `runtime_completed`, `aggregate_completed`, or `root_tombstoned`.
`migration_disposed` instead carries the arbitrary non-empty operator reason and exact
version-2 migration-descriptor digest defined by §16.15; the two shapes are disjoint.
A successful lifecycle operation creates disposal receipts in runtime cleanup order and,
within each runtime, ready entries followed by deferred entries, each in queue order.
Receipt sequences are allocated in that order and may share one committed revision;
`(committed_revision, receipt_sequence)` is strictly increasing. A root or contained
fault freezes noncausal mailbox entries as §6.7 states; frozen deferred entries are not
recalled, processed, or capacity-evicted. They remain engine-owned until successful
owner cleanup or root tombstoning creates terminal receipts. A cleanup fault rolls back
the complete cleanup and every tentative disposal receipt; it cannot report successful
disposal.

The core-only `step` operation returns every successful cleanup removal in its required
`lifecycle_dispositions` list; checkpoint receipts are not a core precondition. The
checkpoint host translates one successful core result atomically as follows: create the
selected causal event's terminal receipt first when it is terminal, then create one
terminal `disposed` receipt for each lifecycle disposition in list order. An
`internal_disposed` core emission names exactly one list index and becomes an
`internal_terminal` emission reference naming the resulting terminal receipt; an
`internal_mailbox` reference continues to name its sole retained mailbox entry. Missing,
duplicate, mismatched, or out-of-range references are `invalid_execution_checkpoint`.
All receipts, state, outbox intents, revision, counters, and digests commit together or
none do. Root tombstoning is a host lifecycle operation and uses the same receipt rules,
but is not returned by a core `step`.

The lifecycle of one accepted event is closed:

| current location | operation | committed next location |
|---|---|---|
| external/unaccepted | rejected admission or pre-commit crash | external/unaccepted |
| external/unaccepted | committed admission | one target ready mailbox plus acceptance receipt |
| ready | enabled handler succeeds | terminal `handled` receipt |
| ready | no enabled handler, active deferral, capacity available | same runtime deferred mailbox |
| ready | no enabled handler, no active deferral | terminal `unhandled` receipt |
| ready | guard/action/invariant or capacity-overflow fault | terminal `faulted` receipt; runtime fault rule applies |
| deferred | structural walk reaches a handler declaration before a deferral-only level, or reaches root with neither | same runtime ready tail |
| deferred | structural walk reaches a deferral-only level before any handler declaration | same deferred position |
| ready or deferred | successful owner disposal | terminal `disposed` receipt |
| ready or deferred | explicit migration disposal rule | terminal `migration_disposed` receipt |
| any engine-owned location | transaction failure before commit | exact prior location and bytes |

An event identity exists in exactly one live location (one ready/deferred mailbox entry)
or terminal location (one terminal receipt or one event-identity tombstone), while its
acceptance receipt may coexist as admission evidence. Permanent replay retains all
receipts. Bounded replay may replace an acceptance-plus-terminal receipt unit with one
`event_identity_tombstone` containing event id, request digest, its exact digest domain,
acceptance sequence, terminal receipt sequence, and terminal disposition. Tombstones are
retained for the lifetime of the root record, including after root tombstoning; therefore
equal replay and conflict detection remain exact after compaction.

Pruning is dependency-closed. An acceptance or producing receipt cannot be removed while
its event is in a mailbox. An acceptance receipt and its terminal receipt are pruned in
one transaction that creates their tombstone. A receipt containing an internal emission
reference cannot be pruned until the referenced event is terminal and represented by a
retained terminal receipt or tombstone. An `internal_terminal` reference resolves to
either its exact terminal receipt sequence or the tombstone retaining that sequence.
External outbox references continue to obey §17.6. No receipt, mailbox entry, terminal
record, producer reference, or tombstone may be deleted while doing so would leave a
dangling reference or lose replay/conflict evidence. Violation is
`invalid_execution_checkpoint`.

Version-2 operation receipts remain ordered by mathematical `receipt_sequence`; event
identity tombstones are ordered by mathematical `terminal_receipt_sequence`. Event ids,
acceptance sequences, and terminal receipt sequences are unique in their respective
domains. Mailbox event ids are disjoint from terminal-receipt and event-tombstone ids;
terminal-receipt and event-tombstone ids are disjoint from each other; an acceptance
receipt may overlap its live mailbox or terminal receipt but not a tombstone that replaced
it. Every retained allocation is below its next counter. Noncanonical order, overlap,
duplicate allocation, dangling dependency, or counter regression is
`invalid_execution_checkpoint`.

A checkpoint containing only deferred entries is pending but not runnable. Empty ready
mailboxes do not authorize polling, timers, or spontaneous execution; later accepted
input or explicit host activity may change configuration and cause recall. A host can
persist configuration and mailbox rows separately only when its transaction or
compare-and-swap operation commits an observably equivalent single checkpoint revision.

Definition migration of a version-2 checkpoint is one maintenance transaction over the
aggregate, every mailbox, generated terminal disposal receipt, audit, outbox, and
revision. It uses §16.15 version-2 descriptors. A queued event whose target incarnation
is deleted or replaced, whose event is removed, or whose normalized payload/correlation
contract is incompatible cannot be guessed, coerced, or silently discarded. It must be
covered by a valid explicit disposal rule or the complete migration fails unchanged.
Backup, restore, cloning, relocation, and authority fencing remain outside this contract
and are reserved to issue #70.

Version-1 to version-2 checkpoint upgrade is explicit and atomic. It first performs the
exact aggregate `1 -> 2` upgrade from §16.15. It wraps the version-1 creation receipt as
`legacy_v1_creation` and every version-1 delivery or maintenance receipt as
`legacy_v1_operation`, preserving each complete version-1 receipt byte value inside
`legacy_receipt`. Wrapper receipt sequences retain the old mathematical order. The first
new native acceptance receipt allocates the source checkpoint's existing
`next_operation_receipt_sequence`. This
makes legacy `internal_delivery` origin and emission-reference links valid historical
evidence without pretending they are native version-2 links.

Every retained version-1 delivery identity, terminal or pending, receives the v2
acceptance identity equal to its original mathematical `delivery_sequence` (called
`accepted_delivery_sequence` in a terminal receipt). The upgraded aggregate's
`next_acceptance_sequence` is the source checkpoint's `next_delivery_sequence`. Thus
legacy terminal evidence can later produce an event tombstone with an exact acceptance
sequence, and gaps from valid old pruning remain gaps rather than being renumbered.

The upgrader then converts each version-1 `pending_delivery` in ascending delivery
sequence. It preserves event id, mode, payload, target, correlation, accepted revision,
and the complete old delivery evidence. Host origin becomes version-2 host source.
Internal origin becomes `{legacy_v1_internal: {producing_receipt_sequence,
emission_index}}`; in both cases `cause_id` is the event id. The resulting envelope is a
new version-2 value, so its digest is recomputed in the
`determa-inbox-envelope-digest-2` domain and MUST NOT equal or claim to preserve the old
version-1 digest. Each converted pending entry uses its original delivery sequence as
its acceptance sequence; the host allocates only queue sequences from zero in pending
delivery order. It sets every converted entry's `deferral_count` to `"0"`, appends it
to its exact ready mailbox, and creates a native acceptance receipt whose optional
`legacy_v1_delivery` field preserves old delivery sequence, digest, and origin.
That evidence field is required when the converted delivery mode is `internal` and is
the only conforming reason a version-2 acceptance receipt has internal mode; native
version-2 runtime sends are represented by producing emission references instead.

Old creation, request, result, envelope, and aggregate digests remain version-1
historical evidence inside wrappers; no version-1 digest is compared with a version-2
digest. A compact tombstone derived from legacy terminal evidence records digest domain
`determa-inbox-envelope-digest-1`; a native version-2 event tombstone records
`determa-inbox-envelope-digest-2`. Equal replay of legacy terminal work therefore uses
the original version-1 request digest, while newly converted pending work uses its
native version-2 acceptance digest. Ambiguous or missing legacy replay evidence is
`invalid_execution_checkpoint`, never guessed.
For a legacy terminal tombstone, `acceptance_sequence` is the nested receipt's
`accepted_delivery_sequence` and `terminal_receipt_sequence` is its enclosing
`legacy_v1_operation.receipt_sequence`.
Its terminal disposition preserves the nested version-1 disposition; `rejected` is
permitted only with the version-1 digest domain because native version-2 admission
rejects invalid work before acceptance.

The upgrader initializes `event_identity_tombstones` empty unless it performs the
dependency-closed legacy compaction just described, removes `pending_deliveries` and
renames the source `next_delivery_sequence` value to aggregate
`next_acceptance_sequence`, increments checkpoint revision once, sets checkpoint schema
version 2, advances `next_operation_receipt_sequence` once per converted pending entry,
and recomputes the new aggregate and checkpoint digests. Changed envelopes and digest
domains mean old and new
outer digests normally differ. Any stale target, invalid legacy link, duplicate identity,
or conversion that cannot satisfy the version-2 schema fails the whole upgrade with the
version-1 checkpoint byte-for-byte unchanged. Version 2 has no general downgrade to
checkpoint version 1; downgrade is permitted only when both mailbox counters are zero,
all mailboxes and event tombstones are empty, and no version-2-only or wrapped legacy
receipt exists.
