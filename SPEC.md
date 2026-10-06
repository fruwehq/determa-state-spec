# Determa State — specification

Status: **pre-alpha, unreleased**. Normative unless a section says informative. Document format:
**1**. Spec version: **0.3.0** (see `VERSION` ; synchronized across the Determa State repositories).
Keywords MUST / SHOULD / MAY are interpreted as in RFC 2119.

### Reading and authoring model

Start with an ordinary YAML machine in `examples/` . Authors declare states, typed variables,
events, CEL guards and ordered actions. They do not write hashes, dependency closures, capability
reports, JSON Pointers or protocol records into transitions. CEL and structured actions are the
recommended portable defaults. Named native providers and custom languages are also first-class
options (§5.4).

There are four distinct boundaries:

1.  **Authored source:** human-readable YAML, validated by
   `schema/machine.schema.json` ; numeric `format: 1` is the only current format.
2.  **Deployment configuration:** selected provider implementations, installation
   policy, named endpoints/scopes and credentials; never machine syntax.
3.  **Generated executable definition:** explicit resolution/compilation pins the
   installed closure in a generated lock and resolved definition before execution.
4.  **Generated runtime artifacts:** the engine/host creates typed state, identities,
   checkpoints, receipts and archives. Their exact portable encoding is JSON.

Users never hand-author `validated_bundle_fingerprint` . The engine/tool computes this SHA-256
content identity from the normalized resolved executable definition. It is not a UUID. Generated
identity material proves content equality, not trust, live authorization, commit fate or permission
to execute.

YAML is accepted for source readability under §2's strict value rules. Generated portable wire
artifacts use §16.2 typed JSON and §9 canonical JSON bytes so that implementations agree on exact
values and hashes. YAML source fixtures do not become alternate wire encodings of a checkpoint or
archive.

The resolver generates provider locks and executable definitions. The engine generates definition
fingerprints and runtime identities. Hosts generate admission, disposition and persistence records.
Caller-supplied event or correlation identifiers may be meaningful application strings; they do not
require a UUID encoding. Generated digests and caller identifiers have different responsibilities.

This pre-alpha redesign replaces superseded draft syntax directly. There is no compatibility parser,
alias, second authored format or migration path for that draft. §16 migration is between validated
current definitions, not draft-format readers.

## 0. References and semantic independence

-  David Harel, *Statecharts: A Visual Formalism for Complex Systems*.
-  OMG UML State Machines, for terminology and established statechart concepts.
-  Miro Samek, *Practical UML Statecharts in C/C++*, for implementation lessons and
  event-driven design patterns.
-  CEL — Common Expression Language (<https://cel.dev/>), used for guards and computed
  action values.

Determa State has Harel/UML lineage, but this document defines Determa semantics. A similar name or
diagram shape does not import behavior from another framework, runtime, or notation. Where
established statechart dialects differ, this document makes an explicit choice and the conformance
suite pins that choice.

## 1. Purpose and core boundary

Determa State defines portable, deterministic statechart behavior:

1.  typed immutable input envelopes;
2.  hierarchical state configuration and transition selection;
3.  run-to-completion processing of one envelope;
4.  typed state-scoped variables and pure CEL expressions;
5.  structured actions that update logical state or emit immutable intents;
6.  isolated reusable components and owned spawned runtimes; and
7.  deterministic identities, lifecycle, rollback, and fault results; and
8.  runtime-local ready and deferred mailboxes for lossless continuation.

The core deliberately does **not** provide:

-  an external transport queue, delivery worker, scheduler, or background thread;
-  dead-letter storage or a dead-letter policy;
-  a clock, timer, delay, sleep, or time-event implementation;
-  transport, broker, retry, acknowledgement, or delivery guarantees;
-  a state store, transaction manager, credentials, or external I/O; or
-  plugin discovery, installation, configuration schemas, or package resolution.

Those are host or plugin responsibilities (§11). The core can admit an envelope to an isolated
runtime mailbox or receive one envelope for immediate foreground processing, processes at most one
targeted RTC step at a time, and returns state plus ordered emissions. It never calls an external
queue, timer, broker, database, or remote service from a guard or action.

This boundary permits an in-memory foreground host, a database-backed request/response host, a
durable worker, or a distributed broker without changing statechart semantics. End-to-end behavior
remains conditional on the delivery trace and guarantees of the selected plugins.

## 2. Conformance, parsing, and format identity

An implementation is conformant iff it passes every applicable case in
[`fruwehq/determa-state-conformance`](https://github.com/fruwehq/determa-state-conformance) . The
conformance suite is the executable arbiter. If prose and the suite disagree, a bug MUST be filed
and resolved; implementations MUST NOT choose their preferred result.

Authored machines MUST be parsed with the YAML 1.2 core schema and validated against
`schema/machine.schema.json` before semantic validation. Generated executable definitions MUST use
`schema/resolved-machine-v1.schema.json`. The caller selects the stage explicitly (§5.4); loaders
MUST NOT retry the other schema after rejection. Both entrypoints share
`schema/machine-grammar-v1.schema.json` and the parsing/value restrictions below.

The accepted parsed value model is deliberately narrower than general YAML:

-  a source contains exactly one document;
-  every mapping key is a string and occurs exactly once;
-  duplicate JSON object names and duplicate YAML mapping keys are rejected before
  schema validation;
-  YAML anchors, aliases, merge keys, and explicit tags are unsupported;
-  the expanded value is an acyclic JSON-compatible tree of maps, lists, strings,
  Booleans, nulls, and numeric values; and
-  every numeric leaf satisfies §5.2, including leaves nested inside `map` , `list` , or
  `meta` .

Loaders MUST detect source-level duplicates before constructing an ordinary host map; “last value
wins” and “first value wins” are nonconformant. The exact pre-schema load codes are `duplicate_key`
, `non_string_map_key` , `unsupported_yaml_feature` , `non_json_value` , `invalid_unicode` ,
`invalid_numeric_syntax` , `invalid_boolean_syntax` , `invalid_null_syntax` , and
`numeric_value_out_of_range` . A later JSON Schema or semantic failure uses that layer's code
instead.

Every string in a bundle, binding, envelope, or logical state MUST be a sequence of Unicode scalar
values. Unpaired UTF-16 surrogates are `invalid_unicode` , including when introduced through an
escaped JSON/YAML string. Code points are preserved exactly: the core performs no NFC, NFD, case, or
newline normalization. String equality compares code-point sequences. Every specified byte ordering
and every hash encodes those code points as strict UTF-8.

For a machine source, invalid Unicode is the pre-schema `invalid_unicode` load error. Programmatic
values crossing `create` or `step` are checked recursively before CEL, state mutation, or identity
hashing:

-  invalid `root_instance_id` or `creation_id` rejects creation with
  `invalid_creation_request` ;
-  an invalid creation-binding map key or nested string value rejects creation with
  `invalid_binding` ;
-  an invalid envelope event name or `event_id` rejects with `invalid_event` ;
-  an invalid `correlation_id` rejects with `invalid_correlation` ;
-  an invalid target identity/reference string rejects with `invalid_instance_target` ;
-  an invalid payload key or nested string value, including reserved `env.changed` ,
  rejects with `invalid_payload` ; and
-  an invalid string anywhere in supplied prior state rejects with
  `invalid_prior_state` .

These are pre-step rejections with no counter, state, fault, or emission change. Author CEL literals
are bundle source and must also pass CEL parsing; every runtime CEL string value is restricted to
the same scalar-value domain.

Portable YAML numeric scalars use exactly this JSON number grammar:

```text
-?(0|[1-9][0-9]*)(\.[0-9]+)?([eE][+-]?[0-9]+)?
```

The `+` metacharacters above denote repetition in grammar notation; a leading plus sign is not an
accepted numeric sign. Hexadecimal, octal, underscores, leading `+` , leading/trailing decimal
points, `.inf` , and `.nan` are `invalid_numeric_syntax` , even when a YAML library would assign
them a numeric core tag. A quoted token is a string. An accepted token without a fraction/exponent
is an `int` ; a token with either is a `double` . Thus `-0` is integer zero, `1e0` is double `1.0` ,
and `-0.0` normalizes to positive double zero under §5.2.

Number, Boolean, and null resolution is one pre-schema pass over the source spelling of every plain
scalar token, including an empty mapping or sequence value position. This pass MUST occur before
library tag coercion can erase the original spelling. The portable resolution is exact:

-  plain `true` and `false` are Booleans;
-  plain `True` , `TRUE` , `False` , and `FALSE` are `invalid_boolean_syntax` ;
-  plain `null` is null;
-  plain `Null` , `NULL` , `~` , and an empty scalar are `invalid_null_syntax` ;
-  plain `yes` , `no` , `on` , `off` , `y` , and `n` , including their ASCII case variants,
  are strings and remain valid identifiers where an identifier is allowed;
-  plain `1` and `0` are integers under the numeric grammar above; and
-  every quoted form is a string, including quoted Boolean-like, null-like, and numeric
  tokens.

A loader MUST apply these source-token rules directly rather than accepting its YAML library's
implicit scalar tags and attempting to reconstruct spelling afterward.

> **Non-normative authoring note:** Quote YAML-1.1 Boolean-like identifiers such as > `yes` , `no` ,
`on` , and `off` when a bundle may pass through YAML 1.1 tooling. This > improves interoperability
with nonconforming intermediary tools; it is not an > additional §5.1 identifier restriction.

Every document MUST carry the YAML/JSON integer:

```yaml
format: 1
```

`format` is required. Omission, a string value, or any number other than `1` is `unsupported_format`
. One document is one self-contained bundle; embedded or mixed formats are invalid.

Format 1 is still pre-release and has no compatibility commitment. Earlier draft documents, schemas,
snapshots, and implementation behavior MAY become invalid while the format is being designed. There
are no legacy aliases, implicit conversions, or dual parsers in this alpha. Compatibility rules
begin only when a format is explicitly published as stable.

Determa State repository/package releases 0.0.1 through 0.0.6 emitted a legacy machine grammar and
snapshot format. Those artifacts are frozen legacy artifacts and MUST NOT be migrated into format 1.
Release 0.0.7 introduced format 1; format-1 definitions produced by 0.0.7 remain ordinary format-1
inputs and are accepted or rejected by the current §2 source checks, schema, and §5 semantic
validation. These historical release numbers describe the support policy only. Repository/package
SemVer remains distinct from the machine `format` discriminator and is never inferred from document
shape.

A machine loader applies the format check above without release-provenance or shape detection. A
definition produced by releases 0.0.1 through 0.0.6 with a missing discriminator or a discriminator
other than the YAML/JSON integer `1` fails `unsupported_format` . If a legacy-shaped document
explicitly carries `format: 1` , the discriminator succeeds and the loader proceeds to the current
schema; the legacy structure MUST be rejected by JSON Schema (`structural_validation` in §5.1
conformance-harness notation) before §5 semantic validation. The loader MUST NOT identify a legacy
document by its other fields, supply omitted fields, select a legacy parser, or silently reinterpret
it as format 1.

There is no 0.0.1-through-0.0.6 definition converter or legacy-snapshot import in this
specification. Users migrating from those releases MUST author a new format-1 bundle and MUST NOT
carry a legacy snapshot forward. The definition migration rules in §16 apply only between recognized
format-1 bundles and their format-1 portable artifacts.

The document `format` is independent of:

-  repository/package SemVer (`VERSION`);
-  an author's machine `version` ; and
-  independently versioned hosts, plugins, and the umbrella launcher.

Identifiers MUST match `^[A-Za-z_][A-Za-z0-9_]*$` . Namespaces are dot-separated identifiers. Public
fields use unabbreviated names except for the deliberately retained keywords `init` , `lang` ,
`meta` , `on_events` , and `transition_to` .

## 3. Vocabulary

-  **Bundle** — one document containing shared event contracts and one or more machine
  definitions.
-  **Machine definition** — a named statechart inside a bundle, identified by
  `(namespace, machine_id, version)` .
-  **Runtime** — one logical execution of a machine definition with configuration,
  variables, lifecycle status, and identity.
-  **Root ownership aggregate** — one root runtime together with every retained
  lifecycle-bound component and owned spawned descendant.
-  **State configuration** — the active leaf plus all active ancestors in one runtime.
-  **Variable** — typed extended state declared on a state and scoped to that state and
  its descendants.
-  **Envelope** — one immutable occurrence of an event, with identity, target, payload,
  and optional correlation.
-  **Run-to-completion step** — atomic processing of one envelope for one target runtime
  from one stable aggregate state to the next.
-  **Emission** — an immutable internal envelope or external output intent returned by
  the core after a successful step.
-  **Queue plugin** — host infrastructure that chooses which envelope to present next
  and what to do with handled, unhandled, rejected, or faulting deliveries.
-  **External peer** — anything outside the aggregate that communicates only through
  declared public events.

The core recognizes exactly four statechart relationships:

1.  **Inline/nested state** — one hierarchy, configuration, and variable scope.
2.  **Lifecycle-bound component** — a statically placed isolated runtime created and
   disposed with a containing `parallel` state.
3.  **Owned spawned instance** — a dynamically created isolated runtime owned by the
   root aggregate.
4.  **Independent external peer** — outside engine state and lifecycle.

No relationship permits direct cross-runtime variable access or a transition outside the executing
runtime's state boundary.

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

See `examples/minimal.yaml` and `vectors/full.yaml` .

### 4.2 Top-level fields

-  `format` — required integer `1` .
-  `namespace` — required dotted identifier.
-  `events` — optional shared event declarations.
-  `machines` — required non-empty ordered list of machine definitions.
-  `meta` — optional opaque annotations ignored by core execution.

### 4.3 Machine fields

-  `machine_id` — required and unique within the bundle.
-  `version` — positive signed-64-bit integer, optional, default `1` .
-  `languages` — optional `{guard, action}` language identifiers.
-  `events` — optional private event declarations; every machine-local event is
  `internal` .
-  `root` — required outermost state node.
-  `meta` — optional opaque annotations.

All machine references resolve inside the same bundle. Package imports, dependency constraints,
visibility across packages, and version resolution are unsupported.

In format 1, `languages.guard` is exactly `cel` and `languages.action` is exactly `determa` ; those
are also the defaults when omitted. A transition `lang` , when present, is exactly `cel` . These
defaults govern CEL strings and structured actions. An exact runtime-provider slot (§5.4) may
coexist in the same machine; it does not change the machine-wide language identifiers or make
arbitrary language names valid here.

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

`direction` is `internal` , `input` , or `output` and defaults to `internal` .

-  Bundle `input` events may cross from a host into a root or spawned runtime.
-  Bundle `output` events may leave the ownership aggregate as output intents.
-  Bundle `internal` events are shared contracts but cannot cross the host boundary.
-  Machine-local events are private and MUST be `internal` .

Payload fields use `string` , `int` , `float` , `bool` , `map` , or `list` . A field with
`required: true` must be present and cannot also declare `default` . An optional field may declare a
correctly typed literal `default` ; an optional field without a default remains absent when omitted.
Extra fields are invalid.

Payload validation materializes defaults into the normalized immutable envelope before any guard,
handler action, capture, or delivery can observe it. For a host envelope, normalization occurs after
structural/type validation and before handler search. For an author `send` , payload expressions are
evaluated first and defaults are then materialized before the immutable emission and its target are
recorded. A declared default is type-checked at bundle load. Rejected host input remains the
caller's unchanged original value.

Numeric payload values and defaults use the single portable normalization rule in §5.2. In
particular, a `float` field deliberately accepts either an integer or floating-point numeric value
and exposes one normalized CEL `double` .

`correlates_to` is valid only on a bundle `input` event and MUST name a bundle `output` event.
Success, rejection, failure, cancellation, or no-response outcomes from an external effect become
machine behavior only through later declared input events.

The names `env` , `done` , `determa.component_completed` , `determa.component_failed` , and
`determa.spawned_instance_failed` are reserved and cannot be author declarations.

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

The `env.changed` record is specialized per target machine: every root external variable is a
declared optional field and no other field exists. Runtime validation requires at least one field.
The `done.relationship` value is exactly `parallel` or `spawned_instance` . A parallel payload
materializes exactly `relationship` , `state_path` , and `owner_runtime_id` ; the spawned-instance
payload materializes exactly `relationship` , `instance` , `instance_id` , `machine_id` , and
`machine_version` . All other optional fields remain absent. This single typed record lets a guard
select the relationship before an action reads a relationship-specific field.

All fields without `?` are required and present. Built-in payloads use the same field-presence
semantics as author payloads. In `public_fault_record` , `step_sequence` is the `canonical_decimal`
string projection of the aggregate's unbounded mathematical counter; the retained logical-state
fault record keeps the counter as a mathematical integer.

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

-  `type` — required scalar/container type or `instance_reference` .
-  `init` — typed literal used on entry or as a missing creation-binding default.
-  `input: true` — permits a typed creation binding.
-  `external: true` — declares a host-source value copied into machine state.
-  `nullable: true` — permits null for an `instance_reference` ; it is invalid on every
  other type.
-  `machine_id` — optional nominal constraint for `instance_reference` .

A declaration cannot be both `input` and `external` . Both flags are valid only on root variables,
which makes creation and refresh bindings unambiguous.

Variable names and declared payload-field names enter typed CEL records and MUST be CEL-visible
identifiers. They cannot be either Determa activation name `event` or `owner` , any CEL keyword
`false` , `in` , `null` , or `true` , or any CEL v0.25.2 reserved identifier `as` , `break` ,
`const` , `continue` , `else` , `for` , `function` , `if` , `import` , `let` , `loop` , `package` ,
`namespace` , `return` , `var` , `void` , or `while` . The schema rejects these names wherever a
variable or declared payload field is named, including corresponding assign, refresh, send-payload,
and binding keys. This restriction does not apply to machine, state, component, event, or metadata
keys because those names do not themselves become CEL identifiers or typed record fields.

-  An ordinary variable, with neither flag, MUST declare a correctly typed `init` .
-  A root `input` or `external` variable without `init` requires a creation binding.
-  A root `input` or `external` variable with `init` uses the supplied binding when
  present and otherwise uses `init` .
-  Missing, extra, or wrongly typed host creation bindings reject `create` with
  `invalid_binding` . The equivalent component/spawn author-binding mismatch rejects the
  bundle at load with `invalid_binding` .

External variables are read-only to `assign` ; they change only through a successful `env`
/`refresh` step. An `instance_reference` cannot be `input` or `external` , cannot be constructed
from an arbitrary string, and in format 1 MUST be declared `nullable: true` with `init: null` .
Non-null initial instance references are therefore unsupported; only the engine writes a non-null
value through `spawn.bind_to` .

At the logical-state boundary, a non-null `instance_reference` serializes as exactly:

```text
{
  root_instance_id: non_empty_string,
  instance_id: non_empty_string,
  machine_id: identifier,
  machine_version: positive_integer
}
```

Equality compares all four fields. A nullable reference may also be `null` . Assigning any other map
or string to an `instance_reference` is invalid.

Variable scope begins when the declaring state is entered and ends after its exit action. Inner
declarations shadow outer declarations. A transition action runs before source exit (§6.4), so
source-scoped variables remain available to it. Entry and exit actions cannot access the current
envelope; a transition action must copy required event data into a variable whose scope survives the
transition. Lifecycle entry and exit actions may update any lexically visible variable that is still
live at that action. Later actions and outer exit actions observe that tentative value until its
declaring scope is actually destroyed.

### 4.6 State nodes

State types are `simple` (default), `composite` , `parallel` , and `final` .

-  A `simple` state has no nested states or components.
-  A `composite` state requires `initial` and non-empty `states` .
-  A `parallel` state requires at least two `components` ; it does not also declare
  nested `states` , `initial` , or history.
-  A `final` state may declare only `meta` , `variables` , and `entry` .

Common state fields:

-  `variables` — state-scoped declarations.
-  `entry` , `exit` — ordered structured actions and runtime action slots (§5.4).
-  `on_events` — event name to transition or ordered transition list.
-  `deferred_events` — optional non-empty unique list of declared events that this state
  defers under §6.7 when no enabled handler exists in the active hierarchy.
-  `deferred_event_capacity` — optional non-negative signed-64-bit capacity for the
  runtime-local deferred mailbox; valid only on a machine root or inline-component
  root. Omission means logically unbounded.
-  `history` — `none` , `shallow` , or `deep` , valid only on composite states.
-  `meta` — opaque annotation.

A choice pseudostate is represented by an ordered `choice` branch list. Its object may contain only
`choice` and optional `meta` ; it cannot declare `type` or any active-state field. A choice may
appear only as a named child in an active state's `states` map and must be reached by a compound
transition. A bundle machine `root` and an inline component `root` MUST be active states; they
cannot be choice pseudostates.

State paths are relative to the machine root. The reserved literal path `root` identifies the
machine root itself; every other path is the dot-separated sequence of child-state identifiers below
it, without a `root.` prefix. A child state therefore cannot be named `root` .

Native time events (`after`), orthogonal regions with implicit broadcast, submachine documents, and
completion activities are not part of format 1.

### 4.7 Transitions

A transition may contain:

-  `transition_to` — either a target state path or
  `{ history: path.to.composite }` ; omit for an internal reaction.
-  `guard` — a CEL Boolean string, `{provider: name}` , or
  `{lang: name, source: text}` (§5.4).
-  `action` — ordered structured actions, `{provider_actions: name}` , and/or
  `{lang: name, source: text}`
  slots (§5.4).
-  `local: true` — for a non-root composite source targeting its strict descendant,
  preserve the source instead of applying the unmarked transition's source reset
  (§6.4).
-  `lang` — optional expression-language override.

Absence of `transition_to` is the only legal spelling of an internal transition. `internal` is not a
format-1 field. A plain state path always performs normal target entry. The explicit history form
requests history restoration. Event transitions and choice branches may use either target form; an
initial transition MUST use a plain state path. When `local` is present its value MUST be the
literal `true` ; `false` is invalid rather than an alias for omission.

An ordered transition list selects the first branch whose guard is true. An unguarded default MUST
be last.

For an event transition, the **source state** is the state whose handler is selected, even when
dispatch began in a deeper active leaf. Its transition action executes once with that source state's
lexical variable scope while the complete pre-exit configuration is still active. It cannot access a
descendant variable that is outside that lexical scope.

### 4.8 Structured actions

| action | shape | result |
|---|---|---|
| assign | `{ assign: { variable: CEL } }` | one typed write in the executing runtime |
| send | `{ send: { event, to? \| targets?, payload?, correlation_id? } }` | ordered immutable emissions |
| refresh | `{ refresh: { only?: [name, ...] } }` | adopt validated `env` values |
| spawn | `{ spawn: { machine_id, bindings?, bind_to? } }` | create an owned runtime |
| cancel | `{ cancel: { instance: CEL } }` | cancel an addressed owned runtime; null or non-targetable is a no-op |
| stop | `{ stop: {} }` | complete the executing runtime |

`stop` MUST be the final action in its list and is invalid in exit behavior. Once it executes, any
transition target is abandoned and deterministic runtime completion replaces normal target entry.

`spawn` is invalid in exit behavior. Runtime termination performs owned-descendant cleanup before
author exit actions; permitting a later spawn would orphan the new child.

Each `assign` action contains exactly one destination. Ordered action lists provide sequential
writes, and a later action observes every earlier tentative write in the same RTC.

The remaining expression maps never use document member order. One `send` action takes one state
snapshot before evaluating anything, then uses this total order:

1.  supplied payload expressions by ascending identifier UTF-8 byte order;
2.  `correlation_id` , when present;
3.  dynamic `{instance: CEL}` target expressions in target-list order (`to` is a
   one-element list);
4.  payload default materialization and numeric normalization;
5.  target resolution and eligibility checks in target-list order; and
6.  only after every prior operation succeeds, identity/output-counter allocation and
   immutable emission construction in target-list order.

The first expression or eligibility check that fails determines the fault and exact locator. No
later operation runs; the action allocates no identity, counter, or partial emission. Static targets
do not add an expression-evaluation slot.

A component or spawn binding evaluates the `input` map first and the `external` map second, each by
ascending identifier UTF-8 byte order, against one owner-state snapshot taken before any binding
expression in that placement or action. Computed map members cannot observe one another. The first
expression that fails in this order determines the `action_fault` and its exact pointer; no later
expression is evaluated. Target-root defaults are materialized only after every supplied expression
succeeds.

A send target is one of:

```yaml
{ self: true }
{ owner: true }
{ component: component_id }
{ instance: CEL }
{ external: true }
```

`targets` is a non-empty ordered list and is mutually exclusive with `to` . Omitting both means
`self` . One target produces one independent emission. There is no implicit broadcast.

Internal sends do not recursively dispatch. Processing appends them to exact target runtime ready
mailboxes in emission order as part of the producing RTC commit.

An internal send to `self` , `owner` , `component` , or `instance` MUST name a bundle or
machine-local `internal` event. A send to `external` MUST name a bundle `output` event and include
`correlation_id` . Input events cannot be sent by author actions. These event-direction and static
target-shape rules are checked at load time.

There is one reserved-event exception: an owner may explicitly forward `env` to one statically named
component placement. That send MUST use exactly `to: { component: component_id }` , MUST omit
`targets` and `correlation_id` , and MUST have exactly one payload member, `changed` . The `changed`
expression MUST be a non-empty CEL map literal whose keys are literal external-variable names
declared by the target component root. Its values are type-checked against those declarations and
normalized by §5.2. The committed emission carries the target component's specialized fixed `env`
payload from §4.4 and is later delivered explicitly in `internal` mode. No other reserved event may
be sent by author behavior. Dynamic, self, owner, spawned, external, or multi-target `env` sends are
invalid at load. This is explicit forwarding, not broadcast, and does not permit direct
host-to-component ingress.

Target existence is checked when the action executes. `{owner: true}` is valid syntax in every
machine because that machine may run as a contained runtime. If it executes in an aggregate root, no
owner exists and the accepted RTC step faults with `invalid_instance_target` . An exit action runs
after automatic component and owned-instance cleanup; a send from that exit action to a component or
instance already disposed by the cleanup faults deterministically with `inactive_component_target`
or `invalid_instance_target` , respectively. Such sends are not cleanup no-ops.

## 5. Static validation and CEL

### 5.1 Load-time validation

After the §2 source checks and JSON Schema validation succeed, semantic validation returns one exact
load code. A rejection below that names a code in parentheses uses that named code. An unavailable
CEL-profile symbol or overload uses `cel_profile_error` . Every other rejection in this section,
including CEL parse/name/type errors, uses the generic `semantic_validation` code.
`structural_validation` is conformance-harness notation for rejection by JSON Schema; it is not an
engine/spec error code.

A bundle is rejected before any runtime is created when it has:

-  duplicate machine, component, state, variable, or event identities;
-  unresolved machine, state, event, or variable references;
-  unreachable states;
-  an initial transition targeting anything outside its owning composite;
-  a reachable composite without an initial transition;
-  a guard or trigger on an initial transition;
-  an `input` or `external` variable outside a machine root;
-  a variable declaration that cannot obtain a value under §4.5;
-  a fraction/exponent-form `double` source value where an `int` is required, including
  machine `version` , variable `init` , and payload `default` ;
-  an unguarded branch before a later branch;
-  a cycle in state nesting or component placement;
-  a cycle in the complete synchronous-initialization dependency graph. Its nodes are
  machine definitions and inline component-root definitions. It has an edge for every
  referenced or inline component placement, plus every `spawn` that can execute during
  initialization through an entry action, initial-transition action, or any reachable
  initialization choice branch. All possible guarded choice branches contribute
  edges. Event-handler spawns do not add an edge from the handler's machine, but every
  possible spawn target's own initialization graph must still be acyclic;
-  an invalid public/private event direction or correlation;
-  a `deferred_events` name that is undeclared, is an output event, or is one of the
  reserved lifecycle/control events `done` , `determa.component_completed` ,
  `determa.component_failed` , or `determa.spawned_instance_failed` , or a
  `deferred_event_capacity` outside a runtime root;
-  a send whose target, event direction, or correlation is inconsistent;
-  `local: true` without a target, with a non-composite source, or with a target that is
  not a strict descendant of its source;
-  `local: true` on a transition selected on the machine root
  (`root_local_transition`);
-  a transition selected on the machine root whose target is the machine root, or any
  history target that resolves to the machine root (`root_reentry`);
-  a history target naming a non-composite state, a composite whose `history` is `none` ,
  or a composite that strictly contains the transition source;
-  an `assign` or `refresh` in an originating event/initial/choice transition action
  whose destination variable belongs to a state exited by that selected outcome
  (`destroyed_variable_write`);
-  a spawn `bind_to` in an originating event/initial/choice transition action whose
  destination reference belongs to a state exited by that selected outcome
  (`destroyed_reference_binding`);
-  a component/spawn binding with a missing or extra name, or whose inferred expression
  type cannot satisfy the target root declaration (`invalid_binding`);
-  invalid `instance_reference` constraints;
-  an initial/choice cycle that cannot reach a stable state;
-  a transition inside entry, exit, or initial behavior;
-  a `spawn` action in exit behavior;
-  a `stop` action in exit behavior;
-  a `stop` action that is not last in its action list;
-  a provider binding with duplicate dependencies, wrong slot output type, a declared
  input unavailable at that slot, or a source digest that does not match present source; or
-  a CEL parsing, name-resolution, or type-checking failure.

For the destroyed-destination rules above, exits performed by root, component, or spawned-runtime
completion after reaching a final state or executing `stop` belong to the same originating
transition/action outcome. Lifecycle entry and exit action lists are deliberately different: they
may update still-live lexical variables for later lifecycle actions or outer exits before actual
destruction. A `stop` reached from an entry action follows this lifecycle rule.

### 5.2 CEL environments and compile-time types

Every CEL expression MUST be parsed and type-checked at bundle load with the exact variables,
current event schema, owner binding environment, and expected result type for its location.

The activation environment is closed:

-  Bare identifiers are exactly the variables lexically visible from the expression's
  source state, with the nearest declaration winning under §4.5 shadowing. They expose
  the tentative values as of that action or guard's defined snapshot.
-  `event` exists only in the two §5.3 locations. It is a record with exactly one field,
  `payload` , whose typed fields are the materialized immutable payload declaration.
  Event name, `event_id` , target, correlation id, and transport metadata are not
  author-visible.
-  `owner` exists only while evaluating a component placement's `with` bindings. It has
  exactly one field, `variables` . That field is a typed record of the variables
  lexically visible from the containing parallel state after its variables and entry
  actions have run, with nearest-declaration shadowing and the one-snapshot behavior
  from §4.8.
-  Component `with` expressions do not also receive bare owner-variable names or
  `event` . Spawn bindings use the ordinary bare lexical variables of the executing
  action and do not receive `owner` .

No other activation name or record field exists. In particular, engine state, configuration, runtime
identities, counters, host data, and plugin objects cannot leak through a CEL library's ambient
activation.

-  A guard must infer `bool` .
-  An `assign` expression must be assignable to the destination variable.
-  Send payload expressions must satisfy the target event payload declaration.
-  `correlation_id` must infer `string` .
-  Component and spawn bindings must satisfy the target root input and external
  declarations.
-  Instance targets and cancellation expressions must infer `instance_reference` .
-  A dynamic value cannot flow into a concrete destination without an explicit checked
  conversion.

Portable numeric types are exact:

-  `int` is a signed 64-bit mathematical integer. A literal or host value outside
  `-9223372036854775808` through `9223372036854775807` is invalid.
-  `float` is a finite IEEE 754 binary64 value exposed to CEL as `double` . A document
  literal declared for a `float` may use integer, fractional, or exponent form and is
  converted from its exact mathematical lexical value. A runtime value supplied to a
  `float` destination may be a signed-64-bit CEL/host integer or a finite binary64
  value. Conversion uses round-to-nearest, ties-to-even; negative zero is normalized
  to positive zero. A literal that overflows binary64, NaN, and infinities are invalid.

This is the only implicit numeric widening: `int` never accepts a `float` . The rule is applied
before a value becomes observable for variable `init` , payload `default` , root creation,
component/spawn bindings, `env` changed values, author `assign` , and host/authored payloads. Send
and binding expressions may therefore infer `int` for a declared `float` destination; the resulting
value is normalized before it enters state or an immutable envelope.

Within a `map` or `list` , a YAML/JSON integer-form numeric literal is a signed-64-bit CEL `int` ,
while a fractional or exponent-form literal is a finite normalized CEL `double` . Implementations
MUST preserve that distinction while parsing. Every nested numeric leaf must satisfy the domains
above, but map/list members do not receive destination-free integer-to-double coercion.

#### Portable CEL profile

Format 1 pins parsing, checking, and base evaluation to
[CEL specification v0.25.2](https://github.com/google/cel-spec/tree/v0.25.2) , narrowed by the
deterministic profile below. An implementation's library version is irrelevant; the accepted program
and result MUST match this profile.

The profile contains only:

-  `null` , `bool` , signed-64-bit `int` , binary64 `double` , Unicode `string` ,
  `list(dyn)` , `map(string, dyn)` , typed event/owner records, and nominal
  `instance_reference` ;
-  literals, field/map selection, list/map indexing, `?:` , `!` , `&&` , `||` , `in` ,
  same-type equality/comparison, checked numeric `+ - * / %` , string/list `+` ,
  `size(value)` , `has(map.field)` , and `has(event.payload.declared_field)` ;
-  `double(int)` , `int(double)` , and `string(bool|int|double|string)` conversions; and
-  equality/inequality between compatible `instance_reference` values or with null.

No other function, macro, receiver method, protobuf/object construction, `uint` , bytes, timestamp,
duration, optional type, regex, comprehension, iteration, random/time/I/O function, host callback,
or implementation extension is available. An unavailable symbol or overload is `cel_profile_error`
at bundle load. `instance_reference` cannot be constructed, inspected by field, ordered, or
converted to string.

Evaluation is exact:

-  `&&` and `||` use CEL's commutative, error-absorbing result semantics. Either operand
  may be evaluated first because profile expressions are pure. For `&&` , `false`
  absorbs an error from the other operand; for `||` , `true` absorbs an error from the
  other operand. An error is returned when the other operand does not uniquely
  determine the Boolean result. Consequently `false && error` and `error && false`
  are `false` , while `true && error` and `error && true` are errors; `true || error`
  and `error || true` are `true` , while `false || error` and `error || false` are
  errors;
-  `condition ? selected : unselected` evaluates the condition and exactly one selected
  branch; the unselected branch is not evaluated;
-  signed-integer overflow is an evaluation error; integer division truncates toward
  zero, remainder has the dividend's sign, and division by zero is an error;
-  double operations use binary64 round-to-nearest, ties-to-even, and any non-finite
  result is an evaluation error;
-  mixed `int` /`double` operators are rejected at load; destination widening under the
  earlier rule occurs only after expression evaluation;
-  `int(double)` truncates toward zero and errors outside signed-64-bit range;
  `double(int)` uses §5.2 rounding; `string` produces `true` /`false`, canonical decimal
  integer text, or the RFC 8785 finite-number spelling with negative zero already
  normalized;
-  list equality is ordered and recursive; map equality ignores member order and
  compares the exact string-key/value set recursively; dynamic numeric members remain
  type-sensitive;
-  a negative/out-of-range list index or missing map key is an evaluation error;
  `has(map.field)` tests exact string-key presence without reading an absent value;
  `has(event.payload.field)` is false only when that declared optional field has
  neither a supplied value nor a default, and true when it was supplied or materialized
  from a default. A required payload field is always present after envelope validation.
  Selecting an absent optional payload field without first taking a branch that proves
  its presence is an evaluation error. No other typed-record presence macro is
  available; and
-  string size counts Unicode scalar values, string equality performs no
  normalization, and string relational operators compare lexicographically by Unicode
  scalar value.

An incompatible expression is an invalid document, not a runtime `type_fault` . A valid bundle plus
a valid input envelope MUST NOT discover an ordinary assignment or payload type mismatch during
execution.

Runtime CEL evaluation can still fail for value-dependent reasons such as division by zero, invalid
indexing, or an explicit conversion failure. A guard evaluation error is `guard_fault` ; another
expression evaluation error is `action_fault` .

### 5.3 Lifecycle expression visibility

The author-visible `event` binding exists only while evaluating:

-  an `on_events` guard selected for the envelope; and
-  that directly selected event transition's actions.

It does not exist in:

-  state entry actions;
-  state exit actions;
-  initial-transition actions;
-  choice guards or actions;
-  component `with` bindings;
-  completion/cancellation cascades; or
-  root creation.

The engine retains causal identity internally for deterministic tracing and emission identity, but
lifecycle CEL cannot inspect the triggering envelope or its payload.

### 5.4 Exact runtime providers and optional source compilation

#### Human-authored slots

CEL guards and structured actions remain the portable default. A human guard may instead contain
exactly `{provider: name}` ; an action-list element may contain exactly `{provider_actions: name}` .
A custom-language guard or action-list element contains exactly `{lang: name, source: text}` . Names
match `[a-z][a-z0-9.-]*` . `cel` and `determa` are reserved defaults, not shadowable
provider/language aliases. Source text is human-authored program text; its type and grammar are
checked by the selected compiler before a resolved definition becomes executable. This form is not
limited to a DSL that translates to CEL: a configured Python or other runtime language may execute
the source and call a preferred SDK through its locked runtime provider. A guard returns a Boolean;
an action returns the closed action proposals described below, possibly empty after explicit I/O.
Host-language objects stay inside the provider; the Determa context and returned values use the
exact typed boundary. No language name implicitly authorizes installation or SDK access.

These short names request explicit trusted resolution, not mutable runtime lookup. Deployment
configuration selects installed implementations outside the machine. Unknown names never trigger
installation, network discovery or fallback. The source schema rejects full inline bindings. Use
sites require no SemVer, digests, dependency arrays, type reports or capability flags.

#### Explicit resolution and generated locks

There are two separate operations: deliberate `resolve` /`refresh` generates a new lock; loading
authored source with a matching existing lock verifies that lock. Ordinary load, evaluation,
restore, replay and migration never refresh it. These are semantic entry points, not a required CLI
or cross-language ABI.

`schema/provider-lock-v1.schema.json` defines the generated `determa.provider_lock` artifact. Its
content has `authored_source_digest` , `resolved_definition_fingerprint` , `runtime_providers` , and
`compiler_providers` . The fingerprint binds the exact generated output, including runtime closure;
a changed compiler output does not remain valid merely because the compiler reference is unchanged.
Loading source with a lock also requires its stored resolved definition, supplied directly or by
exact fingerprint through a trusted artifact resolver. It never runs a compiler. Fresh compilation
belongs only to explicit resolution/refresh. The source digest is
`hash(["determa-authored-machine-source-1", "1", typed(normalized_authored_source)])` . Authored
normalization uses §8's defaults and numeric rules, retaining short names and source text; it does
not discover providers in inert values. The lock digest is
`hash(["determa.provider_lock", "1", typed(content)])` .

Runtime entries contain `name`, `kind` (`guard` or `actions`) and full `binding`. A generated
runtime-language entry additionally requires `source_locator`, the canonical pointer of its
authored region; it distinguishes different source programs using the same language name. Native
name entries omit that field. Compiler
entries contain `name` , exact `provider_reference` and complete transitive `dependencies` . Runtime
entries are ordered by `(name, kind, source_locator or "")` UTF-8 bytes; compiler entries by name.
Duplicate keys and
unused entries reject. Names are scoped by kind: the same name may serve a guard and an action only
when both entries are explicit. A compiler name cannot silently select a runtime provider or vice
versa. Every native slot has one matching native name/kind entry. Every custom region has one matching
compiler name entry; runtime-language output also requires its exact name/kind/source-locator entry.
No generated runtime binding may escape that inventory. Binding input contracts match the
generated context types; an incompatible kind/type rejects.

Resolution traverses executable grammar slots only, including entry, exit, initial, choice, event
transitions, final-state entry and inline components. Native names expand to locked bindings. Custom
source slots generate compiler regions and source-map locators; authors never supply those JSON
Pointers. Resolution selects either compile-time translation or runtime-language execution from
explicit deployment configuration. Translation emits CEL or structured actions. Runtime-language
resolution emits a generated guard/action binding whose source is the exact region text and whose
source media type, source digest, runtime implementation, transitive runtime dependencies, context
types and capabilities are pinned. The lock and compilation manifest also pin the exact compiler
closure and resulting definition. A compiler cannot install providers or select a runtime closure
outside the explicitly authorized deployment configuration. Source maps bind source digest,
original slot and resolved slot(s), preserving evaluation order. Provider-like keys in metadata,
variable values and event payloads remain data.

`schema/resolved-machine-v1.schema.json` validates generated executable definitions. It accepts
exact native/runtime-language bindings and compiled CEL/structured actions, never unresolved names
or authored custom source
slots. Source/resolved entry points are explicit: a loader never guesses a stage from `format: 1` or
tries both as draft readers. CEL-only source resolves without a lock/compiler because its closure is
empty. Source containing native/custom slots requires its matching lock.

Missing/changed closure, source mismatch, wrong kind, invalid compilation, untrusted installation or
altered installed bytes fails before core execution. Deliberate refresh yields a newly validated
definition, never a hidden fingerprint change. Restore/migration verify the stored resolved
definition and runtime closure; they never consult current source aliases or recompile. Pure
compiled CEL/structured output restores without its compiler. Compiler capabilities are historical
provenance, not inherited runtime guarantees. Mutable host capability reports and credentials are
not executable identity material.

#### Generated executable bindings

Only generated resolved slots contain exactly `{provider: binding}` or `{provider_actions: binding}`
. Each binding is a closed object with `provider_reference` , `source_media_type` , `source_digest`
, optional `source` , `dependencies` , `capabilities` , `input_types` , and `output_type` . The
reference is exactly `{identifier, version, content_digest}` with a lowercase SHA-256 digest and
exact SemVer version. The `source_digest` is the SHA-256 digest of the UTF-8 `source` bytes when
source is present; otherwise it identifies the separately resolved source or binary. Dependencies
are a complete transitive, duplicate-free exact reference closure sorted by
`(identifier, version, content_digest)` UTF-8 bytes. A provider must be installed or injected with
matching code, source, closure, declared types and verified capabilities. A mutable name, installed
package version, or callback with the same name is insufficient. Resolution and trust checks occur
for the *whole* executable definition before creation, evaluation, migration target activation or
checkpoint/archive restoration. Missing, changed or untrusted closure returns
`runtime_provider_unavailable` ; it never falls back to CEL, another provider, or recompilation. The
normalized validated bundle fingerprint (§8) includes every complete binding, including source and
dependency digests, type contract and capability flags. Descriptor source/target fingerprints
therefore also bind exact provider closures. The §12 `guard_binding_digest` encodes the entire
validated `guard` member, including its `provider` key and full binding, with the §8 typed
projection; it does not hash only the three-field reference or a resolved callback name.
Configured-instance health, authorization and host policy remain separate §11.5 checks at use time;
a stored fingerprint or a source manifest never proves that a currently configured instance still
has a capability. Closure discovery follows executable grammar slots only: guard providers and
`provider_actions` in entry, exit, initial, choice and event-transition action lists, including
inline component definitions. Keys with those names inside `meta` , a variable value or an event
payload are inert data and cannot require or select a provider.

A runtime provider registration under the common §11.5 `runtime_provider` category also carries the
closed `schema/runtime-provider-descriptor-v1.schema.json` descriptor with `kind` (`guard` or
`actions` ) and the same complete binding. Its installed implementation exposes the kind-specific
evaluator and, only when separately proved, `inspect_guard` . Direct injection, registration,
duplicate refusal, configuration validation, capability reporting, health, discovery and
authorization follow §11.5. The common extension descriptor's `provider_reference` must equal this
kind-specific binding's reference. The registered implementation, kind-specific descriptor and slot
binding must match exactly. A compiler provider uses the same exact reference and resolver rules but
registers under the distinct `compiler` category.

`input_types` names only the read-only context portions a provider receives: `event: event_envelope`
, `variables: typed_variables` , `owner: owner_reference` , and `env: external_values` . Event
visibility still follows §5.3. The engine presents machine-visible inputs as the exact §16.2 typed
values; native SDK, HTTP, protobuf, socket and other host objects remain inside the provider. The
input is an immutable value snapshot, never a live reference to engine state; mutation of a
provider's local copy cannot mutate a tentative or prior aggregate. A guard binding has
`output_type: bool` and returns one Boolean or a typed failure. An action binding has
`output_type: structured_actions` and returns an ordered finite list of concrete `assign` and `send`
action proposals or a typed failure. Their closed result schema is
`schema/runtime-action-output-v1.schema.json` . A proposal contains §16.2 typed operands and is
never a direct mutation of an aggregate, mailbox, counter, receipt or host journal. The engine
validates every destination, value type, event declaration, target and complete emission against the
same statechart rules before applying it. It also rejects a write to a destination destroyed by the
selected transition. Action proposals execute in their returned order at that slot, among
surrounding structured actions in author order. Every send proposal uses the containing
`provider_actions` action element's RFC 6901 pointer as its §9 emission locator and action document
pointer. There is no invented proposal pointer and no `/provider_actions` or `/send` suffix.
Internal-envelope ordinals and external-intent indexes each start at zero for that slot and continue
across all its send proposals, in returned proposal order and then target order. Assign proposals
consume neither ordinal. Internal and external emissions have separate ordinal sequences; an
internal proposal does not advance the external-intent index or conversely. This preserves distinct
identities even when one slot returns identical sends. Each following ordinary action starts its own
unchanged action-local sequence. Dynamic `spawn` , `cancel` , `refresh` and `stop` are unsupported
in a provider result; they remain available as ordinary structured actions. The provider cannot
rewrite the containing transition, choose another slot, or gain arbitrary internal-state write
authority. Invalid output is a `runtime_provider_output_invalid` provider-boundary error and no
tentative Determa state commits. A selected slot alone invokes its provider; an incoming event's
provider-like string grants no invocation authority. An invoked provider's typed execution failure
follows the ordinary `guard_fault` or `action_fault` rule at that exact slot locator (§10). An
invalid output is mapped to the same closed core fault code according to slot kind, while its
boundary diagnostic retains `runtime_provider_output_invalid` . Either result rolls back tentative
Determa state, but cannot undo I/O already performed by a native or runtime-language provider. A provider that is not process
contained may crash its host instead of returning a typed failure. Its profile cannot promise
containment, a returned fault record or automatic recovery from that crash; durable host recovery
requires the separately proved host contract. Source-language execution inherits its actual runtime
closure's guarantees, never the compiler's historical capability claims.

`capabilities` explicitly declares `deterministic` , `pure` , `portable` ,
`semantically_introspectable` , `process_contained` , and `external_io_capable` as Booleans,
independently verified against the exact provider closure and host policy. Self-assertion is
insufficient. Effective guarantees in the first five categories hold only if *every* participating
CEL/runtime provider and applicable host policy supplies them. Historical compiler capabilities do
not change the runtime profile of already compiled output. Effective `external_io_capable` is true
if *any* participating provider may perform external I/O; unknown/unverified I/O is treated as
possible I/O. An embedding API returns the effective profile alongside the closed §8 core result;
the profile is not a member of that portable result or checkpoint. The §11.5 host capability report
also discloses the configured provider claims. Missing guarantees are false for an explicitly
opted-in embedded weak profile; a host profile requiring them rejects at load. A native provider may
deliberately perform I/O during evaluation, before commit. Such I/O survives a rolled-back
transaction, failed CAS, or engine fault. This profile makes no automatic CAS reevaluation,
deterministic replay, purity, portable execution or crash-safe exactly-once claim. A host authority
guard protects Determa writes only; external safety needs separately proved destination fencing,
idempotency or reconciliation. Committed effect handlers (§11) remain the recommended external I/O
boundary.

Semantic inspection (§12) calls a guard only through its separately proved, bounded, nonmutating
`inspect_guard` entrypoint, which preserves ordinary guard truth for the given snapshot and charges
the shared deterministic inspection fuel. The ordinary evaluator is never used as an inspection
shortcut, even when `pure` is true. The engine preflights all potentially reached guards; if any
exact closure lacks this entrypoint it returns `inspection_capability_unavailable` before invoking a
guard. Structural inspection only identifies bindings and never calls providers.

Optional source compilation uses `determa.language_source` version 1 and
`determa.compilation_manifest` version 1 (`schema/language-source-v1.schema.json` and
`schema/compilation-manifest-v1.schema.json` ). Source content has exactly `template` , `regions` ,
and `dependencies` . A region gives its `kind` (`guard` or `actions` ), canonical JSON Pointer
`locator` , exact compiler `provider_reference` , `source_media_type` , and `source` . The generated
template follows the authored schema. Locators identify disjoint custom guard objects or custom
action-list elements, including lifecycle slots. Each region's source equals that slot's source
text. Duplicate, overlapping, mismatched-text or wrong-kind regions reject. Bounded compilers
resolve exact dependency closure and process regions in array order. They may translate to CEL or
structured
actions, or generate exact runtime-language bindings under the authorized resolution policy above.
A runtime-language binding retains the region's exact source text and media type; its runtime
reference/dependency closure identifies the interpreter, compiled wrapper and SDKs actually used.
The tool splices action results at the named element, preserving surrounding author order.
The generated definition then passes the resolved-definition loader. The manifest binds source
digest, exact compiler closure, generated validated-bundle fingerprint, source capabilities and
generated `source_map` . Every map entry has `source_locator` and ordered `resolved_locators` ;
entries follow region order. Guard regions map to one resolved guard; action regions map to their
generated action elements, possibly none for an empty result. Every locator is checked against the
corresponding immutable source/resolved definition. Source compilation and manifest envelopes each
contain exactly `artifact_format` , `artifact_schema_version: 1` , `content` , and `artifact_digest`
. Their digest is `hash([artifact_format, "1", typed(content)])` using §9 SHA-256/JCS and the §16.2
recursive typed projection; a digest mismatch rejects. The compiler list is the duplicate-free exact
closure used by the regions, including transitive dependencies, sorted by
`(identifier, version, content_digest)` UTF-8 bytes. A manifest with a fingerprint unequal to the
strict generated bundle rejects before creation or activation. Source-region locators and
dependencies are validated before compiling; `language_compilation_failed` and
`language_compilation_limit_exceeded` are distinct failures. Source compilation is optional: direct
runtime slots are first-class, and restoring a complete generated definition requires its executable
runtime closure but no compiler. Fresh compilation requires exact compiler closure. Restoration
never silently recompiles or substitutes a mutable alias. Source-level inspection provenance
additionally requires the exact source and manifest; executable structural inspection needs only the
validated definition. No source or manifest embeds endpoint, scope credentials or SaaS-specific
machine semantics.

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

This tagged union is the normalized immutable target. Author target shorthands from §4.8 resolve to
one union member when an emission is created; the resulting envelope stores the complete member and
never resolves it again. A component target therefore names one activation incarnation, not merely a
reusable `component_id` .

A portable mailbox envelope additionally stores required `cause_id` and `source` . Host input uses
`cause_id: event_id` and `source: { host: true }` . Internal emissions preserve their deterministic
cause and use `source: { runtime: target_identity_of_emitter }` ; system emissions use their exact
`system:` source locator.

Portable automatic deferral uses the aggregate/checkpoint mailbox model. The caller owns an envelope
until mailbox admission commits. Before that boundary, the core atomically validates the delivery
mode, event declaration, direction, payload, correlation, exact target incarnation, and target
eligibility. Rejection returns the prior state unchanged, allocates no logical identity, and leaves
the envelope caller-owned. After committed admission the envelope exists in exactly one engine-owned
lifecycle location: one runtime's ready mailbox, that same runtime's deferred mailbox, or a
terminal/disposal receipt under §17.3. Under the optional §21 lossless delivery profile, source
ownership and acknowledgement follow the committed admission or durable terminal-transfer boundary;
a later machine disposition never becomes a source-broker retry.

`input` mode MUST name a bundle `input` event and may supply only the `root` or `spawned_instance`
target member for a running runtime. Direct use of the `component` member rejects with
`invalid_instance_target` . If the declaration has `correlates_to` , the envelope MUST carry a
non-empty `correlation_id` . Reserved `env` is the exception below.

`internal` mode MUST name a bundle or machine-local `internal` event, one fixed reserved lifecycle
event, or the statically targeted component `env` emission defined by §4.8, and may carry any
eligible target member. The core does not authenticate how the caller obtained an envelope; the
host/queue adapter is responsible for using `internal` mode only for an immutable core emission it
is delivering. Using an input event in `internal` mode, an internal/reserved event in `input` mode,
or an output event in either mode rejects with `invalid_event` .

A completed or faulted root, and a faulted spawned runtime, reject delivery with
`invalid_instance_target` . A completed or faulted component rejects internal delivery with
`inactive_component_target` ; it cannot accumulate work that will never run. Rejection is atomic.
Reading a terminal aggregate is not a processing operation and returns its existing status without
emissions.

Aggregate-root fault terminality overrides descendant status. Once the aggregate root is faulted,
every admission targeting the root, a component, or a spawned descendant is rejected atomically with
`invalid_instance_target` , even when the descendant's retained diagnostic status was running. The
aggregate, counters, and emissions remain unchanged. Read-only inspection returns that exact faulted
aggregate and no emissions.

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

It has no correlation id. `changed` is non-empty and every field must match a declared root external
variable on the targeted runtime. A `refresh` action is valid only in the selected `env` handler and
copies the requested changed values into those root variables atomically. `refresh: {}` selects
every field in the current `changed` map; `refresh.only` selects exactly its named subset and every
name MUST occur in that map. If a selected `refresh.only` name is absent from the accepted `changed`
record, the action raises `action_fault` at the exact absolute pointer formed by appending
`/refresh/only/<index>` to the action's pointer in the validated bundle. The ordinary RTC fault rule
rolls back every write, emission, counter allocation, and other tentative change from the step. This
is not a pre-step rejection: the envelope is structurally and semantically valid, and the failure
depends on the selected handler and action. The first absent item in ascending list-index order
determines the fault. An omitted `only` never has this failure because it selects exactly the names
present in `changed` . Changed fields not selected by a committed `refresh` are not retained by the
core. There is no external-source map in logical state: the normalized `env` envelope is the only
source value visible during that RTC step.

For an aggregate root or spawned runtime, `env` arrives only through the `input` mode exception
above. For a component, it arrives only through the owner-to-static-component forwarding form in
§4.8 and subsequent explicit `internal` delivery. Direct host input to a component remains invalid.

### 6.2 Run to completion

One mailbox `step` processes at most one accepted envelope for one explicitly addressed runtime. The
step is non-reentrant and atomic. Internal emissions append to exact target-runtime ready mailboxes.
External emissions remain output intents. No operation chooses another runtime, drains an aggregate,
or introduces a round-robin scheduler.

A successful internal `send` action resolves and validates its exact target incarnation, allocates
the next acceptance and queue sequences, and tentatively appends the complete entry to that target's
ready mailbox at that action's position in emission order. The entry is visible to later lifecycle
cleanup in the same RTC but is never dispatched recursively. The enclosing RTC commit makes the
append durable; rollback removes it and restores both counters.

If the target is already completed, faulted, pending disposal, or absent when the send action
executes, the sender faults with the existing exact target code. If the target remains
retained-faulted after a later fault in the same RTC, the entry remains in its frozen ready mailbox.
If later same-RTC cancellation, natural completion, or aggregate completion disposes the target,
cleanup removes the entry and records one core `lifecycle_disposition` under §8. The producing
emission becomes an `internal_disposed` reference to that record rather than an `internal_mailbox`
reference. A committed result therefore contains either one mailbox entry or one disposition for the
emitted envelope, never both and never neither.

The same-RTC target outcome is closed:

| target state at enclosing RTC commit | emitted-envelope result |
|---|---|
| running or pending initialization | one ready-mailbox entry plus `internal_mailbox` reference |
| retained faulted/frozen without disposal | one frozen ready-mailbox entry plus `internal_mailbox` reference |
| successfully cancelled, completed, or aggregate-disposed | no mailbox entry; one lifecycle disposition plus `internal_disposed` reference |
| lifecycle cleanup or any enclosing action faults | complete enclosing RTC rollback; no entry, disposition, emission, or counter allocation commits |

No transition between these rows may create a second copy or an unreferenced committed envelope.

Other runtimes may be processed concurrently only when the host's persistence layout provides
serializable ownership of any aggregate state they might share. Two calls MUST NOT concurrently
mutate the same root ownership aggregate.

### 6.3 Hierarchical dispatch

The envelope is resolved one state level at a time from the deepest active state to the runtime
root. At each level the engine first evaluates that state's declared handler, if any, using its
ordered branches. An enabled branch handles the event. If no branch at that level is enabled and
that same state declares the event in `deferred_events` , the event is deferred immediately. Only
when neither result applies does resolution continue to the parent.

-  A false guard does not consume the envelope. After every branch at that level is
  false, same-state deferral is checked before ancestor search.
-  A guard evaluation error faults the step.
-  The first true branch in an ordered list wins.
-  If a state has no enabled branch and defers the event, resolution stops with
  `deferred` ; no ancestor handler is evaluated.
-  If the root has neither an enabled branch nor a matching deferral, the disposition is
  `unhandled` .

The only exception is an unhandled `determa.component_failed` or `determa.spawned_instance_failed`
envelope, which faults the owner as specified in §10.2.

Determa deliberately gives ordered guard branches declaration-order priority: the first true branch
wins, and guards need not be mutually exclusive. UML state-machine models do not assign this
priority to competing guarded transitions. Authors porting a UML model MUST NOT assume guard-order
independence. An unguarded default, when present, MUST be last, which keeps the priority
unambiguous. Without a default, all-false guards continue ancestor search.

The closed per-level precedence table is:

| current state level | handler result at this level | defers at this level | result |
|---|---|---|---|
| any | first unguarded or true branch | absent or present | `handled`; same-state handler wins |
| any | every declared branch guard is false | present | `deferred` at this level |
| any | no handler declaration | present | `deferred` at this level |
| non-root | every branch false or no handler | absent | continue to parent |
| root | every branch false or no handler | absent | `unhandled` |
| any | an evaluated guard faults | either | atomic engine fault; deferral is not consulted |

Thus a child handler overrides a parent deferral because the child handles before the parent is
reached. A child deferral overrides a parent handler because resolution stops at the child. At one
state, an enabled handler overrides that state's deferral, while an all-false handler plus that
state's deferral defers. The rule considers only the addressed runtime's ordinary state hierarchy;
components and owned spawned runtimes have isolated configurations and mailboxes. Explicit fan-out
creates independent envelopes that are resolved independently.

An unhandled result is not a core fault. The accepted envelope is consumed into its terminal receipt
and is not retained in logical state. A host may separately audit or dead-letter that terminal
disposition, but no transport plugin may reinterpret it as machine deferral.

### 6.4 Transition execution order

Determa deliberately executes a selected external transition in this exact order:

1.  evaluate the selected guard;
2.  execute the originating transition actions in the source configuration and source
   variable scope, then completely resolve any targeted choice chain as described
   below;
3.  exit the active source path from innermost to outermost, stopping below the
   transition boundary defined below;
4.  enter the target path below that boundary, from outermost to innermost; and
5.  restore explicitly targeted history or follow nested initial transitions until a
   stable leaf is active.

Because step 2 precedes exit, a write to a variable or `bind_to` reference destroyed by step 3 would
have no observable result. §5.1 therefore rejects such a transition at load time. A write to an
ancestor-scoped destination that survives the exit remains valid. For a choice chain, every possible
selected branch must satisfy this rule. Entry/exit actions invoked by steps 3–5 instead use the
lifecycle rule in §4.5.

This differs from canonical UML ordering, which treats the transition effect as behavior of the edge
after source exit and before target entry. Determa instead keeps the source context intact while the
transition action runs. Authors familiar with UML MUST rely on the order above for Determa
definitions.

A transition whose target is a choice forms one **compound transition**:

1.  the event or initial transition that first targets a choice is the originating
   transition;
2.  after its actions, evaluate that choice's branches in order and select the first
   true guard or final unguarded branch;
3.  execute the selected branch actions immediately, before any state exit;
4.  if its target is another choice, repeat steps 2–3; otherwise that target is the
   compound transition's final target; and
5.  compute one transition boundary from the originating source and final target, then
   perform one exit/entry sequence.

Choice guards and actions use the originating transition's lexical variable scope after all
preceding actions in the chain. They never receive the `event` binding. Every possible incoming
origin/choice path MUST therefore parse, name-resolve, and type-check in that origin's scope. A
guard or action fault rolls back the whole RTC before any state exit.

For an event transition, the originating source is the state whose handler was selected. For an
initial transition, it is the already-entered containing composite: the initial and selected choice
actions run in that composite's scope, the full chain resolves before any descendant entry, and no
state is exited. Choice-to-choice paths never enter or exit the transient choice objects themselves.

Entry and exit actions belong to states. Entry initializes the state's variables, runs its entry
actions, then follows its initial transition unless explicit history restoration supplies the
descendant configuration. When a state exits, the engine first performs the automatic owned-child
cleanup defined by §7.2, then runs the state's exit actions, then destroys its variables.

An internal transition has no `transition_to` and executes actions without exit, entry, or initial
descent. No state is exited or entered, including states between the active leaf and an ancestor
state whose handler was selected; the active configuration is identical before and after.

For a transition with a target, the **transition boundary** is computed from the resolved
source/target relationship. The root is an invariant boundary: ordinary transitions never exit or
re-enter it.

-  A plain self-transition on a non-root source uses the parent of the source as its
  boundary, so it exits and re-enters the source. A root self-transition is rejected
  at load time.
-  When a composite source strictly contains the target, an unmarked transition also
  uses the parent of the source as its boundary, except that a root source uses the
  root itself. It exits and re-enters a non-root source, resetting that subtree
  including the source's variables and lifecycle actions. With a root source, only
  active descendants are exited and entered; root variables and lifecycle actions
  remain untouched.
-  For that same strict-descendant relationship, `local: true` uses the source as its
  boundary. It exits and enters only descendants, leaving the source's variables and
  lifecycle actions untouched. A machine-root source cannot carry `local: true` ;
  root invariance would make it a noncanonical no-op.
-  When the target strictly contains the source, the target is the boundary. The active
  source path exits up to but excluding the target, and the target is not re-entered.
  External re-entry of a proper ancestor target is not expressible in format 1.
-  For unrelated source and target states, their ordinary least common ancestor is the
  boundary.

For example, let composite `c` contain active leaf `a` , with the handler selected on source `c` and
target `c.a` :

```text
unmarked:    exitA, exitC, enterC, enterA
local: true: exitA, enterA
```

Local self-transitions, local transitions to ancestors, and local transitions between unrelated
states are deliberately unsupported.

### 6.5 Choice and history

A choice is transient and follows the compound-transition algorithm in §6.4. The first true branch
is selected and the final branch MUST be unguarded.

Each composite whose `history` is `shallow` or `deep` maintains an optional history record. When an
RTC exits that composite, the engine copies its pre-exit active descendant configuration immediately
before the first exit action in the composite's subtree. The copy becomes the new history record
only if the RTC commits. A transition that passes through or changes descendants without exiting the
composite does not update its record.

Shallow history records only the active direct substate. Deep history records the full active
descendant configuration. A plain `transition_to: path.to.composite` always restarts that composite
through its `initial` transition, even when a history record exists.
`transition_to: { history: path.to.composite }` enters the composite and:

-  restores the recorded direct substate for shallow history, then follows that
  substate's normal initial descent;
-  restores the recorded descendant path for deep history; or
-  follows the composite's `initial` transition when no record exists.

Entry actions run and state-scoped variables are initialized for every restored state, outermost to
innermost. History restores configuration, not destroyed state-scoped variable values. A choice
branch may select history using the same object form; an initial transition cannot target history.

A transition from a non-root composite source to its own history is allowed without `local: true` .
It uses the plain self-transition boundary: the engine captures the pre-exit configuration, exits
and re-enters the composite, and restores the same descendant configuration. Lifecycle actions rerun
and state-scoped variables are reinitialized even though the final active leaf is unchanged. A
history target that resolves to the machine root is rejected.

A history target may carry `local: true` when the resolved history composite is a strict descendant
of the composite source. The local boundary from §6.4 applies, then history restoration occurs
normally in step 5.

### 6.6 Stop interruption

`stop` is an immediate interruption point for the runtime executing it:

-  in a transition or choice action, it abandons the transition/choice target and skips
  the remaining choice chain;
-  in an entry action, it skips the rest of that runtime's entry/initial descent;
-  in an initial-transition action, it abandons the initial target; and
-  during component or spawned initialization, it completes that contained runtime,
  not its owner.

Actions and emissions performed before `stop` remain tentative results of the RTC. When `stop`
occurs in lifecycle entry behavior, preceding entry writes and later exit writes to still-live
variables use §4.5 and may feed subsequent exit behavior before destruction. The engine then
performs that runtime's ordinary completion: cancel retained descendants, dispose
allocated-but-not-initialized component placements without running their author behavior or emitting
component notifications, exit the currently entered partial configuration deepest-first, and
finalize completion. Identity/counter allocations made before `stop` remain consumed if the RTC
commits.

If a parallel owner's entry action stops its runtime, component identities allocated before entry
are disposed and no component binding or initialization runs. If a component stops during its own
initialization, it emits its normal component-completed notification and later placements continue
in declaration order. If a spawned child stops during its initialization, it emits normal
spawned-instance `done` , is disposed, and any `bind_to` value remains the resulting non-targetable
nominal reference; later owner actions continue.

Any cleanup or exit fault rolls the enclosing RTC back and uses the normal fault-finalization rules.
Thus retry starts from the same pre-step state, and no skipped initialization or emission leaks from
the failed attempt.

### 6.7 Deferred mailboxes and automatic recall

Each runtime has one FIFO ready mailbox and one FIFO deferred mailbox. They are isolated from every
other runtime even when the runtimes share one ownership aggregate. A `deferred` result atomically
removes the selected ready entry, increments its `deferral_count` , allocates a new `queue_sequence`
, and appends it to that runtime's deferred tail. The immutable `acceptance_sequence` and the
complete normalized envelope, including `event_id` , `cause_id` , source, exact target incarnation,
payload, and optional `correlation_id` , do not change. Deferral never creates an emission or a
second event.

After every successful `handled` RTC step has completed all transition actions, choice resolution,
exit/entry behavior, initial descent, completion behavior, and lifecycle cleanup, the engine
performs one bounded structural recall phase against the resulting stable configuration. A
`deferred` classification is not a handled RTC step and does not immediately recall the entry it
just deferred. The phase freezes the deferred entries present at its start and examines each exactly
once in existing `queue_sequence` order. It evaluates no guard, action, or other expression.

For each frozen entry, structural recall walks the active states deepest-to-root without evaluating
guards and applies this closed rule at each level:

| current level structure | recall action |
|---|---|
| handler declaration present, with or without same-state deferral | move to ready tail; stop structural walk |
| no handler declaration and matching deferral present | remain deferred; stop structural walk |
| neither declaration at a non-root level | continue to parent |
| neither declaration at the root | move to ready tail as no longer deferred |

Every moved entry receives a fresh `queue_sequence` and appends to the ready tail in the same
relative order. Existing ready entries and entries emitted by the just-completed RTC remain ahead of
recalled entries. Entries not in the frozen snapshot are not examined. Recall does not dispatch
recursively, consume a logical-step sequence, or fault the successful RTC merely because a deferred
guard would fault if evaluated.

When a recalled entry later reaches the ready head, the engine performs the complete §6.3
level-by-level evaluation. An enabled handler wins over a deferral on that same state. An all-false
handler with a same-state deferral, or a deeper deferral reached before an ancestor handler, moves
the entry back to the deferred tail. A guard fault faults that selected event's own step. A later
successful handled RTC may make it structurally recall-eligible again. This repeated movement is
bounded to one examination per recall phase and preserves envelope and acceptance identity while
each queue placement receives a new scheduling identity.

`deferred_event_capacity` limits the number of entries in that runtime's deferred mailbox after the
attempted append. Omission is logically unbounded; `0` forbids every deferral. An overflow rolls
back the attempted RTC/classification, consumes the causal entry into a terminal `faulted` receipt
with code `deferred_event_capacity_exceeded` , and applies the normal root or contained-runtime
fault rule. The event is never dropped, left at the ready head for an infinite retry, or delegated
to a plugin overflow policy.

Deferred events do not expire. A successful runtime cancellation, natural completion, or aggregate
completion disposes every remaining ready and deferred entry in canonical queue order and creates
one terminal `disposed` receipt per entry with respectively `runtime_cancelled` ,
`runtime_completed` , or `aggregate_completed` . Those receipts are part of the same atomic
lifecycle commit. A cleanup fault rolls back all disposal records and mailbox removal; fault-frozen
diagnostic runtimes retain their mailboxes unchanged until a later successful owner cancellation
disposes them. Reserved completion and contained-failure events cannot be declared deferred, so
machine behavior cannot hold them indefinitely.

## 7. Components, spawning, and lifecycle

Every lifecycle cascade uses one recursive postorder algorithm:

1.  At a runtime, direct retained component children are visited first in descending
   order of `(owning_state_document_pointer UTF-8 bytes,
   state_activation_sequence, component_declaration_index,
   component_activation_sequence)`.
2.  Direct retained spawned children are then visited in ascending order of
   `(holder_rank, holder_declaration_pointer UTF-8 bytes,
   holder_state_activation_sequence, spawn_sequence)`. A bound child has
   `holder_rank = 0` and uses the exact RFC 6901 variable-declaration pointer plus the
   activation sequence of that declaring state. An unbound child has
   `holder_rank = 1` , empty holder pointer, and holder activation sequence zero.
3.  Each selected child recursively visits its children by the same rules before that
   child is finalized. A running child then executes active exits innermost through its
   root; a completed child has no active configuration; a retained-faulted or
   root-frozen subtree skips all author exit behavior. The finalized subtree is
   disposed before the next sibling is visited.

The component tuple's declaration index is its zero-based index in `components` ; its state
pointer/activation distinguishes repeated or nested placements. A runtime's spawn sequence is unique
and therefore breaks any remaining spawned-child tie. Already disposed children are absent.
Emissions append exactly when their exit action runs, so the traversal above is also the total
cascade-emission order.

State-scope cleanup selects only children bound to variable declarations whose currently active
state scopes are exiting; unbound children and children held by surviving scopes are not selected.
Runtime completion, `stop` , and root termination select every direct component and spawned child,
including unbound children. Explicit `cancel` selects its addressed spawned subtree. Parallel-state
exit selects every placement of that parallel state. Natural component/spawned completion applies
the same traversal to that runtime's descendants. These selection rules plus the traversal are
reused without variation for explicit cancel, holder cleanup, component disposal, natural
completion, `stop` , and root cascade. A fault rolls the entire enclosing cascade back atomically.

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

A placement declares exactly one of `machine_id` or inline `root` . `component_id` is unique across
the containing machine.

An inline `root` uses the containing machine's `(namespace, machine_id, machine_version)` as its
definition identity. Its exact component placement pointer is the additional inline-definition
discriminator used by §9. Nested inline placements use their own full document pointers, so neither
runtime nor effect identities can collide with a sibling placement.

Entering a parallel state is part of the owner step:

1.  allocate component runtime identities in declaration order and mark each placement
   pending-initialization;
2.  initialize the parallel state's variables and run its entry actions;
3.  evaluate the statically validated `with.input` and `with.external` expressions in
   the order and snapshot defined by §4.8, using owner variables only, then apply
   target-root `init` defaults;
4.  create and initialize each component in declaration order; and
5.  reach a stable configuration in every component before the owner step commits.

`pending-initialization` and `pending-completion` are tentative intra-RTC phases, not logical-state
statuses. They never appear in committed state, results, read-only inspection, or the input to a
later call. Stable committed root statuses are exactly `running` , `completed` , and `faulted` ;
stable retained component statuses are exactly `running` , `completed` , and `faulted` ; stable
retained spawned statuses are exactly `running` and `faulted` , because completed spawned runtimes
are disposed before commit.

The triggering event is unavailable to entry actions and `with` . A transition action must first
copy required payload into an owner variable.

Components have isolated configurations, variables, and ready/deferred mailboxes. An event reaches a
component only through an explicit emission targeting its nominal component runtime identity. No
parent, sibling, or owned child participates in its deferral decision.

The `{component: component_id}` syntax resolves at emission time to the allocated
pending-initialization or running placement identity, including its activation sequence. This
permits the parallel owner's entry action to address identities allocated in step 1. The immutable
envelope stores the complete component target from §6.1. Delayed delivery to a disposed placement
therefore cannot accidentally reach a later re-entry incarnation with the same `component_id` .

An emission to a pending-initialization placement is tentative and cannot be delivered before the
enclosing owner step commits. If initialization succeeds, later delivery may target the running
component. If the component initializes and immediately completes, or initialization commits as an
isolated component fault under §10.2, the earlier emission remains in the committed owner result
with its exact target, but later delivery rejects it with `inactive_component_target` . If the
enclosing owner step faults, both the placement and tentative emission roll back.

A component reaching its root final state or executing `stop` becomes completed and emits one
`determa.component_completed` envelope to its owner using the fixed payload from §4.4:

```text
{ component_id, component_runtime_id }
```

When every component placement is complete, the same successful step also emits the parallel branch
of the fixed reserved `done` payload from §4.4:

```text
{
  relationship: "parallel",
  state_path: dotted_identifier_from_root,
  owner_runtime_id: non_empty_string
}
```

Both notifications append to the owner ready mailbox in the committed emission order, after work
already there. `determa.component_completed` precedes `done` .

For `determa.component_completed` , source is the component runtime and target is its owner runtime.
For the all-components-complete `done` , source and target are both the owner runtime. Both retain
the completing component step's cause.

Completed and faulted component runtimes remain inspectable until their parallel owner exits, but
are terminal and non-targetable. Running components accept internal delivery. During one tentative
owner RTC, an entry action may resolve and emit to an already allocated pending-initialization
identity as defined above; this is not delivery to a committed pending runtime.

Exiting the parallel state synchronously cleans and disposes its retained component runtimes with
the canonical cascade above, atomically with the owner transition.

Reset-in-place, component history retention, shared variables, implicit broadcast, and direct
host-to-component ingress are unsupported.

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

The action allocates a deterministic child identity, evaluates its statically validated root input
and external bindings in the order and snapshot defined by §4.8, applies target-root `init`
defaults, optionally stores its nominal `instance_reference` in `bind_to` , and initializes the
child to a stable configuration. A binding-expression evaluation error is `action_fault` ;
missing/extra names and incompatible inferred types were already rejected at load with
`invalid_binding` . Creation and binding are atomic with the parent step. `bind_to` MUST name a
compatible nullable `instance_reference` whose current value is null; otherwise the step faults with
`binding_not_empty` .

The declaring scope of the `instance_reference` used by `bind_to` defines the bound child's maximum
lifetime. Binding records that declaration as the child's lifetime holder; copying or later
replacing the nominal reference value neither transfers nor erases that association. When the
holder's state exits, every running or retained-faulted associated child is synchronously cancelled
and disposed before the declaring state's exit actions run in the canonical cascade order above. The
cleanup is atomic with the owner transition. Emissions from child exit actions are returned in that
cancellation order as part of the owner RTC step. A holder with no live or retained-faulted
associated child requires no cleanup. A cleanup failure rolls the owner RTC step back and finalizes
`cascade_fault` under §10.1. A root-scoped holder therefore lets its child survive every ordinary
transition; a state-scoped holder ties its child to that state's lifetime.

Ownership is not otherwise tied to the transition that spawned the child. An unbound child or a
child whose holding reference remains in scope is processed only when an explicit envelope targets
it and the host invokes a mailbox `step` for that exact runtime.

After its expression type-checks as `instance_reference` , `cancel` is always well-formed. If the
expression currently addresses a running or retained-faulted directly or transitively owned
instance, it synchronously cancels descendants with the canonical cascade, runs remaining eligible
exit actions, disposes logical child state, and invalidates the reference as a target. A retained
reference remains serializable and comparable but no longer addresses a live runtime. Faulted
instances accept cancellation only for this cleanup; they reject ordinary events and sends.
Cancellation of a retained-faulted instance follows the frozen-subtree rule in the canonical
cascade. Every other resolved value succeeds without effect as described below.

If the cancellation expression evaluates to null, or does not currently address a running or
retained-faulted directly or transitively owned instance, the action succeeds as a no-op. The action
itself changes no ownership or counters and produces no fault or emission; the surrounding RTC step
continues normally.

Natural child completion performs the same descendant cleanup and emits the spawned-instance branch
of the fixed reserved `done` payload from §4.4 to its immediate owner:

```text
{
  relationship: "spawned_instance",
  instance: instance_reference,
  instance_id: non_empty_string,
  machine_id: identifier,
  machine_version: positive_integer
}
```

For this `done` , source is the completed child runtime and target is its immediate owner runtime.
It retains the child's completing cause.

The completed child subtree is then disposed and its `instance_reference` becomes non-targetable.
The completion envelope retains the nominal identity needed by the owner. A faulted child subtree is
instead retained for diagnostics and may be disposed only by explicit owner cancellation or
owner/root cleanup.

Remote provisioning is never core `spawn` . A machine requests it through an external output intent
and receives declared correlated input events from a host extension.

### 7.3 Runtime and aggregate-root completion

When any runtime reaches its root final state or executes `stop` , it synchronously cancels all
retained owned descendants with the canonical cascade, runs its active exit actions deepest-first,
and becomes completed before the RTC commits. Component and spawned-runtime retention and
notifications then follow §7.1 and §7.2.

The resulting emission order is exact:

1.  author emissions produced before the completion trigger, including final-state
   entry actions or actions preceding `stop` ;
2.  descendant cancellation/cleanup emissions in the deterministic cascade order;
3.  active-state exit-action emissions, innermost through the runtime root; and
4.  after every exit succeeds, the runtime's reserved completion notification.

For a component, step 4 emits `determa.component_completed` and then, when it completes the
containing parallel placement set, the parallel `done` . For a spawned runtime, step 4 emits
spawned-instance `done` . The aggregate root has no reserved completion notification. Any fault
rolls back every emission in this sequence.

Completion exits the runtime root as well as its active descendants. After its root exit action,
every state-scoped variable is destroyed and the completed runtime has an empty configuration and
variable map. A retained completed component therefore exposes identity, status, history, counters,
and prior fault diagnostics but no active configuration/variables. A completed spawned runtime is
disposed as defined by §7.2.

For the aggregate root, the engine retains terminal identity/status, history, counters, and
fault-history diagnostics and returns `completed` ; it retains no component or spawned descendant.
No new ordinary envelope may target it.

## 8. Foreground interface and logical state

Language APIs may use idiomatic names, but every implementation must provide behavior equivalent to:

```text
create(bundle, machine_id, root_instance_id, creation_id, bindings)
  -> { status, state, emissions, lifecycle_dispositions, fault, rejection }

admit(bundle, prior_state, ordered_deliveries)
  -> { status, accepted, state, rejection }

step(bundle, prior_state, target_runtime_id)
  -> { status, disposition, state, emissions, lifecycle_dispositions, fault, rejection }
```

With CEL and pure structured actions this is a pure foreground state transform. An explicitly
installed impure runtime provider (§5.4) weakens that invocation as declared by its effective
capability report; the API shape and state ownership are unchanged.

`admit` validates its complete ordered batch before mutation, then appends each envelope to its
exact target runtime's ready tail in caller order. It allocates immutable aggregate-wide
`acceptance_sequence` and mutable `queue_sequence` values independently. If any member is invalid,
the entire batch is rejected byte-for-byte. `step` names one exact running runtime and processes
only its ready head; an empty ready mailbox returns `not_runnable` without mutation. Neither
operation selects or advances another runtime.

Before mailbox selection, `step` resolves its target with this closed rule:

| target resolution | result |
|---|---|
| exact retained component runtime whose status is `completed` or `faulted`, or that is otherwise inactive pending owner disposal | reject with `inactive_component_target` |
| absent or disposed runtime identity, or a root/spawned machine runtime that is completed, faulted, or otherwise non-targetable | reject with `invalid_instance_target` |
| exact running runtime with an empty ready mailbox | `not_runnable` with null `rejection` |
| exact running runtime with a ready head | perform the one atomic mailbox step |

The first two rows are pre-step rejection outcomes: they preserve the prior aggregate byte-for-byte,
allocate nothing, and return no emissions or lifecycle dispositions. A retained inactive component
receives the component-specific code because its exact activation identity is still present for
diagnosis. Once that component has been disposed and is no longer retained, the same stale identity
falls into the absent-runtime row and uses `invalid_instance_target` . Root and spawned runtimes
never use `inactive_component_target` .

`create` is the sole portable aggregate creation operation. Before author initialization it sets
aggregate next acceptance and queue sequences to zero and creates every runtime with empty
ready/deferred mailboxes. Each initialization internal emission then allocates from those counters
in emission order and starts with `deferral_count: "0"` ; absent such emissions both next counters
remain zero. Creation-time target disposal follows the same disposition rule as `step` .

`status` is `running` , `completed` , or `faulted` for an existing aggregate. A creation rejected
before an aggregate exists returns `status: rejected` and `state: null` . Admission rejection or an
unhandled envelope preserves the prior aggregate status.

Every named result field is present. `emissions` contains full external intents plus
`internal_mailbox` or `internal_disposed` references only; an internal envelope's deliverable copy
exists solely in its target mailbox or is accounted by the referenced lifecycle disposition.
`rejection` is null except on pre-step rejection, where it is exactly `{ code: rejection_code }` .
`fault` is:

-  the aggregate root's committed fault record when the aggregate root is faulted;
-  otherwise the target runtime fault newly committed by a `faulted` step; or
-  null.

A contained fault committed inside an otherwise successful owner RTC appears only in the
contained-runtime state and its reserved failure emission; it does not populate the top-level
`fault` . There is no plural `faults` result field.

Every `create` and `step` result also contains `lifecycle_dispositions` , including the empty list;
rejected creation returns the empty list. One entry contains exactly `event_id` , `request_digest` ,
`acceptance_sequence` , `final_queue_sequence` , `target_runtime_id` , and lifecycle `reason` . It
accounts for every ready/deferred entry removed by successful lifecycle cleanup, including internal
work emitted earlier in the same RTC. Entries use lifecycle cleanup runtime order, then ready
entries followed by deferred entries in queue order. The closed structural schema for `step` is
`schema/core-step-result-v1.schema.json` ; semantic status/disposition/fault/rejection relationships
remain mandatory. The creation result retains the creation-specific status/state shape above.

Creation rejection codes are exactly `invalid_creation_request` , `invalid_machine_target` , and
`invalid_binding` . Processing rejection codes are exactly `invalid_event` , `invalid_payload` ,
`invalid_correlation` , `invalid_instance_target` , `inactive_component_target` ,
`invalid_prior_state` , and `incompatible_bundle` . The closed `step` pre-step rejection subset is
exactly `invalid_instance_target` , `inactive_component_target` , `invalid_prior_state` , and
`incompatible_bundle` ; admission has the larger closed set in §17.4. Bundle parsing, schema, and
semantic load failures happen before these calls and use §2/§5 codes. A rejection commits no fault
record.

`disposition` is exactly:

-  `handled` — the accepted envelope completed a successful RTC step;
-  `deferred` — no handler at the resolving state level was enabled and that state
  deferred the event;
-  `unhandled` — no enabled handler or active deferral declaration existed;
-  `not_runnable` — a mailbox `step` named a running runtime with an empty ready mailbox;
-  `rejected` — validation failed before an RTC step; or
-  `faulted` — an engine fault occurred during the RTC step.

Deferral ownership is total: mailbox `step` moves a deferred event from the selected ready mailbox
to that runtime's deferred mailbox; structural recall moves an eligible deferred event once to that
runtime's ready tail and leaves an ineligible event in place.

The root ownership aggregate contains:

-  the root runtime;
-  every retained component runtime;
-  every non-disposed owned spawned descendant, including running, retained-faulted, or
  root-frozen diagnostic descendants;
-  each runtime's definition identity, configuration, variables, history, lifecycle
  status, and fault records;
-  ownership and `bind_to` lifetime-holder associations, plus placement, activation,
  state-entry, spawn, logical-step, and output identity counters; and
-  for the queue-bearing abstract aggregate, every runtime's isolated ready and deferred
  mailboxes plus aggregate acceptance and queue-placement counters; and
-  the `validated_bundle_fingerprint` defined below.

It contains no external broker backlog, dead-letter collection, timer, broker acknowledgement token,
credential, transport receipt, or plugin configuration. The sole aggregate-state artifact always
represents these mailboxes and counters, including when every mailbox is empty.

Before creation, resolution produces one normalized **executable** bundle tree. For
CEL/structured-only source this is an identity transform apart from the defaults below. For
native/custom source, explicit §5.4 resolution first expands locked bindings and compiles source
slots. The same default table also normalizes authored source for the generated lock's source digest
without expanding its slots. Default materialization is closed and context-sensitive:

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

Before this table is applied, every typed literal is normalized under §5.2. Thus an integer-form
`init` or payload `default` for a declared `float` becomes a binary64 value in the normalized tree;
destination-free numeric values such as `meta` leaves retain their parsed `int` /`double`
distinction.

An `instance_reference` must explicitly declare `nullable: true` ; non-reference variables cannot
declare `nullable` , so no normalized `nullable: false` is inserted. Initial transitions and choice
branches have no `lang` member and use their fixed language semantics without inserting one. An
explicitly present value is retained after §5.2 numeric normalization.

Every other optional member remains absent. In particular, absent `events` , `variables` , `payload`
, `entry` , `exit` , `on_events` , `states` , `components` , `meta` , binding, correlation, action,
guard, `init` , payload `default` , `local` , and `correlates_to` members are not replaced with
empty maps, empty lists, null, or false. Choice pseudostates do not receive `type` . The normalizer
recursively visits inline component roots and every structured action location. JSON Schema
`default` annotations are informative only; this table is the normative algorithm.

The engine then computes:

```text
validated_bundle_fingerprint = hash([
  "determa-validated-bundle-fingerprint-1",
  typed_bundle_tree
])
```

`typed_bundle_tree` recursively encodes null as `["null"]` , Boolean as `["boolean", value]` ,
string as `["string", value]` , integer as `["integer", canonical_decimal(value)]` , binary64 as
`["float", sixteen_lowercase_hex_bits]` , list as `["list", encoded_items]` , and map as
`["map", [[key, encoded_value], ...]]` with entries sorted by key UTF-8 bytes. Binary64 bits use
network byte order after negative-zero normalization. This typed projection prevents JCS from
collapsing `int` /`double` or rounding a signed-64-bit integer. It includes `meta` and every other
validated field.

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

Creation stores this fingerprint. Every ordinary admission or step, plus read-only inspection, first
validates the abstract prior-state shape and all retained definition/path references, then compares
its stored fingerprint with the supplied validated bundle. Malformed or internally inconsistent
prior state rejects with `invalid_prior_state` ; a fingerprint mismatch rejects with
`incompatible_bundle` . Both return the exact prior state with no counter/state/emission change.
This check precedes envelope validation. A different document reusing the same
namespace/machine/version triple can therefore never reinterpret existing state.

The portable aggregate-state and explicit definition-migration operations in §16 are separate
operations around this processing boundary. A host may decode an aggregate under its exact source
definition and explicitly migrate it before admission or a step. Ordinary processing never chooses,
discovers, or applies a migration.

The host may store one aggregate in one row/document or normalize it, provided every operation sees
serializable prior state and commits an observably equivalent result. The core itself performs no
persistence.

Creation initializes the aggregate and every root initial/entry action atomically. Creation bindings
contain separate `input` and `external` maps. Missing required, extra, or wrongly typed values
reject creation with `invalid_binding` , no state, and no emissions. Omitted declarations use `init`
only where §4.5 permits a default.

A value-dependent root initialization fault rolls author behavior back and commits a terminal
diagnostic aggregate containing only the validated-bundle fingerprint, root
runtime/definition/creation identity, aggregate counters, `faulted` status, and fault record. Its
configuration, variables, history, components, and owned spawned instances are empty. Root-local
spawn/state/component activation counters are reset to their pre-author-initialization values: spawn
sequence zero and empty state/component counter maps. Creation used logical step zero, so
`next_logical_step_sequence` is one; `next_output_sequence` is zero because every author output
rolled back. The result contains no emission and null rejection. Contained component and spawned
initialization faults follow the mandatory isolated behavior in §10.2; they are not implementation
choices.

## 9. Deterministic identities and emissions

All identity hashes use:

```text
"sha256:" + lowercase_hex(SHA-256(UTF-8(JCS(value))))
```

where JCS is RFC 8785 canonical JSON.

Logical counters are unbounded non-negative mathematical integers. Hash operands encode every
counter as a canonical decimal JSON string: `0` , or a non-zero digit followed by digits, with no
sign or leading zero.

`root_instance_id` and `creation_id` are non-empty strings supplied by the caller.
`root_instance_id` identifies the aggregate. `creation_id` identifies one logical creation request
and MUST be reused when retrying that request.

The root runtime identity is:

```text
hash([
  "determa-root-runtime-identity-1",
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
["determa-root-runtime-identity-1","1","sha256:7e48ad82ea5305c24b7730f4fd24c36ec196a0875c982b85eba5b3a5ddcbb92f","example.turnstile","turnstile","1","turnstile-42"]

hash:
sha256:d42f331cbd0c491bba66d512c010f89f94c52df12057a80fecde85d3b954592f
```

Including the validated bundle fingerprint prevents a changed same-version definition from reusing a
prior root runtime or effect identity. Component and spawned identities include an owner runtime
identity, and every effect includes its emitting runtime identity, so the distinction propagates
through the complete ownership tree and outbox.

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

For a `machine_id` placement, the final three operands are the referenced machine's definition
identity. For an inline `root` , they are the containing machine's definition identity and
`component_definition_pointer` is the inline placement's full RFC 6901 pointer. The pointer and
`owner_runtime_id` distinguish nested and sibling inline placements; `activation_sequence`
distinguishes re-entry incarnations.

The first inline placement in the same normative bundle above is reached during root initialization.
With activation sequence zero, its normative identity vector is:

```text
JCS:
["determa-component-runtime-identity-1","1","turnstile-42","sha256:d42f331cbd0c491bba66d512c010f89f94c52df12057a80fecde85d3b954592f","/machines/0/root/states/locked/components/0","0","example.turnstile","turnstile","1"]

hash:
sha256:b0145e3c9c3d470fda4e59e55fe1585a1a9a7d74e29377062d70bf1789ffeb76
```

A spawned `instance_id` , which is also its runtime identity, is:

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

Root creation is logical step sequence `0` . After successful creation, aggregate
`next_logical_step_sequence` is `1` ; `next_output_sequence` is the number of external intents
emitted during creation. Every new runtime initializes `next_spawn_sequence` , every component
placement initializes `next_activation_sequence` , and every runtime/state-path initializes
`next_state_activation_sequence` to `0` .

Allocation always takes the current counter value and increments the counter before using that value
in the same atomic step:

-  entering a state allocates its state activation sequence;
-  creating a component placement allocates its activation sequence;
-  executing `spawn` allocates the owner's spawn sequence; and
-  appending an external output intent allocates the aggregate output sequence.

Rollback restores every tentative allocation. An accepted handled/faulting envelope allocates the
current `next_logical_step_sequence` ; rejection and unhandled delivery allocate none. All author
behavior, initialization, lifecycle work, ownership changes, and emissions in that RTC use the same
step sequence. Fault finalization uses the reserved value and advances it exactly once.

Every executing behavior has a deterministic cause:

-  a delivered envelope's cause id is its `event_id` ;
-  root initialization derives the cause below with source and target equal to the root
  runtime, parent provenance `creation_id` , the machine root pointer, and ordinal `0` ;
-  component initialization uses the owner as source, component as target, the current
  cause as parent provenance, the component placement pointer, and its declaration
  index as ordinal; and
-  spawned initialization uses the owner as source, child as target, the current cause
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

`cause_kind` is exactly `root_initialization` , `component_initialization` , or
`spawned_initialization` . `source_locator` is the RFC 6901 pointer specified above. Thus
entry/initial actions can produce deterministic emissions even though their CEL environment has no
`event` binding.

Using the root vector above with `creation_id = "create-7"` , the normative root initialization
vector is:

```text
JCS:
["determa-cause-identity-1","1","root_initialization","turnstile-42","sha256:d42f331cbd0c491bba66d512c010f89f94c52df12057a80fecde85d3b954592f","sha256:d42f331cbd0c491bba66d512c010f89f94c52df12057a80fecde85d3b954592f","create-7","0","/machines/0/root","0"]

hash:
sha256:a1ba8e26366fdefe45f62e34e2856d0bebbf22b9b73fad742ebf5294421ae91c
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

Its immutable envelope `event_id` is exactly that derived value; there is no second internal cause
identifier. `emission_ordinal` is zero-based within the executing action and follows declared target
order. An author send's `emission_locator` is its RFC 6901 action pointer. Engine lifecycle
emissions use an `emission_locator` exactly equal to one of `system:component_completion` ,
`system:spawned_completion` , `system:component_failure` , or `system:spawned_failure` ; the ordinal
is zero unless that lifecycle operation emits multiple envelopes, in which case it is their
specified order. Distinct fan-out targets therefore have distinct event ids. For native action
proposals, the containing action slot is the executing action; §5.4 defines its locator and the
continuing internal-envelope ordinal across send proposals. It is not reset for each proposal.

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

For effects emitted by an inline component, the definition triple is the containing machine identity
above. `emitting_runtime_id` already contains the full placement pointer and activation sequence, so
identical actions in sibling, nested, or later inline placements cannot collide.

`emission_index` is the zero-based external-intent ordinal within that executing action. For a
native slot, that index continues across all external send proposals under §5.4; it is not reset for
each proposal. The intent also carries the allocated aggregate-monotonic output `sequence` , event
name, typed payload, and correlation id. Retrying the same uncommitted prior state and envelope
reproduces the same state, emissions, ids, and order. Processing several envelopes by repeated
`step` calls produces the same result as a host convenience API that applies that same ordered
envelope sequence atomically.

The queue or external adapter may use these identities for deduplication, but the core does not
require a delivery guarantee.

## 10. Faults and envelope disposition

### 10.1 Engine faults

Engine faults include:

-  `guard_fault` ;
-  `action_fault` ;
-  `invalid_instance_target` ;
-  `inactive_component_target` ;
-  `binding_not_empty` ;
-  `deferred_event_capacity_exceeded` ;
-  `contained_runtime_fault` ;
-  `invariant_fault` ; and
-  `cascade_fault` .

There is no runtime `type_fault` ; statically checkable types are load-time validation.
`invalid_instance_target` and `inactive_component_target` may also be pre-step rejection codes. They
are engine faults only when an already accepted RTC action attempts an invalid send target.

A `cancel` action whose expression evaluates to null, or does not currently address a running or
retained-faulted directly or transitively owned instance, is the successful no-op defined by §7.2
and is never `invalid_instance_target` .

On an engine fault, the core:

1.  rolls the RTC step back to its exact pre-step aggregate state, including variables,
   configuration, history, ownership changes, and tentative emissions;
2.  commits one deterministic fault finalization using the reserved step sequence;
3.  records the fault and marks the executing runtime faulted; and
4.  returns no emission from the rolled-back author actions.

When the executing runtime is the aggregate root, that fault finalization is terminal for the entire
ownership aggregate. The engine preserves the rolled-back diagnostic tree exactly: retained
component and spawned descendants keep their pre-step configuration, variables, individual status,
and fault records, but are frozen and non-targetable. It runs no descendant exit/cancellation
behavior and emits no contained failure notification for this freeze. The aggregate status is
`faulted` ; all later processing and read-only behavior follows the aggregate-terminal rule in §6.1.

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

`source_locator` uses a closed vocabulary. A value beginning `/` is the exact RFC 6901 pointer into
the validated bundle; a value beginning `system:` is one of the fixed locators below. Engines MUST
use this mapping:

| fault code | exact `source_locator` |
|---|---|
| `guard_fault` | pointer to the failing event-transition or choice `guard` value |
| `action_fault` | pointer to the failing CEL expression value inside the action, entry, exit, initial transition, choice branch, component binding, or spawn binding; for a provider action failure, the action-slot value; for an absent `refresh.only` field, the pointer to the first absent list item |
| `invalid_instance_target` | pointer to the executing send action's `to`/`targets` member or, for a dynamic instance expression, that exact expression value |
| `inactive_component_target` | pointer to the executing send action's `to`/`targets` member that names the component |
| `binding_not_empty` | pointer to the executing spawn action's `bind_to` value |
| `deferred_event_capacity_exceeded` | exactly `system:deferred_event_capacity` |
| `contained_runtime_fault` | exactly `system:unhandled_contained_failure` |
| `cascade_fault` | exactly `system:cascade_cleanup` |
| `invariant_fault` | exactly `system:invariant` |

For a `targets` list, the pointer includes the failing zero-based list index. A contained failure
notification embeds the §4.4 public projection of the child fault record; if its delivery later
faults the owner, the owner receives a separate retained record with the system locator above.
Rejected pre-step envelopes and rejected creation have no committed fault record and therefore no
`source_locator` .

For deferred-capacity overflow, selection reserves the current aggregate
`next_logical_step_sequence` before classification. The failed deferred append and any tentative
queue-sequence allocation roll back completely. Fault finalization consumes the selected ready
entry, uses its immutable envelope `cause_id` , records code `deferred_event_capacity_exceeded` ,
the reserved step sequence, and locator `system:deferred_event_capacity` , then advances
`next_logical_step_sequence` exactly once. `next_queue_sequence` is unchanged because no new queue
placement committed. No author action or external intent ran. The checkpoint host creates the
selected event's terminal `faulted` receipt in that same commit; root or contained-fault propagation
then follows the ordinary rules.

For a previously accepted mailbox entry, fault finalization removes that entry and creates its
terminal `faulted` receipt in the same commit. Other ready/deferred entries remain in their exact
locations; a root fault freezes them, while a contained fault freezes only that runtime subtree as
§6.7 defines. A transport plugin cannot retry or discard an engine-owned mailbox entry
independently.

### 10.2 Contained runtime faults

A component or spawned-runtime fault freezes that runtime and retained descendants, then returns one
deterministic internal failure emission to the immediate owner:

-  `determa.component_failed` with:

  `` `text
  {
    component_id: identifier,
    component_runtime_id: non_empty_string,
    fault: public_fault_record
  }
  `` `

-  `determa.spawned_instance_failed` with:

  `` `text
  {
    instance: instance_reference,
    instance_id: non_empty_string,
    machine_id: identifier,
    machine_version: positive_integer,
    fault: public_fault_record
  }
  `` `

The `fault` field is the fixed `public_fault_record` projection from §4.4. It copies `runtime_id` ,
`cause_id` , `code` , and `source_locator` unchanged and encodes the retained mathematical
`step_sequence` as its `canonical_decimal` string. Each failed runtime emits its notification once.
Source is the failed runtime; target is its immediate owner; the emission retains the faulting cause
and uses the corresponding system locator from §9.

Initialization faults are isolated contained-runtime faults with mandatory behavior:

-  A component whose initialization faults rolls back only its tentative author
  initialization state and emissions, commits the diagnostic projection below as a
  retained-faulted placement, and contributes exactly one
  `determa.component_failed` emission. Later component placements continue
  initialization in declaration order.
-  A spawned child whose initialization faults rolls back only its tentative author
  initialization state and emissions, commits the diagnostic projection below as a
  retained-faulted child, leaves `bind_to` set to that nominal child reference, and
  contributes exactly one `determa.spawned_instance_failed` emission. Later actions in
  the owner's ordered action list continue.

The retained diagnostic projection is exact: runtime and definition identity, component-placement or
spawned ownership/lifetime-holder identity, allocated component activation or owner spawn sequence,
`faulted` status, and the committed fault record. Its configuration, variables, history, components,
owned spawned descendants, and author emissions are empty. Supplied root input/external values and
root `init` values are not retained. Its child-local next spawn sequence is zero and its
state/component activation-counter maps are empty; no author-initialization allocation survives. The
containing owner's already allocated placement activation or spawn counter remains consumed, and a
spawned parent's `bind_to` value remains set as specified above.

The failure emission occupies the point at which that contained initialization faults: component
failures follow placement declaration order; spawned failures follow their spawn action's position
relative to other owner emissions. Earlier tentative owner emissions targeting a now-faulted pending
component remain in the result as specified by §7.1. No author emission from the failed
initialization survives.

All of these retained faults, bindings, counters, later component/action work, and failure emissions
remain tentative until the enclosing owner RTC commits. If any later work faults that owner step,
the owner rollback removes the newly created contained runtimes, bindings, contained fault records,
and failure emissions and restores every counter. If the owner step commits, the owner remains
running and the retained-faulted child is inspectable and cleanup-cancellable but cannot process
ordinary delivery.

A reserved failure event that reaches its owner unhandled faults the owner with
`contained_runtime_fault` ; its source locator is exactly `system:unhandled_contained_failure` and
its cause id is the failure envelope's `event_id` . A committed notification is an aggregate-owned
internal mailbox entry. A transport cannot discard, delay, reorder or expire it independently of the
owner's normal mailbox processing and explicit lifecycle disposition.

The immediate owner may cancel a retained-faulted spawned child for cleanup. Ordinary input cannot
advance a faulted runtime.

### 10.3 Domain failures

Application outcomes such as `payment_rejected` , `email_failed` , or `schedule_rejected` are
ordinary declared events. They do not become engine faults unless their own handling violates the
engine contract.

### 10.4 Dead letters are not core state

The specification defines no `dead_letter` , `dead_letters` , or `dead_letter_policy` field and no
dead-letter storage shape.

A transport or audit plugin may select an explicit pre-admission or post-terminal policy to:

-  deliberately discard an item while returning its observable disposition;
-  retain complete envelopes and fault metadata;
-  retain metadata without payloads;
-  retry before retention;
-  forward to a broker-native dead-letter facility; or
-  expose any other explicitly configured policy.

**No profile permits silent event deletion.** A deliberate policy discard returns the closed
`schema/event-disposition-v1.schema.json` record: exactly `event_id` , `source_identity` , `policy`
, `decision: "discarded"` , and `reason_code` . At least one identity is non-null; an event known to
Determa includes its exact event ID. A source identity is the adapter's immutable nonsecret item
identifier within the caller's selected source binding. `policy` identifies the explicitly selected
host policy; `reason_code` identifies its concrete decision. Missing identity, policy or reason is
not a valid discard. A non-durable host returns the record to the caller even when it retains
nothing. A durable profile additionally retains and links the exact evidence required by its
contract. §21 keeps its stronger source ownership, acknowledgements, terminal records and retention
requirements.

Engine-owned ready/deferred entries and committed internal failure notifications cannot be discarded
by a transport. Declared helper behavior consumes inputs through normal processing; deliberate
lifecycle removal returns §8 `lifecycle_dispositions` . Neither is an implicit transport exception.
Uncommitted rollback is not deletion of accepted input: committed mailbox and fault/disposition
rules remain authoritative.

Beyond the closed discard record, policy configuration, storage shape, privacy and operational
guarantees belong to that plugin. Under §21, an unhandled, faulted, or disposed event has an
explicit terminal decision and retained evidence. These policies apply only after terminal machine
disposition or before Determa acceptance; they cannot replace, reorder, expire, or cap a runtime's
normative ready/deferred mailboxes.

## 11. Plugins and hosting

This section defines the core boundary. The optional portable execution-checkpoint hosting contract,
adapter registration behavior, and durability capabilities are defined in §17. Neither section
defines a cross-language plugin ABI.

### 11.1 Transport queue plugins

A transport queue plugin owns external backlog until a host commits admission into a Determa
aggregate. It may later receive external output intents. The core does not standardize a concrete
plugin API, but acceptance must preserve each envelope's immutable value and identity and transfer
ownership exactly once.

Plugins may differ in:

-  ordering and fan-out;
-  in-memory or durable storage;
-  delivery attempts and acknowledgements;
-  duplicate delivery and deduplication;
-  retry, delay, and dead-letter behavior outside the accepted machine mailbox;
-  transactional integration with aggregate persistence; and
-  capacity, overflow, backpressure, and availability.

Core determinism means that the same valid prior state and same accepted sequence produce the same
mailbox state and result. Different transport plugins may produce different admission traces, but
after acceptance they cannot alter §6.7 deferral, recall, ordering, capacity, or disposition
semantics.

The normative ownership, acknowledgement, retry, ordering, and durable ingress dead-letter rules for
a lossless delivery profile are in §21. A transport's ability to deliver or acknowledge is not an
execution-store capability.

### 11.2 Timer extensions

Time is modeled through external event-producing extensions, never through core clock state. A
machine may emit a declared scheduling request and later receive a declared, correlated elapsed,
rejected, failed, or cancelled event.

Without the optional §23 helper profile, the timer extension is a black box. It need not know which
state or business process uses the event. Its request payload, cancellation behavior, delivery
reliability, duplicate policy, clock source, persistence, and credentials are extension concerns. An
installed §23 helper follows that profile's closed contract.

Different timer extensions may provide best-effort in-memory behavior, durable at-least-once
delivery, database integration, or real-time-oriented scheduling. Core Determa provides no guarantee
that a scheduled event arrives, arrives once, arrives in order, or arrives near its requested time.

Late or duplicate elapsed events are ordinary input. Machines and queue plugins handle them through
correlation, explicit state behavior, or plugin policy.

### 11.3 External effects

All remote I/O follows the same boundary:

1.  a successful RTC returns a deterministic external intent;
2.  the host persists/delivers it according to its plugin guarantees;
3.  the external system eventually may produce a declared correlated input; and
4.  that envelope is processed in a later independent RTC step.

No external success is inferred merely because an intent was emitted.

### 11.4 Hosting profiles

Informative examples of valid hosts include:

-  foreground request/response processing with an in-memory queue;
-  one-row database persistence with transactional inbox/outbox plugins;
-  a durable background worker;
-  a broker-backed distributed host;
-  an embedded main-thread loop; and
-  an MCP adapter exposing declared public inputs as tools.

Host store, transport, endpoint, scope and credential configuration never appear in machine grammar.
The short §5.4 source names select explicitly resolved language or runtime slots, not deployment
authority or transport configuration.

### 11.5 Public extension identity, registration, and capabilities

This section is the common public host boundary for embedded applications and future hosted
implementations. It defines the shape and meaning of extension discovery and capability negotiation,
not a plugin ABI or a mandatory service. The core remains a foreground transformation (§8). A host
MAY inject an object directly; no URI, registry, daemon, clock, coordinator, or network endpoint is
needed to evaluate the pure core. If a host offers named registration, both bundled and third-party
extensions MUST use the same public `register` operation and lookup rules. A named extension's
category is exactly one of `execution_store` , `projection` , `transport` , `timer` , `http` ,
`native_handler` , `compiler` , `runtime_provider` , `resolver` , `authority` , and
`archive_participant` . These categories are distinct: a store claim does not grant authority, and a
transport or timer claim does not imply a durable host profile. The public registration path exposes
`register` , `validate_configuration` , `capabilities` , and `health` ; category-specific setup is
separate. Registration validates a descriptor before making a factory visible. Configuration
validation precedes opening an instance and evaluating its claims. Direct injection supplies the
same descriptor and validation operations, even when no name resolver is installed.

The closed provider reference is defined by `schema/provider-reference-v1.schema.json` : exactly
`identifier` , `version` , and `content_digest` . The identifier matches `[a-z][a-z0-9.-]*` ; the
version is an exact SemVer (no range, wildcard, alias, or `latest` ); the digest is a canonical
SHA-256 content digest. All three compare exactly. The digest binds the executable provider and its
declared dependency closure according to that provider's separate contract; the descriptor alone is
not proof that arbitrary installed code matches the digest. The host MUST verify the actual loaded
provider and its trusted allowlist before executing it or claiming a guarantee. Dynamic discovery is
explicit or allowlisted; untrusted source text, a machine document, or an unauthenticated event MUST
NOT cause provider installation or selection. A scheme or URI is a lookup hint only. It grants no
privilege, scope authority, credential, capability, or override right. Each host authorizes
registration, discovery, and invocation against the authenticated scope and operation; possession of
a provider reference is not authorization.

`schema/extension-descriptor-v1.schema.json` defines a registration descriptor with exactly
`category` , `provider_reference` , `interface_version` , and `supported_capabilities` .
`interface_version` is the integer `1` for this boundary. The capability names are closed and
category-specific in that schema. A descriptor lists only capabilities the provider knows how to
evaluate; it does not assert that every configured instance has them. A host registers at most one
factory for each `(category, identifier, version)` ; duplicate registration, including a conflicting
digest, fails as `duplicate_extension_registration` and does not replace the first. For direct
injection, the host checks the supplied object's descriptor and exact reference through the same
validation path. Missing reference yields `unknown_extension` ; mismatched version or digest is
`extension_identity_mismatch` . Malformed descriptors or references yield
`invalid_extension_descriptor` ; invalid host-owned configuration yields
`invalid_extension_configuration` . No fallback to another installed version or bundled provider is
permitted.

After validating host-owned configuration, the host calls the registered provider's
`capabilities(configured_instance)` and `health(configured_instance)` operations.
`schema/extension-capability-report-v1.schema.json` defines the resulting public report: exact
category and provider reference, a nonsecret configured `instance_id` , `health` (`healthy`,
`degraded` , `unavailable` , or `unknown` ), and closed `claims` . Every reported claim MUST be
listed by the matching descriptor's `supported_capabilities` ; a report with a different category,
reference, or instance is never evidence for this instance. The report is scoped to that exact
configured instance and current health. The host MUST verify claims against its policy and actual
topology; self-assertion is not proof. `degraded` , `unavailable` , or `unknown` health cannot
satisfy a requirement unless the capability's separate contract explicitly proves safe operation at
that health, which no common capability here does. A missing or unproved claim is false. This
false-by-absence rule applies to requested **guarantees**. It does not assert the absence of a
hazard: a healthy runtime-provider report that omits `external_io_capable` still leaves external I/O
possible unless the exact provider and host policy independently prove it cannot occur (for example,
through a verified `pure` guarantee). The host MUST treat unresolved I/O status as possible I/O.
Provider families, URI schemes, installed packages, and a previously healthy report do not inherit
capabilities. A changed configuration or relevant health requires reevaluation before a newly
requested operation. The schema's empty capability sets for `http` , `compiler` , and `resolver`
mean this common foundation assigns them no standalone standard guarantee yet. Their separate
contracts may add versioned category-specific capabilities; implementations MUST NOT invent a
meaning under this version-1 vocabulary. The execution-store names have the meanings in §17.11. The
remaining nonempty category sets reserve names for the corresponding projection, transport, timer,
native-handler, runtime-provider, authority, and archive-participant contracts. A host MUST NOT
advertise one of these reserved names until its defining public contract and applicable conformance
cases exist and the configured instance passes them. In particular, `authoritative_scope_fencing`
means the configured authority rejects stale scope writers under its proved epoch and ownership
boundary. It is not inferred from a durable store, a URI, a process lock, or a readable archive.
`safe_relocation` is a distinct claim for the exact source, destination, and authority topology,
supported only when the separate transfer contract proves old-owner retirement and destination
activation. Local guarded writes or a successful export do not imply it. A host MUST check the
actual operation's source and destination against that proved topology; a configured-instance report
alone does not authorize a particular transfer. The host MUST report `safe_relocation` unavailable
for an unproved topology, refuse relocation before activation, and leave any staged import inactive;
it MUST NOT silently invoke the weaker standalone takeover. Multi-host coordination and managed
control-plane operations are outside this common foundation.

`schema/extension-capability-requirement-v1.schema.json` defines one exact requirement: `category` ,
`provider_reference` , `instance_id` , and `capability` . A host profile resolves all its required
extensions and checks every requirement against healthy, verified configured reports **before**
loading or creating a root, admitting an envelope, dispatching an intent, or changing host evidence.
An unknown provider, invalid configuration, unhealthy instance, or unsatisfied requirement fails
closed without mutation. Hosts MAY use category-specific codes such as §17.10's
`adapter_capability_mismatch` ; otherwise they report `extension_capability_mismatch` . Requested
profile guarantees never silently downgrade. The positive and negative vectors in
`vectors/extensions/capability-cases-v1.json` are normative for these common checks.
Category-specific operations and stronger guarantees are defined in their respective sections and
conformance profiles, not inferred from the registry.

For a composition, a guarantee such as `pure` , `deterministic` , `portable` ,
`semantically_introspectable` , or `process_contained` is effective only when **every**
participating provider and the host policy prove it. `external_io_capable` is a hazard flag: it is
effective when **any** participant may perform external I/O, including an unknown or unverified
participant. The absence of an `external_io_capable` claim is not a no-I/O attestation. Unknown I/O
therefore requires explicit weak profile opt-in and prevents automatic retry or replay claims based
on purity. Advertised `external_io_capable` grants no transactional safety. Runtime provider claims
have the exact meanings in their dedicated provider contract; this section only fixes truthful
composition and refusal semantics.

Endpoint URLs, named endpoint/scope aliases, authentication credentials, provider configuration, and
secret material are deployment configuration, never machine semantics or members of a portable
state, checkpoint, receipt, or archive. A client MAY configure multiple named endpoints and scopes;
changing a scope's endpoint never rewrites its statechart or proves ownership transfer. Local hosts
and a future Determa SaaS MUST use the same public machine model, protocol, portable Determa-owned
artifacts, capability names, and conformance tests for each capability they claim. Import must
validate exact definitions, state, queues, receipts, unresolved intents, provenance, and required
capabilities. External helper state moves only through a declared participant/export contract;
unsupported required helper state must be reported, not silently lost. This compatibility boundary
does not require or authorize a distributed coordinator or private SaaS infrastructure in the
open-source core.

Changes to public protocols, portable artifacts, capability meanings, archive/import behavior,
effect identity, or helper boundaries require hosted-compatibility review. The release conformance
gate MUST track fingerprints for these contracts, require an explicit change record and updated
positive and negative vectors for changed boundary files, and run cross-language interoperability
checks where the capability is claimed. A future SaaS implementation MUST run the same public
conformance suites for all its claimed capabilities. Exact re-execution equality applies only to
deterministic portable profiles; weak or nondeterministic profiles check canonical artifacts,
identity, retained evidence, and safety refusals instead.

## 12. Inspection and visualization

Implementations SHOULD expose read-only inspection of:

-  aggregate/root and runtime identities;
-  active configurations and variables;
-  component and ownership relationships;
-  status and fault records;
-  history; and
-  deterministic emissions returned by the last call.

Runtime-local ready/deferred mailbox contents and their portable ordering identities are core state
and SHOULD be inspectable. External broker backlog, delivery attempts, dead letters, scheduled jobs,
and broker acknowledgements remain plugin-owned.

### 12.1 Exact candidate inspection

`inspect_candidate` is a read-only core operation over one validated executable definition, one
valid portable aggregate, one exact runtime incarnation, and one normalized candidate envelope. Its
closed request and outcome are defined by `schema/inspection-v1.schema.json` . The request has
exactly `mode` , `aggregate_state_digest` , `runtime_id` , `runtime_incarnation` , `envelope` , and
`limits` . `runtime_incarnation` is the runtime's exact portable `identity_origin` (§16.3),
including its component activation or spawned sequence where applicable. A caller cannot substitute
a matching machine or placement with a different incarnation. `structural` requires `limits: null` ;
`semantic` requires two positive canonical decimal limits, `maximum_guard_evaluations` and
`maximum_evaluation_steps` . The caller supplies a complete normalized envelope; inspection
validates it with the same declaration, direction, payload, target, and reserved-event rules as
admission, but never admits it. The envelope's exact target must identify the requested runtime
incarnation. The digest must match the supplied aggregate before inspection begins. Invalid request
shape or digest is an operation failure, `invalid_inspection_request` , with no classification.

A successful result has exactly `aggregate_state_digest` , `definition_fingerprint` , `runtime_id` ,
`runtime_incarnation` , `classification` , `possible_dispositions` , `disposition` , `reason` ,
`levels` , and `guard_evidence` . The fingerprint is the addressed runtime's exact current
executable definition, including provider closure when applicable. For an absent or mismatched
runtime, it is the aggregate root's exact current executable definition; this does not imply that an
absent target was resolved. `possible_dispositions` is a nonempty duplicate-free subset in canonical
order `handled_now` , `deferred` , `unhandled` , `invalid` . A `definitive` result has one
possibility and the same non-null `disposition` ; a `conditional` result has at least two
possibilities and `disposition: null` . `reason` is null except for `invalid` , when it is exactly
one of `target_not_found` , `target_incarnation_mismatch` , `invalid_envelope` , or
`runtime_inactive` . Missing runtime and wrong incarnation take precedence over envelope validation;
an inactive exact runtime takes precedence over envelope validation. Invalid envelope includes wrong
target, visibility, direction, payload, and reserved failure-event usage. Invalid results have no
levels or guard evidence. These classifications are predictions about dispatch, never admission
receipts or promises that actions will complete. A guard error is not an `invalid` disposition.

Each `levels` element has exactly `state_id` , `handler_branches` , and `defers` . Elements
enumerate the active ancestry deepest state first, ending at the runtime root. `state_id` is the
exact state-definition pointer; `defers` is the Boolean answer for this event at that level.
Branches preserve declaration order and have exactly `branch_index` , `guard_locator` , and
`guard_binding_digest` . The index is a zero-based canonical decimal string. An unguarded branch has
both guard fields null. A guarded branch's locator is an exact pointer into the validated executable
definition; its binding digest binds the normalized CEL guard or exact runtime provider reference
and source/dependency closure under that definition fingerprint. It is
`hash(["determa-guard-binding-1", definition_fingerprint, guard_locator, typed_guard_binding])`
under §9. `typed_guard_binding` applies the §8 typed-tree projection to the exact validated guard
member. For a CEL guard, this is `["string", source]` , where `source` is exactly the Unicode string
value of that `guard` member in the §8 normalized validated bundle tree. Parsing and type checking
do not rewrite, trim, pretty-print, case-fold, or otherwise canonicalize its source text; spaces and
line breaks are retained as code points. For a runtime-provider guard, `typed_guard_binding` is the
complete exact `{provider: binding}` guard member, including its source/dependency closure, encoded
as the §8 typed tree. Its object-member order is canonicalized by §8 and §9; exact string/source
values are preserved. Recompilation cannot substitute a different binding at the same locator. For
example, `"event.payload.amount > 1"` and `"event.payload.amount  > 1"` have distinct guard bindings
and distinct digests, even though both evaluate the same way for every numeric amount. Structural
inspection enumerates branches without invoking CEL, providers, actions, helper routes, or host
delivery. `guard_evidence` is empty.

Structural `possible_dispositions` is the union of every outcome reachable by assigning true/false
to each guarded branch while obeying §6.3 branch order and same-state deferral precedence. At a
level, any true branch or final unguarded default handles; only the all-false path tests same-state
deferral, then continues to the parent when no deferral exists. A child deferral blocks all
ancestors. Thus a guarded branch followed by an unguarded default is definitively `handled_now` ; a
guard-only handler with same-state deferral may be `handled_now` or `deferred` ; and the same
handler without deferral may be `handled_now` or the ancestor outcome. No reachable declaration
means definitively `unhandled` . Structural inspection does not predict guard faults, since it
executes no guards.

### 12.2 Optional safe semantic inspection

Semantic inspection follows §6.3 branch order and deferral precedence on the provided snapshot. It
evaluates only guard slots reached along that path. It never simulates actions, choices, entry/exit,
creation, cancellation, recall, or another RTC step. A successful semantic result is definitive and
has one Boolean evidence record per evaluated guard, in evaluation order. Each record has exactly
`state_id` , `branch_index` , `guard_locator` , `guard_binding_digest` , and `value` . Any guard
enumerated in the active ancestry whose exact resolved provider closure lacks a separately proved
bounded, nonmutating introspection entrypoint fails with `inspection_capability_unavailable`
**before any guard evaluation**. A host MUST NOT substitute an ordinary evaluator on a purity claim.
A guard evaluator failure returns `inspection_guard_failure` ; exhaustion returns
`inspection_limit_exceeded` . Neither returns a disposition or faults/mutates the supplied
aggregate. A failed operation has exactly `code` and `source_locator` ; the locator is null for
request/capability failures and the exact guard pointer for evaluation failures. The operation
returns no partial evidence on failure. Semantic inspection is an optional capability; a host that
does not advertise it returns `inspection_capability_unavailable` .

Portable CEL guards use the following shared abstract fuel, independently of a particular
interpreter's instruction count. Before evaluation, reject any guard whose UTF-8 source exceeds 4096
bytes, checked AST exceeds 1024 nodes, or input snapshot (envelope plus all visible variable values)
exceeds 65536 value units. One value unit is one scalar value, one Unicode scalar in a string, one
list slot, or one map entry plus its key's Unicode scalar count, summed recursively. The request
limits cannot exceed 64 guard evaluations or 1000000 evaluation steps. Larger limits are
`invalid_inspection_request` , rather than a host-dependent extension of this profile. The exact
pass/fail fuel boundaries are pinned in `vectors/inspection/fuel-boundaries-v1.json` . Preflight
failure is `inspection_limit_exceeded` with the first reached guard locator. All arithmetic below
uses unbounded nonnegative counters and charges before an operation; a charge crossing the remaining
budget fails immediately.

| Evaluated operation | Abstract step charge in addition to child expressions |
|---|---:|
| every evaluated AST node, including literal, identifier, selection, index, operator, call, or conditional | 1 |
| materialize a string/list/map literal | value units of the constructed value |
| field or map selection, `has`, map index, or map membership | 1 + Unicode scalars in the key + sum over inspected map entries of (1 + Unicode scalars in each key); typed record selection has no entry sum |
| list index or `size(list)` | 1 |
| list membership | value units of the complete list and candidate value |
| `size(map)` | value units of the complete map |
| `size(string)`, string comparison, string equality, or string-to-string conversion | Unicode scalar count of each string operand |
| string concatenation | sum of operand Unicode scalar counts and result count |
| list concatenation | sum of operand slot counts and result slot count |
| list/map equality or inequality | value units of both complete operands |
| scalar comparison/equality, numeric arithmetic/conversion, Boolean operation, negation, or null test | 1 |
| `string(bool|int|double)` conversion | Unicode scalar count of resulting canonical string |

Map inspection sums the complete map regardless of lookup success or host hash layout. Nested
collection equality uses the complete operand units once at that operator, with no recursive extra
charge. A failed operation incurs its listed charge before returning its error. `?:` evaluates only
its condition and selected arm. `&&` /`||` evaluate both operands in source order to preserve the §5
error absorption rule and deterministic fuel, even when the first Boolean alone could decide the
result. No comprehension, iteration, receiver call, or other intrinsic is admitted by §5.2; adding
one requires a corresponding exact fuel rule before semantic inspection can claim it. Guard-count
budget is charged once immediately before each reached guard. Exhaustion takes precedence over a
guard error at the operation whose charge cannot be paid. Native safe guard providers must expose a
public deterministic work schedule and honor the same request limits through their separately proved
entrypoint. Without that proved schedule their semantic inspection capability is unavailable.

Neither mode changes aggregate bytes, counters, queues, receipts, audit records, or provider state.
Inspection does not resolve a plugin route or read host backlog. The outcome belongs to the exact
supplied snapshot; a subsequent mutation requires a fresh inspection and digest. Hosts may
additionally expose presentation views, but MUST NOT label an implementation-specific enabled-event
list as this contract.

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

Mermaid renders entry/exit/action text but does not enforce Determa execution order. Exported
diagrams MUST therefore be treated as views of the normative bundle, not an alternative executable
definition.

## 13. Deliberately unsupported in format 1

The current alpha classifies the previously explored capability areas as follows. This is a
completeness boundary, not a compatibility promise for earlier drafts.

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
| definition migration/hot-swap | explicit portable aggregate migration under §16; never implicit in ordinary processing |
| observers and export | exact-candidate structural inspection is core; safe semantic inspection is optional under §12 |
| snapshots | closed portable aggregate-state envelope and package under §16 |
| stores and CLI protocols | host/implementation concerns, not bundle grammar |

The pre-release format deliberately omits:

-  native timers, clocks, `after` , sleeps, and time-triggered transitions;
-  external transport queues, retries, acknowledgement, or dead-letter storage;
-  orthogonal regions with implicit event broadcast;
-  shared mutable variables or shared queue state across runtimes;
-  direct host-to-component delivery;
-  cross-runtime transitions;
-  remote or detached core `spawn` ;
-  package imports and dependency/version resolution;
-  live definition replacement without the explicit §16 migration operation;
-  identity rekeying during migration;
-  destructive reset as a migration fallback;
-  arbitrary executable migration code or author behavior during migration;
-  standardized CLI/store JSON shapes;
-  root engine-fault recovery/reset;
-  reset or re-entry of the machine root by an ordinary transition;
-  distributed transactions, exactly-once delivery, or hard real-time guarantees;
-  local transitions whose target is not a strict descendant of their composite source;
-  external re-entry of a proper ancestor target; and
-  plugin discovery, installation, manifests, or standardized configuration fields.

These omissions are not reserved implementation hooks. A host may provide them only outside core
through events and plugins unless a later format revision defines otherwise.

## 14. Conformance plan

Format-1 conformance MUST be added before engine implementation or release. Cases should be one
behavior per fixture and include:

-  document/schema positives and negatives;
-  variable initialization/default requirements and creation/component/spawn binding
  rejection;
-  payload-default materialization and optional-field absence;
-  integer bounds and integer-to-binary64 normalization at every typed boundary;
-  strict parsed-value and Unicode-scalar validation plus portable numeric, Boolean,
  and null source-token resolution;
-  CEL profile name/type checking, event visibility, numeric faults, selected conditional
  branches, commutative error absorption for `&&` /`||`, and Unicode string ordering;
-  one-snapshot send-expression evaluation and deterministic payload/correlation/target
  fault precedence;
-  leaf-to-ancestor dispatch and false-guard fallback;
-  child-handler-over-parent-deferral, child-deferral-over-parent-handler, and same-state
  handler-versus-deferral precedence, including true, false, all-false, and faulting
  guards;
-  repeated deferral, selective FIFO recall after stable RTC, reclassification at the
  ready head, capacity overflow, and exact envelope-identity preservation;
-  internal, self, local, and external descendant-reset transition traces;
-  proper-ancestor transition bounds and the absence of external ancestor re-entry;
-  schema rejection of non-canonical internal/local transition shapes;
-  least-common-ancestor exit/entry paths;
-  transition-action-before-exit ordering;
-  load-time rejection of transition writes to destinations that the transition exits;
-  root-boundary preservation plus root self/history rejection;
-  root-local rejection and terminal aggregate-root fault behavior across descendants;
-  destroyed `refresh` rejection, missing-`refresh.only` rollback, and owned-child
  cancellation on reference scope exit;
-  initial descent and ordered choice;
-  compound event/initial/choice-chain actions resolved before one lifecycle transition;
-  explicit history resume/restart, self-history lifecycle replay, first-entry fallback,
  capture timing, shallow/deep restoration, local history targeting, and variable
  reinitialization;
-  immutable envelope validation and each disposition;
-  explicit owner-to-component `env` forwarding, typed component refresh, and rejection
  of every broader reserved-event send form;
-  no recursive delivery of internal sends;
-  exact immutable root/spawned/component targets and stale component-incarnation
  rejection;
-  component creation/routing/completion/disposal, including pending-initialization
  addressing and inline identity vectors;
-  spawn, nominal reference, completion, cancel and cascade, including no-op
  cancellation of null and disposed references;
-  completion and `stop` output ordering across author behavior, descendant cleanup,
  active exits, and reserved owner notifications;
-  isolated component/spawn initialization faults and enclosing-owner rollback;
-  root execution of `{owner: true}` , exit-time spawn rejection, and deterministic
  exit-time sends to disposed targets;
-  omitted `refresh.only` and absence of retained external-source maps;
-  coherent validated-bundle-fingerprint, root-runtime, initialization-cause,
  event/effect, and inline-component identity vectors across languages;
-  prior-state shape and supplied-bundle compatibility rejection, including changed
  metadata and same-version definitions;
-  full RTC rollback and contained-runtime failure propagation;
-  exact document/system fault locators;
-  completed runtime empty configuration/variables and retained terminal diagnostics;
-  runtime-local mailbox isolation across root, component, and spawned targets; and
-  explicit absence of timers, external transport queues, and dead-letter fields from
  core state.

Persistence and migration conformance additionally requires:

-  canonical aggregate encoding, decoding, digest verification, and byte-stable
  round trips, including exact no-trailing-newline RFC 8785 byte vectors;
-  strict rejection of unknown artifact fields, formats, schema versions, invalid
  typed values, invalid relations, and inconsistent source definitions;
-  exact root, four-field spawned-reference, and five-field component target shapes,
  with rejection of every extra/missing format-1 target member;
-  complete faulted-aggregate round trips including fault `step_sequence` and historical
  definition anchor;
-  content-addressed definition and descriptor resolution, including missing,
  untrusted, and hash-mismatched artifacts;
-  package attachment equivalence with the definition-registry contract;
-  unchanged-definition resume through encode/decode;
-  aggregate-shape-compatible migration with no logical-state transform;
-  exact per-descriptor `migration_applied` audit records and empty equal-fingerprint
  route no-op results;
-  total active-state, variable, history, component, owned-runtime, lifetime-holder,
  counter, and fault-anchor transforms;
-  repeated-runtime/activation transform scoping with no cross-runtime value mixing;
-  explicit deleted-state quarantine with no name, ancestor, initial, history, or reset
  guess;
-  exact pinned multi-hop routes, adjacency checks, and cycle/alternate-route rejection;
-  immutable runtime, target, nominal-reference, activation, spawn, logical-step, and
  output identities across migration;
-  queue-bearing aggregate/checkpoint round trips, event acceptance/terminal receipt
  separation, and ready/deferred migration totality;
-  retry-identical success or failure, complete rollback, and no counter consumption;
-  migration followed by handled, unhandled, rejected, and faulted dispatch in one
  host transaction;
-  terminal completed/faulted maintenance migration without reactivation or emission;
-  exact terminal maintenance/policy failures and definition/descriptor authorization
  failures; and
-  descriptor-declared and cumulative resource-limit accounting.

Execution-checkpoint profile conformance additionally requires:

-  exact checkpoint and envelope digests plus byte-stable canonical round trips;
-  zero-based mailbox counters, native receipt identities, and one digest domain;
-  same-RTC internal-send retention, frozen-target retention, successful lifecycle
  disposal, and rollback cases with exactly one mailbox/disposition result;
-  duplicate event ids within one batch, equal/conflicting terminal replay after root
  completion/fault/tombstone, event-identity tombstone compaction, and dependency-closed
  pruning;
-  exact deferred-capacity fault locator, allocation rollback, causal consumption, and
  root/contained finalization;
-  reusable queue migration disposal selectors, arbitrary migration reasons, reduced
  capacity totality, and retained-faulted mailbox preservation without recall;
-  closed-schema and semantic rejection for identity, root, ordering, counter, outcome,
  revision, and digest inconsistencies;
-  durable host-input acceptance, unified host/internal sequence ordering, pending
  same-content replay, pending/committed disjointness, and every unequal-content
  conflict;
-  every closed pre-acceptance failure, including replay-before-tombstone ordering and
  proof that no failed acceptance mutates or acknowledges;
-  creation, handled/unhandled/rejected/faulted delivery, applied/no-op maintenance, and
  tombstone idempotency receipts, including every otherwise-case;
-  explicit receipt-versus-§8-result boundaries and optional same-transaction
  application-response replay;
-  permanent/bounded retention, irreversible pruning history, terminal
  checkpoint/tombstone retention in both modes, dependency-closed pruning, restore
  completeness, and root-identity no-reuse;
-  rejection of physical checkpoint/root-marker deletion,
  including bounded mode and backup/restore;
-  exact accepted/committed revision equations for delayed, foreground, creation, and
  internally emitted deliveries, including every impossible ordering;
-  atomic aggregate/mailbox/receipt/outbox/audit/revision replacement at every
  injected pre-commit crash point;
-  post-commit/pre-acknowledgement replay without redispatch or duplicate insertion;
-  embedded foreground accept/process and delayed processing with equal committed
  results;
-  all pending outbox states (`not_attempted`, `retryable_failure` , `ambiguous` ) and all
  terminal outcomes (`confirmed`, `permanently_rejected` , `operator_cancelled` ,
  `discarded` , `dead_lettered` ), equal-state update replay, compact effect tombstones,
  and silent-deletion rejection;
-  one-winner concurrent revision updates without lost writes;
-  built-in and synthetic third-party registration through one public route;
-  registry-free direct injection and mandatory registry use for every offered
  scheme/identifier resolution;
-  deterministic unknown, duplicate, invalid-configuration, and capability-mismatch
  execution-store failures; and
-  truthful store-capability versus composed-host-profile negotiation, including
  durable-profile rejection of memory and rejection of store-only broker claims.

Quarantine storage and local immutable-cache mechanics remain profile-owned host details outside the
checkpoint artifact.

Golden-trace cases SHOULD make every action emit a trace token so ordering is directly reviewable.

## 15. Future example repositories

After conformance and engine support, a separate examples repository should validate:

-  local foreground processing;
-  database/ACID embedding with queue plugins;
-  hibernation and later host ingress;
-  isolated parallel components;
-  owned spawning and cancellation;
-  remote orchestration through effects;
-  best-effort and durable timer extensions;
-  broker retry/dead-letter policies;
-  package reuse after import semantics exist;
-  MCP exposure; and
-  real-time-oriented hosting;
-  portable aggregate-state round trips; and
-  one-row and normalized database persistence with lazy migration, transactional
  inbox/outbox/audit, rollback injection, and quarantine recovery.

Those examples are empirical design validation. They are not part of this specification-only change.

## 16. Portable persistence and definition migration

### 16.1 Independent artifact identities

Machine documents remain numeric `format: 1` . Portable persistence and archives use exactly these
five independent closed JSON artifacts:

| artifact | exact format field | exact schema-version field | schema |
|---|---|---|---|
| aggregate-state envelope | `aggregate_state_format: "determa.aggregate_state"` | `aggregate_state_schema_version: 1` | `schema/aggregate-state-v1.schema.json` |
| migration descriptor | `migration_descriptor_format: "determa.aggregate_migration"` | `migration_descriptor_schema_version: 1` | `schema/migration-descriptor-v1.schema.json` |
| transport package | `aggregate_state_package_format: "determa.aggregate_state_package"` | `aggregate_state_package_schema_version: 1` | `schema/aggregate-state-package-v1.schema.json` |
| execution checkpoint | `execution_checkpoint_format: "determa.execution_checkpoint"` | `execution_checkpoint_schema_version: 1` | `schema/execution-checkpoint-v1.schema.json` |
| portable archive | `archive_format: "determa.scope_archive"` | `archive_schema_version: 1` | `schema/archive-v1.schema.json` |

Queue-bearing core results use `schema/core-step-result-v1.schema.json` . Artifact schema version 1
is the sole supported portable artifact version. No alternate artifact representation, compatibility
wrapper, conversion path, or implicit conversion is defined. Machine document format 1 is a separate
version domain and is not an artifact schema version.

The final version-1 shape is identified by its exact closed schema and digest domain, not by its
version number alone. A prior pre-release artifact with the same numeric version but a different
shape MUST be rejected. Decoders MUST NOT accept old field sets, aliases, or hash domains under
version 1.

Artifact schema versions, machine format, repository/package SemVer, launcher SemVer, and
author-controlled machine `version` are independent version domains. Unknown artifact formats or
schema versions are rejected before semantic validation; there is no nearest-version parsing or
best-effort field retention.

Snapshots produced by releases 0.0.1 through 0.0.6 are not portable artifacts. The caller or host
MUST select one artifact decoder before decoding. Missing or unknown discriminators fail with the
decoder's exact closed format/version code; a decoder MUST NOT probe another artifact kind, infer
one from field shape, or attempt conversion.

Artifacts MUST be strict UTF-8 JSON and apply the §2 source-level duplicate-name, acyclic
JSON-value, Unicode-scalar, Boolean, null, and finite-number requirements before schema validation.
YAML is not a portable artifact encoding. Every schema is closed. Structural validity is necessary
but not sufficient: all ordering, cross-reference, digest, type, and totality invariants in this
section remain mandatory.

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

Language APIs may use idiomatic names. These are pure operations; they do not define a database API,
registry transport, queue, transaction manager, or bulk migration job.

### 16.2 Canonical values and aggregate encoding

Artifact-owned counters and machine versions use canonical decimal strings: `0` , or a non-zero
digit followed by zero or more digits. Signed typed integer values use `0` or an optional `-`
followed by a non-zero digit and zero or more digits. Bounds that are semantic rather than
structural are checked after schema validation.

`target_identity` embeds the exact normalized format-1 §6.1 mathematical target value but uses
artifact-owned decimal-string projections for its integer-valued members. A spawned target's
`machine_version` is a positive signed-64-bit canonical decimal string. A component target's
`activation_sequence` is an unbounded non-negative canonical decimal string. Numeric JSON forms are
invalid even when their values would be exactly representable. Decoding reconstructs the
mathematical integers before target equality, reference equality, routing, or dispatch; the
decimal-string projection is not a change to format-1 identity semantics. Origin, current relation,
and counter records use the same artifact-owned decimal strings and may carry the additional
migration data that is deliberately absent from the target.

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

Map entries are strictly increasing by key UTF-8 bytes and contain no duplicate key. Negative
binary64 zero is encoded as positive zero. Non-finite binary64 values are invalid. Lists preserve
order. This is the same value distinction used by the §8 validated-bundle fingerprint; a declared
`instance_reference` value is encoded through its exact §4.5 map projection and is retyped from its
declaration on decode.

All arrays representing sets or maps have one canonical order:

-  `runtimes` : `runtime_id` UTF-8 bytes;
-  active leaf pointers, history pointers, and definition-pointer counter domains:
  pointer UTF-8 bytes;
-  state activations: pointer, then numeric activation sequence;
-  variables: declaration pointer, then numeric declaring-state activation sequence.

Any duplicate canonical key or noncanonical order is `invalid_aggregate_state` ; a decoder never
silently sorts an accepted envelope. Serialization emits RFC 8785 JCS bytes of the complete envelope
with no byte-order mark, leading/trailing whitespace, or trailing newline. A parser may accept
insignificant JSON whitespace and then verify that the semantic data is canonical.

`vectors/persistence/aggregate-state-v1.json` is the normative human-readable aggregate example.
Conformance byte vectors MUST equal RFC 8785 serialization of their corresponding semantic value.

The digest is:

```text
aggregate_state_digest = hash([
  "determa-aggregate-state-digest-1",
  envelope_without_aggregate_state_digest
])
```

`hash` is the §9 SHA-256/JCS construction. A mismatch is `aggregate_state_digest_mismatch` . The
digest does not include database metadata, quarantine metadata, or package attachments. Ready and
deferred mailboxes are aggregate state and therefore are covered.

### 16.3 Complete root ownership aggregate

One envelope represents exactly one §3 root ownership aggregate. It contains:

-  the current validated-bundle fingerprint, namespace, root machine identity, machine
  format, root/creation/runtime identities, and migration sequence;
-  aggregate next logical-step and output sequences;
-  every retained root, component, and owned spawned runtime;
-  each runtime's immutable identity origin and immutable target identity;
-  each runtime's current definition binding and current relationship;
-  lifecycle status, active leaves and state activations, live variables, history,
  spawn/state/component counters, lifetime-holder association, and retained fault;
  and
-  every runtime's isolated ready and deferred mailboxes plus aggregate acceptance
  and queue-placement counters.

It contains no external broker backlog, timer, broker acknowledgement token, credential, transport
receipt, or plugin configuration.

The root runtime occurs exactly once and matches every top-level root identity field. Every other
runtime has exactly one retained owner. The ownership graph is acyclic and reachable from the root.
Runtime identifiers are unique. Component placement and owned-child relation data agree with the
immutable target identity and with the current target definition. Every active pointer, variable
declaration, history slot, component placement, spawn action, and counter domain resolves against
the runtime's current definition.

The envelope uses declaration pointers plus declaring-state activation sequences for live variables,
so shadowed names remain lossless. History records contain null or the exact recorded target-pointer
set. A wire fault record is exactly the five committed format-1 fields `runtime_id` , `cause_id` ,
`code` , `step_sequence` , and `source_locator` , plus required `definition_fingerprint` .
`step_sequence` uses the artifact canonical-decimal projection. The definition fingerprint anchors
the historical locator; migration never reinterprets that locator against a later definition.
Conformance faulted-state vectors MUST round-trip the complete fault record and MUST verify its
retained definition fingerprint against the definition that owns the historical locator.

Encoding first validates the implementation's abstract aggregate under the supplied source bundle.
Decoding verifies structure, canonical form, digest, definition availability and fingerprint, all
relationships, all typed values, and the complete §8 abstract-state invariants before returning
state. An implementation-specific dictionary, object graph, compiled machine, callback, or pointer
is never portable state.

### 16.4 Immutable identity and mutable definition binding

Migration separates three concepts for every existing runtime:

1.  `identity_origin` : the definition, owner, placement/action pointer, and allocation
   sequence from which the runtime identity was originally derived;
2.  `target_identity` : the immutable value accepted by already-created envelopes and
   nominal references; and
3.  `current_definition` and `relation` : the definition and current placement/action
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

It has no `kind` , namespace, owner/spawn metadata inside a spawned reference, or
definition/placement pointer inside a component target. Those facts belong to `identity_origin` or
`relation` . The four-field spawned reference is exactly §4.5, and the component target is exactly
§6.1. On the wire, spawned `machine_version` and component `activation_sequence` use the §16.2
decimal-string projections. A decoder MUST reconstruct their mathematical integer values before
comparing them with in-memory references or targets and before using them for routing or dispatch.

`runtime_id` , `identity_origin` , and the complete normalized `target_identity` bytes are invariant
across every descriptor. Root identity, component runtime identity, component activation sequence,
spawned instance reference, spawned instance id, spawn sequence, and existing lifetime-holder
activation identity are never rederived.

For a migrated component, author syntax using the target definition's current `component_id`
resolves through the current relation to its preserved target identity. For a migrated spawned
runtime, its existing `instance_reference` remains byte-for-byte stable while `current_definition`
changes. New components and spawned instances use the target definition normally. Identity rekeying,
external-reference rewriting, and detached child migration are unsupported.

### 16.5 Content-addressed definition registry

An aggregate references its current definition by the exact §8 `validated_bundle_fingerprint` . A
conforming resolver stores the canonical typed normalized bundle tree once under that key:

-  put-if-absent is idempotent;
-  the same key with different canonical bytes is an integrity failure;
-  bytes are rehashed and semantically revalidated before admission to a trusted local
  cache;
-  source and target definitions remain available while any aggregate, descriptor, or
  audit record references them; and
-  garbage collection is reference-aware, never age-only.

Content addressing proves integrity, not authority. A deployment separately allowlists or verifies a
signed release manifest containing trusted definition, descriptor, and route digests. Signature
algorithms, key management, and registry transport belong to the host.

An ordinary aggregate row does not embed its definition. This avoids copying old definitions into
every dormant row while still permitting lazy migration. Retaining old normalized declarative
definitions centrally does not retain old host executable logic.

### 16.6 Aggregate-shape fingerprint

The aggregate-shape fingerprint proves only that existing logical state can be bound to another
definition without transformation. It does not claim behavioral equivalence.

Starting from the §8 normalized bundle, implementations construct this exact plain JSON projection
before applying the §8 typed-value projection:

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

Machines retain bundle array order. Child states are sorted by state identifier UTF-8 bytes.
Variables are sorted by declaration pointer UTF-8 bytes. Components retain declaration order. Spawn
sites are sorted by action-pointer UTF-8 bytes after recursively visiting entry, exit, handler,
choice, and nested action lists.

Every state has `definition_pointer` and normalized `type` . `history` is included only for a
composite state and contains its normalized mode. Empty `variables` , `states` , `components` , and
`spawn_sites` arrays are omitted. Variable `nullable` is `true` only for an `instance_reference`
declaration and otherwise `false` ; normalized `input` and `external` are always included. Variable
`machine_id` is included only when declared. A component includes exactly one of `machine_id` or
recursive `inline_root` . `declaration_index` is its zero-based array index. A spawn site's holder
pointer is the resolved `bind_to` declaration pointer or null.

This recursive tree therefore contains every state, placement, spawn, holder, and
declaration-pointer domain from which a retained path, relationship, or counter key can be drawn.
Object keys are encoded with the §8 typed-tree map ordering. Metadata, event declarations and
payload defaults, guards, ordinary action expressions, transition targets, entry/exit behavior other
than spawn-site shape, and component `with` expressions are excluded because they cannot make an
existing logical-state field structurally invalid.

```text
aggregate_shape_fingerprint = hash([
  "determa-aggregate-shape-fingerprint-1",
  typed_state_bearing_tree
])
```

A `compatible` descriptor is valid only when independently recomputed source and target shape
fingerprints are equal and every mapping array is empty. It changes only the aggregate and runtime
current definition references and increments `migration_sequence` ; every other field is
byte-for-byte preserved before digest recomputation. A machine-version change, path change, rename,
or any other state-bearing projection difference requires `transform` mode.

### 16.7 Immutable declarative migration descriptors

A migration descriptor names exactly one source and one target machine format, validated-bundle
fingerprint, and independently recomputed aggregate-shape fingerprint. The descriptor requires both
machine formats to be numeric `1` . Its digest is:

```text
migration_descriptor_digest = hash([
  "determa-migration-descriptor-1",
  descriptor_without_migration_descriptor_digest
])
```

Changing any descriptor member creates a different descriptor. A digest match does not make it
trusted. The deployment must authorize the exact digest.

`transform` descriptors contain closed rules for machine/root bindings, active-state
materialization, variables, history, components, owned runtimes, lifetime holders, and counter
domains. Descriptors are immutable pure data. They cannot execute Python, Rust, JavaScript, WASM,
shell code, author actions, entry/exit behavior, transitions, choice selection, component/spawn
initialization, host callbacks, plugins, network or filesystem I/O, clocks, randomness, environment
reads, credentials, or secrets. Migration itself emits no author, lifecycle, internal, or external
event.

Variable `transform` and `initialize` rules use a closed migration CEL profile. It is the §5.2
portable profile restricted to null, Boolean, signed-64-bit integer, binary64, string, list, and
string-keyed map values and their already enumerated pure operators/functions. It excludes `event` ,
`owner` , runtime inspection, `instance_reference` , `has(event...)` , comprehension, iteration, and
every extension. For a transform, the descriptor's `source_declaration_pointers` order binds exact
symbols `source_0` , `source_1` , and so on. An initialize expression has no symbols. Each
expression is parsed and statically checked against source declaration types and the single target
declaration type before migration. Identity and nominal-reference mappings are descriptor
operations, never CEL values.

### 16.8 Exact route and migration algorithm

A route is the exact ordered array of trusted descriptor digests supplied by deployment
configuration or a trusted release manifest. The engine never searches a registry graph or selects a
shortest, newest, cheapest, or otherwise preferred path.

Before transformation:

-  the first descriptor source equals the stored aggregate fingerprint;
-  each descriptor target equals the next descriptor source;
-  the final target equals the requested target fingerprint;
-  no descriptor digest repeats and no source/target cycle occurs;
-  every definition and descriptor is present, hash-valid, semantically valid, trusted,
  and within declared resource requirements; and
-  the route's first source still equals the locked aggregate when execution begins.

An empty route with equal stored/requested fingerprints succeeds as a strict no-op: it returns the
exact input aggregate envelope bytes and `audit_records: []` , performs no artifact lookup beyond
the already required source-definition integrity and authorization checks, does not recompute the
digest, and does not require terminal maintenance mode. An empty route with unequal fingerprints
fails with `migration_route_missing` . Multiple available routes are irrelevant; only the pinned
ordered array is evaluated. All intermediate states remain in memory and only the final aggregate is
committed.

For each descriptor, migration:

1.  validates the complete source candidate against the exact source definition;
2.  reserves no logical-step, output, spawn, state, or component sequence;
3.  applies each mapping to an isolated candidate;
4.  increments `migration_sequence` exactly once;
5.  validates every field and relationship against the target definition;
6.  computes the target canonical envelope and digest; and
7.  either makes that candidate the next source or discards it completely.

The same canonical source bytes, exact route, trusted artifacts, and limits reproduce the same
success bytes and audit records or the same deterministic failure.

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

After every descriptor, no source field or required target field may remain unaccounted for.
Duplicate source consumption, duplicate target production, ambiguous mapping, invalid target type,
incompatible relation, stale current pointer, and counter inconsistency are
`migration_totality_failure` .

Mapping rules are definition rules, not single aggregate occurrences. A rule applies independently
to every retained runtime whose current source binding resolves the rule's source machine/root.
State, history, component, owned-runtime, holder, and counter mappings operate on each occurrence in
that runtime while preserving that occurrence's runtime and activation identity.

Variable occurrence identity is exactly:

```text
(runtime_id, variable_declaration_pointer, declaring_state_activation_sequence)
```

For each target runtime whose mapped active configuration makes a target declaration live, one
`copy` , `transform` , or `initialize` rule must produce that target occurrence. A `copy` consumes
the unique live source occurrence with the mapped declaration pointer in the same source runtime. A
`transform` resolves every `source_declaration_pointers[i]` to the unique live occurrence in that
same runtime and binds only its value as `source_i` . An `initialize` has no source occurrence. A
`drop` consumes each matching live source occurrence independently.

All inputs come from one immutable pre-descriptor snapshot. Rules cannot observe another rule's
output. Rules that consume one source occurrence twice, produce one target occurrence twice,
reference declarations outside the applicable source/target runtime bindings, or could combine
values from different runtimes are `invalid_migration_descriptor` . If a statically valid required
source occurrence is not live for an applicable target occurrence, is multiply live, or a required
target occurrence remains unproduced, the result is `migration_totality_failure` . Runtime iteration
uses canonical `runtime_id` order only for resource accounting; because occurrences are isolated and
rule domains cannot overlap, result bytes never depend on host map or iteration order.

When an active state is deleted or incompatible, the descriptor permits only:

-  an exact source-leaf to target-leaf mapping;
-  an explicit ancestor mapping accompanied by the complete resulting target leaf set;
-  a declared target initial/history selection only when it resolves without any guard
  or author action and the descriptor still provides the final leaf set and every new
  live value; or
-  deterministic failure and quarantine.

The engine never guesses by equal name, nearest surviving ancestor, initial state, history, or root
reset. Removed variables require an explicit destructive `drop` with an operator-facing reason. Type
changes require a statically checked transform. Removed live components, incompatible owned
runtimes, missing holders, or ambiguous identity mappings fail rather than being silently disposed.

### 16.10 Terminal aggregates

Completed and faulted aggregates remain terminal. Ordinary admission or processing is rejected under
the stored source definition before automatic host migration begins. Read-only inspection may
continue using the source definition without migration.

`maintenance_mode` is a required Boolean migration-request member. Omission or a non-Boolean value
is `invalid_migration_request` . An explicit migration may advance a terminal aggregate only when
`maintenance_mode` is true and every descriptor's matching terminal policy is `preserve` . A
non-empty route against a terminal aggregate with `maintenance_mode: false` is
`terminal_migration_requires_maintenance` ; a matching policy of `reject` is
`terminal_migration_rejected` . These failures preserve the exact source envelope and produce no
audit record. A completed migration preserves root identity, completed status, counters, history,
and diagnostics and creates no runtime or emission. A faulted migration preserves the root fault
including `step_sequence` , the frozen diagnostic tree, every retained child status/value, counters,
and historical fault-definition anchors; it cannot reactivate any runtime.

If no allowed route exists, retaining the terminal source aggregate is valid while its source
definition remains registered. A host requiring a uniform target may quarantine it. This
specification has no recovery, restart, or destructive reset policy.

### 16.11 Lazy transactional host ordering

A persistence host supporting lazy migration follows this ordering:

1.  Resolve and cryptographically verify the target definition, exact route, all
   descriptors, trust metadata, and resource requirements into a local immutable cache
   before taking the aggregate lock.
2.  Begin a serializable transaction or acquire an observably equivalent aggregate
   compare-and-swap guard.
3.  Lock/read the aggregate and the inbox receipt for the presented envelope.
4.  If that idempotency key is already committed, return its recorded outcome without
   migration or processing.
5.  Parse and verify the aggregate, resolve its source definition from the local cache,
   validate source state, and recheck route source.
6.  Apply the complete route to an in-memory copy, validating every intermediate.
7.  If the resulting aggregate is running and an accepted delivery is ready, invoke
   target-definition `step` exactly once.
8.  Build deterministic migration audit records and normal core result records.
9.  Atomically replace aggregate bytes, record receipt disposition, retain ordered
   internal emissions in target mailboxes, insert external intents in the outbox under
   their existing unique identities, and append audit rows.
10.  Commit once, then acknowledge broker ingress or dispatch outbox work.

`handled` , `unhandled` , `rejected` , and `faulted` are core outcomes, not storage failures. If the
host contract commits an outcome, migration and that outcome commit together. In particular,
target-definition fault finalization cannot commit while its preceding migration rolls back.

Definition/descriptor resolution, signature verification, and remote registry calls MUST NOT occur
inside the database transaction. A transaction conflict retries from the newly committed row. Parent
and owned-child state remains one aggregate transaction boundary.

### 16.12 Failure, rollback, quarantine, and audit

The closed deterministic migration failure codes are:

-  `invalid_aggregate_state` ;
-  `invalid_aggregate_state_package` ;
-  `aggregate_state_digest_mismatch` ;
-  `invalid_migration_request` ;
-  `source_definition_unavailable` ;
-  `target_definition_unavailable` ;
-  `definition_untrusted` ;
-  `definition_fingerprint_mismatch` ;
-  `migration_descriptor_untrusted` ;
-  `invalid_migration_descriptor` ;
-  `migration_route_missing` ;
-  `migration_route_mismatch` ;
-  `migration_transform_fault` ;
-  `migration_totality_failure` ;
-  `migration_resource_limit_exceeded` ;
-  `terminal_migration_requires_maintenance` ; and
-  `terminal_migration_rejected` .

The pure failure value is exactly `{ code: migration_failure_code }` ; it has no aggregate candidate
or audit-record member. The caller retains the exact supplied envelope. A successful result is
exactly `{ aggregate_envelope, audit_records }` .

Failure to produce a complete state valid under the target definition, including an invalid target
type, relationship, pointer, configuration, or required value, is `migration_totality_failure` . A
CEL evaluation or other runtime failure while executing a statically valid transform is
`migration_transform_fault` . There is no separate target-state-validation result code.

Artifact format/schema rejection occurs before this operation and uses
`unsupported_aggregate_state_format` , `unsupported_aggregate_state_schema_version` ,
`unsupported_migration_descriptor_format` , `unsupported_migration_descriptor_schema_version` ,
`unsupported_aggregate_state_package_format` , or
`unsupported_aggregate_state_package_schema_version` . After a recognized package format/version,
structural, attachment-uniqueness, or cross-reference failure is `invalid_aggregate_state_package` .
A recognized aggregate or descriptor that fails its closed schema is respectively
`invalid_aggregate_state` or `invalid_migration_descriptor` .

`source_definition_unavailable` means the definition named by the stored aggregate cannot be
resolved. `target_definition_unavailable` means the requested target or an intermediate non-source
definition cannot be resolved. A definition whose bytes reproduce its digest but whose digest is
absent from the deployment allowlist/signed manifest fails with `definition_untrusted` ; it is never
treated as unavailable or implicitly trusted. A descriptor has the parallel
`migration_descriptor_untrusted` result.

Registry/cache unavailability, transaction conflict, and temporary storage failure are transient
host failures. Digest mismatch, unauthorized/invalid artifacts, invalid request/route,
transform/type/totality/target validation failure, terminal-policy failure, and deterministic
resource-limit failure are permanent for the same state, request, and route.

Any descriptor failure discards every intermediate candidate. It writes no aggregate, inbox success,
outbox intent, successful audit, logical counter, or migration sequence. For a permanent failure, a
host atomically retains the exact original aggregate bytes, records a quarantine marker and failure
audit, and keeps the triggering inbox item blocked. Quarantine is host metadata, not an aggregate
lifecycle status and not a core dead-letter collection. Resolution installs a new trusted route,
restores an artifact, or explicitly releases the blocked item.

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

No other member is present. `migration_sequence` is the post-descriptor sequence. Source/target
state digests are the exact candidate digests immediately before and after that descriptor.
Conformance compares the complete ordered record list. Host observation time, worker identity,
database transaction id, and operator metadata may be stored alongside but outside this
deterministic record. Success and quarantine audit rows are append-only and transactional with the
state they describe.

### 16.13 Package transport

A package contains exactly one aggregate, zero or more normalized definitions, zero or more
descriptors, and the exact route digest array. It is a transfer/archive artifact, not the ordinary
row format.

Every normalized definition attachment must reproduce its declared validated-bundle fingerprint.
Every descriptor must reproduce its declared descriptor digest. Duplicate attachment digests are
invalid. The route obeys §16.8 and every artifact needed to decode the aggregate and execute that
route must be available either in the package or the receiving trusted registry. Package attachments
seed the same put-if-absent resolver contract; they never override a registry entry and are excluded
from the aggregate-state digest.

Portable packages MUST validate against `schema/aggregate-state-package-v1.schema.json` .
Conformance package vectors MUST fix their source and target bundle fingerprints, shared shape
fingerprint, aggregate digest, descriptor digest, and canonical attachment trees.

### 16.14 Security and resource limits

Digest verification is mandatory at every package/registry/cache boundary. Deployment pins an
allowed target and route, preventing silent downgrade or alternate-path selection. Definitions and
descriptors are immutable after trust admission. Source definitions and historical fault definitions
are retained while referenced.

Implementations expose configurable limits, but core conformance defines a minimum supported floor
for aggregate, definition, descriptor and transformed-output bytes; JSON nesting; runtimes; active
states; variables; map/list members; string bytes; migration-chain length; descriptor rules; and
migration CEL expression length, AST nodes, and evaluation steps.

Migration-chain length is exactly the number of descriptor digests in the requested route. Before
applying any descriptor, the operation compares that count with the implementation's configured
migration-chain-length limit. Exceeding it fails with `migration_resource_limit_exceeded` . The
empty route has length zero.

The four descriptor `resource_requirements` members are non-negative canonical-decimal upper bounds
for one descriptor application:

-  `maximum_transformed_output_bytes` : cumulative UTF-8 byte length of RFC 8785
  serialization of every typed value produced by `transform` and `initialize` rule
  occurrences; copied values and the surrounding aggregate envelope are not counted;
-  `maximum_cel_expression_length` : cumulative UTF-8 source bytes of all distinct CEL
  expressions declared by the descriptor, counted once each;
-  `maximum_cel_ast_nodes` : cumulative checked CEL abstract-syntax nodes across those
  distinct expressions, counted once each; and
-  `maximum_cel_evaluation_steps` : cumulative evaluator steps across every expression
  occurrence, including repeated applicable runtimes/activations.

Static expression length/node requirements are checked before evaluation. Dynamic evaluation-step
and transformed-output counters start at zero for each descriptor, advance in canonical runtime/rule
occurrence order, and may not exceed either the descriptor declaration or the implementation's
configured limit. A descriptor that understates actual use fails with
`migration_resource_limit_exceeded` ; the field is a limit, not permission to truncate work.

A `compatible` descriptor has no CEL expressions and produces no transformed/initialized typed
values, so all four requirements MUST be `"0"` . Structural definition-binding updates and final
aggregate serialization are deliberately not transformed-output bytes. A transform descriptor with
only pointer/identity mappings may also use zero. Cycles are forbidden. Each descriptor is checked
independently against its declared four bounds and the corresponding configured per-descriptor
limits. A route is bounded by its exact descriptor count plus those independent per-descriptor
checks. The descriptor defines no mixed-unit cumulative-chain-work counter and does not sum resource
dimensions across descriptors. Exceeding the chain-length limit or any deterministic per-descriptor
limit fails closed with `migration_resource_limit_exceeded` .

This section does not standardize a production database schema, object-relational mapper, registry
transport, queue plugin, distributed transaction, exactly-once external delivery, package import, or
bulk row rewrite. A later runnable database example belongs in the separate examples repository
after conformance and both engines implement this contract.

### 16.15 Queue-bearing artifact semantics

The aggregate contains `next_acceptance_sequence` and `next_queue_sequence` , plus each runtime's
`ready_mailbox` and `deferred_mailbox` . Each mailbox entry contains immutable `acceptance_sequence`
, current `queue_sequence` , delivery mode, complete normalized envelope, envelope digest, and
`deferral_count` . Entries in each mailbox are strictly increasing by mathematical `queue_sequence`
; acceptance sequences and event ids are unique across all runtime mailboxes. Both next counters
exceed every allocated value and are never reduced or reused. One entry occurs in exactly one
mailbox.

Each mailbox entry's envelope digest is:

```text
envelope_digest = hash([
  "determa-inbox-envelope-digest-1",
  "1",
  root_instance_id,
  delivery_mode,
  envelope
])
```

The digest is also the acceptance receipt's `request_digest` and the eventual terminal event
receipt's `request_digest` .

Aggregate serialization uses the §16.2 canonical rules and:

```text
aggregate_state_digest = hash([
  "determa-aggregate-state-digest-1",
  envelope_without_aggregate_state_digest
])
```

The complete logical checkpoint is the configuration, variables, identities, ready/deferred entries,
counters, receipts, and participating output state. A host may store those records in one document
or physically separate tables/objects, but one read must reconstruct one schema-valid revision and
one commit must replace it atomically or with observably equivalent serializable compare-and-swap
behavior. An enum or relational projection that cannot preserve or exactly reconstruct every
required member MUST reject before any application row, mailbox, receipt, or outbox mutation; it
never drops an unrepresented field. Ephemeral in-memory ownership is conforming when no durability
is claimed.

No aggregate artifact conversion or alternate mailbox-free representation is defined.

The migration descriptor directly contains its source/target identities, mappings, terminal and
resource policies, the fixed `queued_event_default: preserve_if_compatible` , and reusable
`queued_event_rules` . A rule is an explicit disposal override keyed by exact source `machine_id` ,
event name, and delivery mode and carries `action: dispose` plus a non-empty operator reason.
Duplicate selectors are invalid even when their reasons are equal. The descriptor is
definition-pair-specific but backlog-independent; it MUST NOT list event ids, runtime ids,
acceptance sequences, or queue sequences.

For every source mailbox entry, one matching disposal override removes it. Otherwise the fixed
default preserves the exact envelope, acceptance identity, current queue identity, location, and
deferral count only when the mapped target incarnation still exists and the target definition
accepts the exact event direction, correlation contract, and normalized payload without coercion. No
match plus any incompatibility is `migration_totality_failure` ; the source remains byte-for-byte
unchanged. A disposal creates terminal outcome `migration_disposed` with the rule's exact reason and
this descriptor's digest. It does not use lifecycle-only `disposed` reasons.

The descriptor digest is independently recomputed as:

```text
migration_descriptor_digest = hash([
  "determa-migration-descriptor-1",
  descriptor_without_migration_descriptor_digest
])
```

After every successful descriptor, preserved deferred entries use the bounded structural eligibility
table in §6.7 against the mapped stable configuration. No guard or action is evaluated. Entries not
structurally eligible remain in relative order; eligible entries move to the ready tail in prior
deferred order with fresh queue sequences. Changing only a deferral declaration is therefore
explicit and deterministic, not silent event loss. Migration executes no handler action and emits no
event. After disposal and structural recall, each runtime's resulting deferred occupancy MUST be no
greater than its mapped root `deferred_event_capacity` when one is declared. Lowering capacity below
retained occupancy without enough explicit disposal rules fails the whole migration unchanged;
migration never evicts an entry merely to meet capacity.

Mailboxes in a retained-faulted runtime or frozen descendant remain part of the source and use the
same preservation/disposal rules, but structural recall is skipped for that subtree because it is
non-runnable. Preserved entries remain frozen and serializable until a successful owner cleanup or
root tombstone disposes them. Migration cannot revive, retarget, or process them. The package
carries exactly one aggregate-state envelope and only direct migration descriptors. Alternate
artifact versions and version mixing are invalid.

## 17. Portable execution checkpoints and hosting adapters

### 17.1 Scope

The execution checkpoint is the sole portable durable-host state for exactly one root ownership
aggregate transaction boundary. It combines the current aggregate-state envelope or terminal
tombstone with durable operation receipts, event-identity tombstones, pending/terminal/compact
outbox work, and migration audit. Accepted events live exactly once in aggregate runtime mailboxes;
there is no host-level duplicate pending-work collection.

A host MAY embed this contract directly in an application process; no daemon, socket, broker,
database server, background thread, or subprocess plugin protocol is required. Every
`ExecutionStore` is one logical store scope. The host assigns it one opaque scope identity and binds
it to exactly one owning party or deployment trust domain. Mutually untrusted parties or deployment
trust domains MUST NOT share a logical store scope. Multiple authenticated users or service
principals MAY operate within one such trust domain; principal authentication, authorization,
approvals, and audit remain host policy and do not require a separate store per principal.

Within the execution-checkpoint profile, root, creation, operation, event, and effect identities;
checkpoint revisions and digests; receipts and tombstones; and replay, conflict, no-reuse, and
deduplication guarantees are unique or evaluated only within the selected logical store scope. Equal
portable identities and bytes MAY coexist in independent scopes. A portable identity or digest is
not globally unique and is not evidence that an artifact belongs to a host scope.

A physical backend MAY contain multiple logical store scopes only when a mandatory external
isolation key participates in every lookup, mutation, uniqueness constraint, transaction or
compare-and-swap guard, lock, replay check, receipt, tombstone, and outbox operation. Before any
such operation, the host MUST select and authorize exactly one scope. Missing, ambiguous,
mismatched, or unauthorized selection MUST fail closed without a core call, checkpoint mutation,
outbox delivery, broker acknowledgement, fallback, probing, or access to another scope.

The active scope identity, ownership binding, principal policy, and physical isolation key are host
metadata outside core and checkpoint semantics. They MUST NOT be added to a machine document,
aggregate state, migration descriptor, aggregate-state package, execution checkpoint, event, effect
intent, or their digest inputs. A §22 archive MAY record nonsecret source scope and binding
provenance, but those bytes never select or authorize an active scope. Machine namespaces and
portable identities do not select or authorize a scope. The portable engine, bundle, checkpoint, and
their hash bytes remain tenant-agnostic; credentials, endpoints, tenant identifiers, and SaaS policy
fields remain host configuration.

The schemas remain unchanged because scope selection is deliberately external host metadata. The
execution-checkpoint conformance profile MUST add host-adapter cases proving that equal portable
identities coexist independently in two logical scopes; that equal portable `effect_id` values
route, retry, reconcile, and deduplicate independently in those scopes; and that missing, ambiguous,
mismatched, or unauthorized scope selection makes no core `create` , `admit` , `step` , or migration
call, leaves checkpoint bytes unchanged, and performs no outbox mutation, delivery, or broker
acknowledgement. Those artifacts are follow-up work and are not added here.

The checkpoint is strict UTF-8 JSON under §16.1. Unknown formats and versions fail with
`unsupported_execution_checkpoint_format` and `unsupported_execution_checkpoint_schema_version`
before semantic validation. A recognized artifact that fails schema or semantic invariants is
`invalid_execution_checkpoint` ; a wrong digest is `execution_checkpoint_digest_mismatch` .

The closed checkpoint-host conflict codes are `event_id_conflict` , `creation_id_conflict` ,
`operation_id_conflict` , `effect_id_conflict` , and `checkpoint_revision_conflict` . They are
deterministic non-core failures and preserve the supplied committed checkpoint byte-for-byte.

### 17.2 Closed checkpoint artifact

The complete schema is `schema/execution-checkpoint-v1.schema.json` . A checkpoint has exactly:

```text
{
  execution_checkpoint_format: "determa.execution_checkpoint",
  execution_checkpoint_schema_version: 1,
  root_instance_id: non_empty_string,
  revision: canonical_decimal,
  root_record: { status: "retained", aggregate_state } | root_tombstone,
  replay_retention: permanent_or_bounded_retention,
  next_operation_receipt_sequence: canonical_decimal,
  operation_receipts: [operation_receipt, ...],
  event_identity_tombstones: [event_identity_tombstone, ...],
  pending_outbox_intents: [pending_outbox_intent, ...],
  next_outbox_terminal_sequence: canonical_decimal,
  terminal_outbox_records: [terminal_outbox_record, ...],
  outbox_effect_tombstones: [outbox_effect_tombstone, ...],
  migration_audit_records: [migration_audit_record, ...],
  execution_checkpoint_digest: sha256_string
}
```

For a retained root, checkpoint and aggregate root identities match and the aggregate passes all §16
validation. The first successful checkpoint has revision `"0"` ; every later changing transaction
advances it exactly once. Reads, equal replays, failed transactions, and CAS conflicts change no
byte or counter.

```text
execution_checkpoint_digest = hash([
  "determa-execution-checkpoint-digest-1",
  checkpoint_without_execution_checkpoint_digest
])
```

The normative example is `vectors/persistence/execution-checkpoint-v1.json` . Operational leases,
locks, credentials, connection details, broker acknowledgement tokens, wall-clock attempt
timestamps, worker identities, and application rows are not checkpoint members.

### 17.3 Durable operation receipts and replay

Operation receipts allocate zero-based `receipt_sequence` values in commit order. A fresh checkpoint
is created only from a successful `create` result. Its creation receipt is sequence `"0"` ,
`committed_revision` is `"0"` , and `next_operation_receipt_sequence` begins at `"1"` ;
initialization lifecycle receipts then allocate in defined order. The creation receipt is retained
for the checkpoint's lifetime.

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

A successful `create` , including a committed faulted initialization, atomically writes revision
`"0"` , the retained aggregate, creation receipt, and all referenced mailbox entries/outbox intents.
Retrying the same root and creation id with the same digest returns
`{ result: "committed", receipt: creation_receipt }` without calling `create` . Any different
creation id or request digest for an existing root identity is `creation_id_conflict` . The rule
applies equally after terminal tombstoning; the root identity is not recreated.

A creation rejected before an aggregate exists does not create an execution checkpoint or reserve
the root identity. Such a rejection is outside checkpoint replay. An application requiring replay of
rejected creation requests MUST commit its own request record and response under an
application-owned identity. This is safe because no Determa aggregate, emission, or checkpoint
mutation was committed.

An acceptance receipt proves host admission, not processing. It records event identity, the
`determa-inbox-envelope-digest-1` request digest, acceptance sequence, accepted revision, and input
delivery mode. Internal emissions append directly to a target ready mailbox in the producing RTC and
are referenced by that operation's receipt; they do not create acceptance receipts.

A terminal event receipt proves exactly one `handled` , `unhandled` , `faulted` , `disposed` , or
`migration_disposed` outcome and records the immutable acceptance identity, final queue sequence,
committed revision, resulting aggregate digest, and ordered emission references. Acceptance and
terminal receipts may coexist because they attest different facts; the complete envelope remains in
only one live or terminal location.

Each emission reference's `emission_index` copies the referenced emission's local, zero-based
ordinal defined by §9. For an external author effect, it copies the action-local `emission_index` .
For an internal emission, the receipt field `emission_index` copies §9's `emission_ordinal` :
action-local for an author emission or lifecycle-operation-local for an engine lifecycle emission.
It is not a transaction-wide ordinal and is not the reference's index in `emission_references` .
Consequently, references to emissions from different actions or lifecycle operations may carry the
same `emission_index` . Array position preserves cross-action and cross-lifecycle-operation core
emission order; `event_id` for an internal reference or `effect_id` for an external-outbox reference
identifies the referenced emission unambiguously.

Equal replay returns retained evidence without mutation. Unequal content for the same identity is
`event_id_conflict` . Replay/conflict checks precede terminal-root rejection. Permanent retention
keeps every receipt. Bounded retention may replace one acceptance-plus-terminal unit with one
event-identity tombstone using only digest domain `determa-inbox-envelope-digest-1` . Pruning is
dependency-closed and MUST NOT leave a dangling producer, mailbox, terminal, outbox, or audit
reference.

Applications needing replay of an HTTP body, domain projection, or historical aggregate MUST store
that response as application data in the same host transaction. It is not a checkpoint member.

### 17.4 Aggregate-owned admission and processing

Admission is one atomic checkpoint mutation. The host normalizes the complete ordered batch,
including source, cause, exact target, payload, correlation, and envelope digest, then applies this
closed order before allocating anything:

1.  malformed batch/member: `malformed_delivery` ;
2.  wrong checkpoint root: `wrong_root` ;
3.  duplicate event id within the batch: `duplicate_event_id_in_batch` ;
4.  retained identity replay/conflict comparison;
5.  non-replay target on completed/faulted root: `terminal_root` , or on tombstone:
   `tombstoned_root` ;
6.  mode, source, target, declaration, payload, correlation, and digest validation.

The remaining closed admission failures are `event_id_conflict` , `invalid_delivery_mode` ,
`invalid_delivery_source` , `invalid_instance_target` , `inactive_component_target` ,
`invalid_event` , `invalid_payload` , `invalid_correlation` , and `delivery_digest_mismatch` . Any
failure rejects the complete batch without mutation or acknowledgement. An all-replay batch is
read-only; a mixed replay/new batch allocates only new members and commits once.

Successful host admission allocates one aggregate-wide `acceptance_sequence` and initial
`queue_sequence` per new envelope in caller order, appends it to its exact target ready tail,
creates one input acceptance receipt, advances revision once, and recomputes both digests. Only then
may an adapter acknowledge transfer of ownership. An adapter claiming §21 MUST retain the exact
source-to-acceptance binding and prove that it committed atomically with this admission under §21.1.

Processing explicitly targets one runtime and selects only its ready head. A deferred result
atomically moves the entry to that runtime's deferred tail with a new queue sequence and no terminal
receipt. Structural recall creates no receipt. A terminal result removes the entry and creates
exactly one terminal event receipt. Successful lifecycle cleanup creates disposal receipts in
cleanup order; a cleanup fault rolls back the cleanup and every tentative receipt.

Terminal outcomes are exactly `handled` , `unhandled` , `faulted` , `disposed` , and
`migration_disposed` . Lifecycle `disposed` has exactly one reason: `runtime_cancelled` ,
`runtime_completed` , `aggregate_completed` , or `root_tombstoned` . `migration_disposed` instead
carries a non-empty operator reason and the exact migration descriptor digest. A successful
lifecycle operation creates disposal receipts in runtime cleanup order and, within each runtime,
ready entries followed by deferred entries in queue order. Receipt sequences allocate in that order
and may share one committed revision. Fault-frozen entries remain owned until successful cleanup or
tombstoning.

The core `step` result reports every successful cleanup removal in `lifecycle_dispositions` . The
checkpoint host creates the selected causal event's terminal receipt first when terminal, then one
terminal `disposed` receipt for each lifecycle disposition in list order. An `internal_disposed`
emission names one list index and becomes an `internal_terminal` reference to the resulting receipt;
an `internal_mailbox` reference names its sole retained mailbox entry. Missing, duplicate,
mismatched, or out-of-range references are `invalid_execution_checkpoint` . State, receipts, outbox
intents, revision, counters, and digests commit together or not at all.

The lifecycle of one accepted event is closed:

| current location | operation | committed next location |
|---|---|---|
| external/unaccepted | rejected admission or pre-commit crash | external/unaccepted |
| external/unaccepted | committed admission | one target ready mailbox plus acceptance receipt |
| ready | enabled handler succeeds | terminal `handled` receipt |
| ready | no enabled handler, active deferral, capacity available | same runtime deferred mailbox |
| ready | no enabled handler, no active deferral | terminal `unhandled` receipt |
| ready | guard/action/invariant or capacity-overflow fault | terminal `faulted` receipt |
| deferred | structural recall becomes eligible | same runtime ready tail |
| deferred | structural recall remains ineligible | same deferred position |
| ready or deferred | successful owner disposal | terminal `disposed` receipt |
| ready or deferred | explicit migration disposal rule | terminal `migration_disposed` receipt |
| any engine-owned location | transaction failure before commit | exact prior location and bytes |

The lifecycle is closed: an accepted event is in exactly one ready or deferred mailbox, then one
terminal receipt or event tombstone. Transaction failure preserves the exact prior location. A
checkpoint containing only deferred entries is pending but not runnable. Empty ready mailboxes do
not authorize polling, timers, or spontaneous execution.

Definition migration is one transaction over the aggregate, every mailbox, generated disposal
receipt, audit record, outbox member, and revision. An event whose target incarnation or declaration
disappears, or whose payload/correlation contract becomes incompatible, cannot be guessed, coerced,
or silently discarded. It must remain exactly compatible or match an explicit disposal rule;
otherwise the migration fails unchanged.

### 17.5 Maintenance-migration operations

A migration with no delivery MUST carry a non-empty host-supplied `operation_id` . The writer also
supplies the exact checkpoint revision and digest it read, as required by §17.9. The operation id is
unique among retained native maintenance-migration receipts for the root. Its request digest is:

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

It atomically replaces the aggregate, appends the §16 audit records, writes the receipt at the
current `next_operation_receipt_sequence` , increments that counter, increments checkpoint revision
exactly once, recomputes the checkpoint digest, and commits. The receipt's `committed_revision` is
that new revision. Its source and resulting aggregate digests are respectively the exact
pre-transaction aggregate and committed aggregate digests. Its `target_validated_bundle_fingerprint`
is the exact target definition fingerprint used in the request digest. The receipt retains that
definition identity even after root tombstoning removes the aggregate.

For a non-empty route of length N, `result_code` is `migration_applied` , exactly N §16 audit
records are appended in descriptor-route order, and `migration_sequences` is exactly their ordered
sequence list. The values are strictly increasing and contiguous: the first is the source
aggregate's `migration_sequence + 1` , and the last is the resulting aggregate's
`migration_sequence` . A one-hop route therefore has one audit record and one sequence; a multi-hop
route has one of each per descriptor.

For an empty route, source and target definition identities are equal, the aggregate bytes and
aggregate digest are unchanged, no migration audit record is appended, `result_code` is
`migration_no_operation` , and `migration_sequences` is empty. The checkpoint transaction still
allocates its receipt and increments checkpoint revision once so response loss cannot make the
operation ambiguous. Its retained `target_validated_bundle_fingerprint` equals the unchanged source
aggregate definition fingerprint. A loader verifies the canonical request digest from the receipt's
root, operation, source aggregate digest, target definition fingerprint, empty descriptor route, and
Boolean maintenance mode even when the root record is a tombstone. Because the receipt does not
repeat `maintenance_mode` , its digest is valid only when it equals the canonical construction for
one of the two exact Boolean values; replay still compares the caller's complete request digest
byte-for-byte.

After loading and validating the checkpoint, the host checks retained operation identity before
applying the caller's stale-writer guard. Presenting the same operation id and digest returns
exactly `{ result: "committed", receipt: maintenance_migration_receipt }` without rerunning
migration or changing revision, even when the replay carries the revision and digest from its
original request. Reuse with a different digest is `operation_id_conflict` and preserves the
checkpoint. If no retained replay/conflict applies, a mismatched expected revision or checkpoint
digest is `checkpoint_revision_conflict` . A deterministic migration failure produces no successful
receipt or candidate aggregate; quarantine/failure audit remains host metadata under §16.12.
Retrying that failure is safe because no migration state committed.

An implementation claiming this checkpoint profile MUST NOT expose an unkeyed maintenance-migration
commit path. Operator tools MAY generate an operation id, but must display and reuse it when
retrying after an unknown response.

### 17.6 Durable outbox lifecycle

Every committed external emission enters `pending_outbox_intents` in the same transaction as its
source operation. One entry retains the complete §9 intent:

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

`state_revision` is the checkpoint revision that inserted or most recently changed the pending
state. Initial insertion uses the producing operation receipt's `committed_revision` .

`retryable_failure` means the destination did not confirm acceptance and policy permits another
attempt. `ambiguous` means acceptance may have occurred but no durable confirmation was obtained; it
MUST be retried with the same `effect_id` or resolved by an operator/destination-specific
reconciliation. Both remain pending with the full intent. Attempt timestamps, connection errors, and
credentials are operational data outside the portable artifact; `reason_code` is a stable
host-defined identifier.

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

After `not_attempted` , the seven closed attempted-delivery states have exact meanings:

-  `confirmed` — the destination supplied the adapter's configured durable acceptance
  confirmation;
-  `retryable_failure` — no confirmation, retry remains permitted, still pending;
-  `ambiguous` — confirmation is unknown, still pending;
-  `permanently_rejected` — the destination definitively refused the intent and retry
  is forbidden;
-  `operator_cancelled` — an authorized operator deliberately ended delivery;
-  `discarded` — declared policy deliberately ended delivery without destination
  acceptance; and
-  `dead_lettered` — declared policy moved responsibility to a durable terminal
  dead-letter record.

`confirmed` , `permanently_rejected` , `operator_cancelled` , `discarded` , and `dead_lettered` are
terminal and retain the complete intent. Terminal records allocate zero-based `terminal_sequence`
values from `next_outbox_terminal_sequence` and remain ordered by that sequence. An `effect_id`
occurs exactly once across pending and terminal full/compact outbox sets. The producing operation
receipt references that same id.

The pending-to-pending or pending-to-terminal update increments checkpoint revision once and is
atomic. `state_revision` or terminal `committed_revision` is set to that new revision. A failed
update leaves the prior state unchanged. Reuse of an existing effect id with unequal intent is
`effect_id_conflict` ; equal insertion from a replayed source operation is a no-op because the
source receipt prevents redispatch.

A pending-state update request names the effect id, desired closed pending state, and the checkpoint
revision/digest read by the writer. If the desired state exactly equals the stored state, the host
returns `{ result: "committed", record: pending_outbox_intent }` with its existing `state_revision`
; it does not apply the stale-writer check, mutate, or increment revision. This is the response both
after the first committed update and after response loss. A genuinely different pending state
requires a successful revision/digest compare-and-swap, changes state once, sets `state_revision` to
the new checkpoint revision, and returns the same committed shape. Repeated equal
`retryable_failure` /`ambiguous` reports therefore cannot consume revisions indefinitely.

Once terminal, the record is immutable. Repeating the same terminal outcome returns the exact
terminal record without mutation; requesting a different terminal outcome fails `effect_id_conflict`
.

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

This `outbox_effect_tombstone` preserves the exact effect identity, original terminal sequence,
terminal policy evidence, and enough immutable content evidence to replay an equal insertion or
reject unequal content. Compacting a full terminal record replaces it atomically with the tombstone,
preserves its `committed_revision` , and increments checkpoint revision once. Retrying equal
compaction returns the existing tombstone without mutation. An effect id occurs in exactly one of
pending, full terminal, or effect-tombstone storage.

A full terminal record or compact effect tombstone may be deleted only when no retained operation
receipt references its effect id. The producing receipt must first be pruned under the
dependency-safe §17.8 rules. Otherwise silent deletion is `invalid_execution_checkpoint` . This
makes weak cleanup compatible with receipt linkage without requiring every host to retain complete
historical payloads.

Under the strict durable-outbox host profile, an intent that is not confirmed MUST remain pending or
move atomically to one retained terminal outcome. Silent deletion, compaction to an effect
tombstone, retention expiry of terminal records, and best-effort fire-and-forget are forbidden. A
host permitting any of those weaker policies MUST declare a weaker outbox policy and MUST NOT claim
the strict durable-outbox profile. A compact-retention profile retains effect tombstones while
referencing receipts exist. A bounded cleanup profile may delete tombstones only after those
receipts are dependency-safely pruned. A policy deleting either side earlier is not a valid
checkpoint profile.

Delivery begins only after the checkpoint transaction commits. An intent remains in pending, full
terminal, or compact tombstone form until its permitted atomic lifecycle update commits. A crash
after remote acceptance but before confirmed terminalization leaves it ambiguous/pending and may
deliver it again. External delivery is therefore at least once. It is effectively once only when the
destination treats `effect_id` as an idempotency key. Determa does not claim distributed ACID,
universal external exactly-once delivery, or proof of remote business success. Remote outcomes
become aggregate facts only through later declared input envelopes.

### 17.7 Migration audit and canonical ordering

`migration_audit_records` contains the exact successful §16.12 records in commit order, strictly
increasing by `migration_sequence` . Permanent replay retains complete successful history. Bounded
replay may remove audit records only in the same dependency-safe transaction that prunes every
receipt referencing them. Failed or quarantined migration metadata is host-owned because it does not
describe a committed aggregate replacement.

Canonical semantic order is exact:

-  every runtime's ready and deferred mailbox is ordered by mathematical
  `queue_sequence` ;
-  operation receipts are ordered by mathematical `receipt_sequence` ;
-  event-identity tombstones are ordered by mathematical
  `terminal_receipt_sequence` ;
-  pending outbox intents are ordered by mathematical intent `sequence` ;
-  terminal outbox records and effect tombstones are each ordered by mathematical
  `terminal_sequence` ; and
-  migration audit records are ordered by mathematical `migration_sequence` .

Acceptance sequences and event ids are unique across all runtime mailboxes. A live mailbox event id
is disjoint from terminal receipts and event tombstones; terminal receipts and event tombstones are
disjoint. An acceptance receipt may overlap its live mailbox or terminal receipt but not the
tombstone that replaced it. Maintenance operation ids are unique. Effect ids and terminal sequences
are unique and disjoint across pending, terminal, and compact outbox forms.

Every `internal_mailbox` emission reference resolves to its sole retained mailbox entry. Every
`internal_terminal` reference resolves to its exact terminal receipt or a tombstone retaining that
sequence. Every external reference resolves to one pending intent, terminal record, or effect
tombstone. Every retained migration receipt retains the target definition fingerprint used by its
canonical request digest. When its audit records remain, their final target fingerprint equals the
receipt target fingerprint. An empty migration appends no audit; its receipt fingerprint remains
authoritative for request-digest validation after aggregate tombstoning.

Every retained allocation is below its next counter. Duplicate allocation, noncanonical order,
cross-set overlap, dangling or unequal linkage, wrong-root record, digest conflict, invalid
retention transition, impossible revision/outcome union, or counter regression is
`invalid_execution_checkpoint` .

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

Creation receipt sequence `"0"` is always first and retained for the checkpoint's lifetime. In
permanent mode, retained receipt sequences are exactly the contiguous range from zero through
`next_operation_receipt_sequence - 1` .

In bounded mode, a non-null cutoff `C` MUST be strictly less than `next_operation_receipt_sequence`
. Every non-creation sequence from `1` through `C` has been pruned, and every sequence from `C + 1`
through `next_operation_receipt_sequence - 1` is retained exactly once. With a null cutoff, the same
contiguous permanent allocation rule applies. An empty retained suffix is valid only when
`C = next_operation_receipt_sequence - 1` ; the cutoff can never cover the next unallocated
identity.

Pruning is dependency-closed. No mailbox entry may depend on a pruned producer receipt. No retained
internal emission reference may lose its mailbox entry, terminal receipt, or event tombstone. Audit
records and terminal/compact effect records referenced only by pruned receipts may be removed in the
same transaction. Advancing the cutoff commits all required removals and the new revision
atomically. Equal cutoff replay is read-only; a lower cutoff, skipped sequence, dangling dependency,
or cutoff at or beyond the next counter is `invalid_execution_checkpoint` .

The transition from permanent to bounded is allowed and irreversible. `permanent_replay_eligible`
becomes false in that same commit and can never become true for this root identity, even if the
current retained set later happens to contain all new receipts. A checkpoint restored from bounded
mode or with a non-null pruning cutoff MUST remain bounded. This recorded history prevents prior
cleanup from being hidden by later configuration.

The checkpoint retains root identity evidence in both permanent and bounded modes. A completed or
faulted aggregate MUST remain as a retained terminal aggregate, or may be replaced only after all
accepted mailbox entries and pending outbox intents are resolved by a root tombstone:

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

Tombstoning is one compare-and-swap mutation with a stable operation id, preserves all receipts
retained under the selected mode, terminal outbox records/effect tombstones, and retained migration
audit, and increments revision once. It does not permit admission, processing, migration, new
pending work, or aggregate reconstruction. The first commit and an equal retry return exactly
`{ result: "tombstoned", tombstone: root_tombstone }` ; the retry does not change revision. Another
operation id fails with `operation_id_conflict` . A running aggregate cannot be tombstoned.

In both retention modes, `root_instance_id` is never reused after creation, including after
completion, fault, application deletion, backup, restore, or tombstone compaction. Physical deletion
of the checkpoint or its root identity marker is unsupported. Bounded mode may prune only the
dependency-safe receipt/audit/effect history defined above; it never prunes creation receipt `"0"` ,
the retained terminal aggregate/root tombstone, or root identity.

A complete backup MUST retain every checkpoint, including every bounded or permanent terminal
checkpoint/root tombstone. A restore that omits one loses root identity, creation-conflict, and
no-reuse evidence and is not a conforming restore of this checkpoint profile. Bounded restores
retain their recorded horizon and cannot be changed to permanent replay. Permanent restores may
advertise permanent replay only when the collection is complete.

### 17.9 Transaction and concurrency ordering

A durable host processes one presented delivery in this exact order:

1.  Select and authorize exactly one §17.1 logical store scope, then resolve, verify,
   authorize, and locally cache all required definitions, migration descriptors, route
   metadata, adapter configuration, and capability declarations.
2.  Begin one transaction with exclusive ownership of the root checkpoint, or an
   observably equivalent compare-and-swap guard over its exact revision and digest.
3.  Read and validate the checkpoint, pending identity, and retained operation receipts.
4.  On an equal pending or committed identity, return its pending result or receipt
   without redispatch. On a digest conflict, fail without mutation.
5.  If migration is requested, apply the complete §16 route to an in-memory copy.
6.  Invoke `step` exactly once against that candidate, or `create` exactly once for
   an absent root creation.
7.  Build the candidate root record, delivery/operation receipt, accepted mailbox entries,
   pending/terminal/tombstoned outbox records, migration audit records, counters, next
   revision, retention state, and digest.
8.  Atomically commit the complete candidate plus any application rows participating
   through a shared native transaction.
9.  Only after commit, acknowledge broker ingress and begin pending outbox delivery.

The transaction includes migration and processing even when processing is unhandled, rejected, or
faulted. A failure before step 8 preserves the prior checkpoint byte-for-byte. A crash after step 8
but before ingress acknowledgement causes redelivery to return the recorded receipt without
migration, processing, duplicate mailbox admission, or duplicate outbox insertion.

Every writer supplies the revision and digest it read. A stale writer fails with
`checkpoint_revision_conflict` ; it MUST NOT overwrite, merge, or append to the newer checkpoint. It
restarts from the committed checkpoint and re-evaluates pending or receipt replay. A
`durable_concurrent` execution store MUST provide serializable behavior or this exact
compare-and-swap result. Lost updates are nonconformant.

### 17.10 Execution-store registration and resolution

This specification defines adapter behavior, not a language API, binary interface, wire protocol,
database schema, or cross-language dynamic-loading mechanism. Execution stores also obey the common
§11.5 identity, direct-injection, descriptor, configured-report, and requirement rules. An adapter's
URI scheme in this section is a lookup key distinct from its exact provider-reference identifier.
The `duplicate_adapter_registration` and `adapter_capability_mismatch` codes below are the
established execution-store-specific outcomes of those common rules. Applications SHOULD be able to
inject an execution-store object directly without a registry, URI, discovery, or command-line
interface. If an implementation offers any adapter identifier, URI, or scheme resolution, it MUST
expose and use one public registry whose registration operation associates:

-  one lowercase adapter identifier and URI scheme matching
  `[a-z][a-z0-9+.-]*` ;
-  one factory;
-  configuration validation;
-  capability evaluation for the resulting configured instance;
-  health operations; and
-  optional adapter-storage schema migration operations.

URI parsing extracts the scheme generically and asks that registry to resolve it. Core, host, and
command-line code MUST NOT branch on a particular adapter identifier. Registration of an already
registered identifier fails with `duplicate_adapter_registration` ; later registration never
overrides the first. Resolution of an absent identifier fails with `unknown_adapter` . Invalid
configuration fails with `invalid_adapter_configuration` . Unsatisfied requested capabilities fail
with `adapter_capability_mismatch` before any root is created, loaded, or processed.

Bundled and third-party execution stores use the same public registration operation, validation,
resolution, and error behavior. In particular, ordinary bundled identifiers `memory` , `file` ,
`sqlite` , and `postgresql` receive no private switch branch, precedence, override right, discovery
path, or implicit capability elevation. An implementation need not bundle all four. If it bundles
one, that registration is observationally indistinguishable from a third-party registration except
for who invoked the public operation. Automatic bundled inclusion is an ordinary startup call to
that same operation, not pre-population of a privileged internal map.

Automatic loading of arbitrary installed code is forbidden. Discovery is explicit or restricted by a
host allowlist because factory loading executes code. A Python host MAY opt into package-entry-point
discovery. A Rust host MAY link crates that explicitly register trait-object factories. Both MUST
preserve direct object injection. A stock binary exposes only registrations it explicitly includes
or explicitly discovers. A language-neutral subprocess or socket protocol is future work and is not
required for embedded hosts.

### 17.11 Execution-store capabilities and composed host profiles

Execution-store capabilities describe only the configured store instance, not its name,
implementation family, ingress adapter, broker, or outbox worker. The standard store capability
names and guarantees are:

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

Hosts declare every required store capability before processing. Capability evaluation occurs after
configuration validation because durability and retention may depend on transaction isolation,
filesystem synchronization, journal mode, connection topology, or cleanup settings. There is no
implicit capability inheritance: a store advertises every capability it proves.

`memory` MUST advertise only `ephemeral` from this standard set. A `file` adapter MAY advertise
`restart_persistent` , but MUST NOT claim either durable capability merely because files survive
normal restart. A configured `sqlite` adapter MAY advertise `durable_single_writer` only when its
transaction and synchronization behavior proves that guarantee. A configured `postgresql` adapter
MAY advertise `durable_concurrent` and `shared_application_transaction` only when its isolation,
locking/revision checks, and transaction API prove them. No identifier alone proves a capability.

Host profiles describe a composition of an execution store with ingress handling, queue policy, an
outbox worker, destination semantics, and application transactions. They are not execution-store
capabilities. Every durable checkpoint host profile below requires `root_identity_retention` ;
physical checkpoint/root-marker deletion is not a weaker profile:

-  `durable_embedded_processing` requires a durable store and the §17.4 atomic
  accept/process rules; no broker is required.
-  `exactly_once_committed_processing` additionally requires permanent checkpoint
  retention mode and `permanent_receipt_retention` .
-  `broker_integrated` requires a durable store, an ingress adapter that acknowledges
  only after checkpoint commit, durable redelivery behavior, and an outbox worker; a
  store alone can never claim it.
-  `strict_durable_outbox` requires the §17.6 total lifecycle,
  `permanent_outbox_terminal_retention` , and an outbox worker that never silently
  deletes unresolved work.
-  `compact_durable_outbox` permits full terminal intents to become §17.6 effect
  tombstones, requires `compact_effect_identity_retention` , and forbids deleting a
  tombstone while a retained receipt references it.
-  `shared_application_transaction` is available to a host only when the store exposes
  that store capability and the application actually uses one native transaction.

A bounded checkpoint cannot satisfy `exactly_once_committed_processing` . Selecting `memory` for any
durable host profile fails before processing rather than silently weakening the requested profile. A
host MUST validate the complete composed profile, not infer it from the storage scheme.

### 17.12 Exact guarantee boundary

For every identity still covered by the checkpoint's replay-retention guarantee, the checkpoint
contract provides exactly-once **committed processing** within one selected §17.1 logical store
scope:

-  at most one aggregate replacement is committed for that identity;
-  every retry with equal content returns the first durable host receipt;
-  accepted mailbox entries, outbox records, migration audit, application rows included in a
  shared transaction, and revision change commit with that receipt; and
-  an uncommitted attempt has no durable effect.

This guarantee does not mean exactly-once network receipt, broker delivery, remote side effect, or
globally distributed transaction. Broker delivery may repeat before acknowledgement. External
effects are at least once and require destination idempotency for effectively-once behavior. Hosts
MUST state their selected durability, retention, queue-ordering, broker, and external-idempotency
profiles without attributing stronger guarantees to the pure core.

Every store operation and every outbox routing, retry, reconciliation, and idempotency record MUST
retain the selected logical store scope. External effect idempotency MUST use the host-owned scope
identity together with the portable `effect_id` ; `effect_id` alone is not globally unique. This
host metadata MUST NOT alter the portable intent.

### 17.13 Cluster checkpoint composition

One execution checkpoint never represents a complete deployment or an independently committed owned
child: every owned child remains inside its root aggregate. A complete deployment backup is a
consistent collection containing:

-  one valid checkpoint for every created root identity, with either a retained
  aggregate or root tombstone in its `root_record` ;
-  every content-addressed normalized definition referenced by those aggregates and
  fault anchors;
-  every trusted migration descriptor and route needed by deployment recovery policy;
-  adapter metadata needed to restore pending broker ownership without treating an
  uncommitted message as accepted;
-  relevant application data; and
-  a manifest that identifies the exact member bytes and the consistency point chosen
  by the host.

The checkpoint schema does not define that cluster manifest or require a global transaction across
unrelated roots. A cluster backup is valid only if the storage and broker-specific procedure
supplies an application-appropriate consistency point and does not omit committed
checkpoints/tombstones, operation receipts, accepted mailbox entries, pending/terminal/tombstoned
outbox records, retention history, application response data needed by its API, or referenced
trusted artifacts. A restore that omits any created root's checkpoint/tombstone is nonconforming in
either retention mode. A restored deployment may claim permanent replay only when the collection is
complete and every checkpoint remains permanently eligible. Restoring one checkpoint requires no
other root checkpoint, but application-level cross-root invariants may require coordinated backup
and restore.

This section defines only checkpoint-collection completeness. The optional host authority interface
in §18 governs fencing and ownership; it does not turn a checkpoint collection into a scope archive
or a relocation proof. Backup and restore operation protocols, cloning, transfer or rebinding, and
multi-scope archives require their separate host-profile contracts. Portable checkpoint bytes,
identities, or digests MUST NOT authorize access to or movement across logical store scopes.

### 17.14 External timer durability

Machine format 1 introduces no timer semantics and the checkpoint has no timer member. A host
therefore MUST NOT advertise accepted timer work as covered by a durable checkpoint while retaining
that work only in process memory. A timer contract that participates in durable processing MUST add
a versioned checkpoint representation for accepted scheduling requests, deadlines, cancellation
state, and deterministic delivery identity, or use a separately identified durable timer artifact or
an external durable service whose accepted ownership and recovery boundary is stated explicitly. The
optional §23 helper uses a separate artifact.

Adding non-empty timer state requires a later checkpoint schema version or a separately identified
durable timer artifact.

## 18. Optional host scope authority

### 18.1 Boundary and capability claims

Scope authority is an optional host or plugin interface. The pure foreground core accepts an exact
definition, prior portable state and input, then returns the next state, dispositions and intents
(§8). An embedding application may commit that result in its own transaction without an authority
service. The core does not select a scope owner, allocate an epoch, fence a worker, coordinate hosts
or create an authority grant. Hosts MUST NOT treat core evaluation or ordinary root compare-and-swap
as evidence of any such operation.

An authority provider registered under §11.5 MAY claim `authoritative_scope_fencing` ,
`consistent_scope_inventory` and `safe_relocation` only for the exact configured instance and
topology in which it proves them. A host also reports its effective, separately checked
`guarded_local_writes` , `worker_fencing` , `complete_scope_inventory` and `safe_relocation`
guarantees as explicit Booleans; without a proved authority provider all four are false. The first
two registered capability names bind respectively to §18.3 and §18.4. A `safe_relocation` registry
claim and a true host-profile report mean the configured instance has proved support for the
specified source, destination and topology. They do not assert that this transfer has already frozen
the source, proved retirement or activated the destination. The host rechecks that support at each
operation; activation additionally requires the transfer-specific §18.5 evidence. A generic instance
report cannot authorize a particular transfer. The closed
`schema/host-authority-profile-report-v1.schema.json` composes the §11.5 extension report, or null
when no authority provider is configured, with the exact authenticated scope, epoch and generation
when authority is present, a nonsecret authority-storage-boundary identifier, topology identifier
and configuration digest, process and host boundaries, source and optional destination binding
digests, required participant references, and the four effective guarantee Booleans. With no
authority provider, epoch, generation and authority storage boundary are null and every authority
guarantee is false. It is a host profile report, not an extension registration report or authority
credential. Descriptions and digests bind a configured topology but are not themselves proof of
fencing. `safe_relocation: true` requires an exact destination binding and verified support for all
§18.5 predicates in that topology; their operation-specific evidence is checked before activation.
The report MUST NOT contain credentials, live claims, retirement grants or secret connection
strings. A host returns it only after authenticating and authorizing the caller for the scope; a
request for another scope reveals no report or existence fact. A new configuration, health state,
source or destination requires a new report. A profile MUST recheck these facts before an operation
that depends on them. An unknown, unhealthy or unproved claim fails closed as
`host_capability_mismatch` .

The 0.3.0 public reference profile MAY implement the operations below within one tested SQLite
authority database or one configured PostgreSQL authority database. It MUST advertise only behavior
its actual transaction and crash tests prove. A copied SQLite file is inert data, not a second
authority. A deployment using the same PostgreSQL database may support guarded local processes if
that topology is proved. Neither topology by itself proves retirement across independent authority
backends. Distributed coordination, leader election, independent-authority grant issuance and a
managed control plane are outside this release. A third-party host MAY later supply those
capabilities through this same public boundary.

### 18.2 Identity, records and authorization

An authority domain allocates each logical `scope_identity` exactly once and retains permanent
no-reuse evidence, including after archive export, restore, tombstoning or deletion of portable
checkpoints. An authority record contains the exact `scope_identity` , `ownership_binding_digest` ,
`authority_epoch` , `owner_binding` , `state` , `scope_generation` , `active_transfer_id` and
operation receipts. The `authority_epoch` and `scope_generation` are monotonically increasing
canonical decimal integers. `state` is `active` , `freezing` , `frozen` , `retired` or
`transaction_in_doubt` . `active_transfer_id` is null unless a transfer is in progress. The owner
binding and authority record are trusted host data. They are never imported from a portable
checkpoint/archive as executable authority.

The public version-1 authority operation envelope has exactly `interface` , `interface_version` ,
`operation` , `operation_id` , `scope_identity` , `expected_authority_epoch` ,
`expected_scope_generation` , `request_digest` , and `arguments` . `interface` is
`determa.host_authority` and `interface_version` is `1` . `request_digest` is
`hash(["determa-host-authority-request-1", request_without_request_digest])` , using §9's
SHA-256/JCS construction. An operation ID is unique within its authenticated scope. The host
authenticates the caller through trusted transport/invocation context, authorizes the operation and
scope before revealing existence or replay evidence, and checks the current owner, epoch and state
before mutation. Credentials, endpoint aliases and principal claims are not machine fields or
portable archive members. Neither a scope ID, operation ID, digest, archive, root ID, effect ID nor
an epoch number is a bearer credential.

The closed interface operations in this section are `read_authority` , `guarded_commit` ,
`freeze_scope` , `fence_worker` and `prove_retirement` . A `guarded_commit` represents the host's
native commit boundary; it is not a request for the core to perform storage I/O. Each operation's
`arguments` is closed by `schema/host-authority-operation-v1.schema.json` . A result has exactly
`interface` , `interface_version` , `operation` , `operation_id` , `status` , `scope_identity` ,
`authority_epoch` , `scope_generation` , `state` , `evidence_digest` , `error_code` , and `claim` .
`claim` is the exact active worker claim only for a successful `fence_worker` ; it is null
otherwise. For success, `evidence_digest` is
`hash(["determa-host-authority-evidence-1", request_digest, result_without_evidence_digest])` . It
binds the exact request digest and response bytes for integrity and replay; it does not by itself
prove a transaction committed, an inventory is complete or a source retired. The host retains the
response with the actual committed operation receipt and linked authority/proof records in its
trusted ledger. A safety decision MUST resolve that retained ledger record and verify its linked
committed evidence; caller-supplied response bytes or a matching digest alone never authorize it. An
authorized `read_authority` binds its returned snapshot without committing a new receipt.
`error_code` is null; for rejection, the digest is null and `error_code` is one of the closed errors
in that schema. Results expose no owner or scope details to an unauthorized caller. `read_authority`
has null expected epoch and generation and is read-only; an authorized receipt read may return
historical evidence without granting mutation rights.

Request shape, known protocol name/version, closed arguments and request digest are validated before
an operation result can be formed. Malformed, unknown-protocol or unsupported-version requests
return the schema's closed `earlyError` with null operation, operation ID, scope, epoch, generation,
state, evidence and claim. Its `error_code` is respectively `invalid_host_request` ,
`unsupported_host_protocol` or `unsupported_host_protocol_version` ; no parsed request field is
echoed. Authentication and scope authorization follow structural validation and precede scope lookup
and replay; a valid but unauthorized request receives `unauthorized_scope` with null scope, epoch,
generation, state, evidence and claim. Only after these checks does the host compare the request
digest against the retained operation receipt. Unknown fields, invalid canonical values or a
computed digest mismatch cause `invalid_host_request` before mutation. These early errors never
reveal cross-scope existence or current authority metadata.

For a mutation, the host stores `(scope_identity, operation_id, request_digest)` and the exact first
result with the native authority change. Once current authentication and scope authorization pass,
equal replay returns that result without mutation; unequal reuse returns `scope_operation_conflict`
. A new operation with a stale epoch or generation returns `stale_scope_authority` or
`scope_generation_conflict` . The host MUST NOT resolve current aliases to change an already
recorded operation's identity or target. A request whose outcome is unknown retries/queries its same
authenticated scope and operation ID; a timeout never licenses a new epoch or takeover. If replay
evidence has expired under a declared bounded retention profile, the host reports
`replay_evidence_expired` , not presumed rollback.

### 18.3 Guarded commit, freeze and transaction fate

For `authoritative_scope_fencing` , every hosted mutation of a checkpoint, host journal, operation
receipt, worker claim or ingress acknowledgement obtains a scope operation guard and validates the
current owner, epoch and active state. The guard MUST remain effective through the actual native
transaction commit or rollback. `guarded_commit` binds the digest of the host's exact proposed
native mutation, checks the expected generation, and advances it atomically with that mutation and
receipt. The host MUST reject a mutation whose bytes do not match that digest. A check in a separate
transaction followed by an unfenced root compare-and-swap does not conform. The authority domain
serializes a freeze and all guarded commits; once `freezing` is visible, it grants no new writer
guards. Freeze waits for existing guarded transactions to reach known commit or rollback, revokes
current hosted worker claims and records unresolved external attempts as ambiguous, then may record
`frozen` and a complete inventory consistency point. A root-level CAS, process lock, lease timeout,
readable backup or disconnected client proves none of these facts.

If the native transaction fate is unknown, the record enters or remains `transaction_in_doubt` ; the
host MUST refuse a frozen inventory, source retirement, new owner activation and a safe-relocation
claim. It may clear that state only after the authoritative storage boundary proves the
transaction's commit or rollback and the old session is unable to commit later. A lease expiry alone
is insufficient. The host MUST bound and contain runtime-provider evaluation while it holds a writer
guard if it advertises this capability. Committed external handlers run after the commit. A database
rollback cannot undo native provider I/O already performed and a scope guard cannot fence an
arbitrary external destination. Claims about external effect safety additionally require destination
idempotency/fencing or explicit reconciliation under that helper's contract.

### 18.4 Worker fences and scope inventory

`fence_worker` is available only when the composed host proves `worker_fencing` for the actual
worker and journal topology. Its request names the exact root, effect work identity and operation
token, plus `expected_attempt_fence` (null only if no claim has ever been issued) and
`expected_worker_principal` . The principal field is an equality precondition, never authentication:
the host verifies it against the trusted authenticated caller and the authorized assignment. A
mismatch returns `worker_principal_mismatch` without mutation; a stale attempt precondition returns
`stale_attempt_fence` . The host allocates an attempt fence strictly greater than any previous fence
for that work, chooses expiry from its own host clock and policy, and atomically commits the new
active claim, journal state, generation and operation receipt under the §18.3 guard. The success
result's closed `claim` contains the scope, root, work kind/identity, operation token, current scope
authority epoch, new attempt fence, authenticated worker principal, `expires_at` and `active` state.
The fixed host clock basis is `unix_nanoseconds` ; `expires_at` is a canonical signed decimal
Unix-epoch nanosecond value in the signed 64-bit interval
`[-9223372036854775808, 9223372036854775807]` . `-0` , floating values and values outside that range
reject before a claim is issued. The host selects expiry using its trusted clock and configured
policy, never an implicit core clock. A claim is expired at `now >= expires_at` ; if current trusted
time is unavailable, dispatch and result submission fail closed without treating a possible external
call as undone. `vectors/authority/host-authority-clock-cases-v1.json` gives normative boundary
values. A worker claim is host authority data; neither its fields nor its result grant a different
principal access. Dispatch and result submission recheck the authenticated principal, active claim,
current epoch and attempt fence. A stale epoch/attempt or revoked claim cannot mutate the
checkpoint, journal or ingress acknowledgement. Claim expiry may revoke mutation rights but never
proves that an external call did not execute; unresolved attempts become ambiguous and require
proved destination idempotency or reconciliation before retry. A portable archive MUST NOT activate
a source worker claim at the destination.

`consistent_scope_inventory` requires the host to enumerate every created root, retained checkpoint
or root tombstone, receipt, pending/terminal intent and required host journal record at one proved
frozen consistency point. It MUST discover these from authoritative storage, not accept a
caller-supplied list as proof. The inventory references exact definition and migration artifacts and
separately declared required application/helper participants. Missing required participant state is
an explicit failure. A single portable checkpoint or a selected collection can still be exported for
standalone inspection or restoration, but cannot claim complete scope inventory solely by being well
formed. Portable snapshots remain usable without this optional capability.

### 18.5 Retirement and relocation boundary

`prove_retirement` requires a frozen, fate-known scope. It atomically records `retired` , advances
the scope generation and returns retained, destination-bound, single-use evidence that the old owner
cannot commit another guarded mutation. A later owner may advance the epoch only when that evidence,
exact destination binding, generation and all required participant proofs are verified inside the
same trusted authority domain. A transfer must also prove that the source can never resume and that
no second destination can consume the grant. The archive supplies state, never that proof. A copied
database, timeout, endpoint switch, credentials change or archive digest does not prove retirement.
The `freeze_evidence_digest` input is the successful `freeze_scope` result's `evidence_digest` , not
its request digest. The host resolves it to the retained committed freeze operation and checks its
matching scope, epoch, generation, known transaction fate, revoked claims and inventory consistency
point. A request digest, unretained response hash or mismatched freeze record returns
`scope_fence_unproven` without retiring the scope. Retirement proof and any single-use grant
likewise live in the trusted authority ledger; their response digest alone is never a grant.

Future archive/transfer host operations may expose `prepare_transfer` , `stage_import` ,
`commit_transfer` and `activate_import` against this version-1 authority boundary. A host claiming
`safe_relocation` MUST prove source retirement, single-use destination-bound grant, transaction
fate, complete import and absence of concurrent writers for its advertised topology. Equal operation
replay retains its first result; stale generation, changed destination, consumed grant or uncertain
source fate rejects. An import may be validated and staged inactive when safe relocation is
unsupported, but MUST NOT become active as the same logical scope. Unsupported transfer returns
`host_capability_mismatch` or `scope_fence_unproven` ; no host may silently invoke a fresh-scope
takeover instead. The archive/import contract separately defines the complete artifact and
participant checks; this section neither requires a distributed grant service in 0.3.0 nor allows an
implementation to claim safe relocation without one where its topology requires it.

Strict fenced recovery remains read-only and inactive when old-owner retirement is unproved. A
separately requested standalone takeover may create a never-used new scope and idempotency namespace
from validated portable state, retaining source provenance and marking inherited unresolved external
work ambiguous. It has `safe_relocation: false` and `no_duplicate_external_work: false` ; source
receipts and claims do not authorize new-scope requests. This operation requires its own explicit
risk acknowledgement and import contract. It is never a fallback from a failed or unsupported safe
relocation. `vectors/authority/host-authority-cases-v1.json` contains normative positive and
negative interface cases. The companion `vectors/authority/host-authority-profile-cases-v1.json`
fixes positive and negative report and capability combinations.

## 19. Committed native effects and authenticated results

### 19.1 Scope and distinct identities

This optional host profile executes only a selected, committed §17 external outbox intent. It does
not execute an uncommitted core result, a merely proposed intent, or a request inferred from a
pending journal row. The pure §8 core needs no journal, worker, scheduler, coordinator, or native
handler. A host claiming durable native results MUST supply an atomic checkpoint, outbox,
pinned-route, and journal-outcome boundary and the authority capability required for every claim it
advertises (§18). An embedded callback without those capabilities MUST report its weaker guarantees;
it MUST NOT advertise durable worker claims, safe relocation, or exactly-once external execution.
This profile defines no distributed authority service.

Ingress `event_id` , host request `operation_id` , durable business `operation_token` , and
deterministic §9 `effect_id` have separate lifetimes and MUST NOT be interchanged. The nonempty
opaque `operation_token` identifies one outstanding business invocation through request replay,
attempts, and its result. It is chosen before the producing transaction commits. A workflow that
checks the token at machine level carries it in a declared initiating input, retained variables, and
a declared emitted payload. The pinned route copies that exact value into its declared result
location. This adds no reserved machine field. For host-only use a host MAY derive
`hash(["determa-host-operation-token-1", scope_identity, root_instance_id, effect_id])` ; that value
is host metadata and cannot be presented as a token already known by the machine. Correlation may
express a declared business link but never authenticates a worker, claim, result, or scope. External
destination idempotency is scoped by `(logical_scope_identity, effect_id)` ; scope is not added to
the portable intent.

### 19.2 Closed host journal and pinned route

`schema/host-effect-journal-v1.schema.json` is the closed version-1 host journal. Its fields are
exactly `host_effect_journal_format` , `host_effect_journal_schema_version` , `scope_identity` ,
`root_instance_id` , `checkpoint_revision` , `checkpoint_digest` , `journal_revision` ,
`effect_records` , `operation_response_references` , and `host_effect_journal_digest` . The digest
is `hash(["determa-host-effect-journal-digest-1", journal_without_host_effect_journal_digest])` .
The journal is host-owned and scoped; it is not part of the portable aggregate or checkpoint digest.
Its checkpoint pair MUST match an extant committed checkpoint at the same root and revision.
Inserting an intent and its effect record, recording an outcome, result admission, and every
checkpoint/outbox mutation that changes their joint facts MUST commit atomically. Recovery MUST
reject a torn checkpoint/journal pair. A metadata-only journal mutation MAY advance
`journal_revision` without a core step and MUST retain the unchanged checkpoint reference. Journal
revision starts at `"0"` and advances exactly once per changing journal transaction; equal replay
does not advance it.

`effect_records` are strictly ordered by effect ID UTF-8 bytes and contain no duplicate ID.
`operation_response_references` are strictly ordered by operation ID and contain no duplicate ID.
Each reference binds a retained host operation ID to
`hash(["determa-host-operation-response-1", normalized_response])` . The normalized response is the
exact public result returned for the committed operation, excluding transport-only headers,
credentials and redacted views. The host MUST retain or be able to reconstruct those exact bytes
while it claims equal operation replay; a digest alone is not the response. A referenced intent MUST
exist in exactly one pending, terminal, or compact checkpoint outbox location, and its recomputed
digest MUST match `intent_digest` . Attempt records are strictly ordered by numeric fence, contain
no duplicate fence, and no fence exceeds the record's `attempt_fence` . `unclaimed` , `leased` , and
`ambiguous` have null outcome, result ID, and admission receipt; `outcome_recorded` has a terminal
outcome and result ID but null admission receipt; `result_admitted` and `closed` have all three,
with the receipt's event ID equal to the result ID. A terminal outcome's fence MUST name its
corresponding immutable attempt report, except the §19.3 host-finalized preclaim `cancelled`
outcome: it has fence `"0"` , no attempt report, and `cancellation.state: prevented_start` . No
other terminal outcome may omit its attempt report. Schema validity alone is insufficient for these
checks. Unknown journal format/version, unequal digest, dangling checkpoint reference, or semantic
inconsistency fails closed before any dispatch or result admission.

Each effect record contains exactly `effect_id` , `operation_token` , `intent_digest` ,
`handler_reference` , `destination_binding_digest` , `route_configuration_generation` ,
`result_mapping` , `target` , `idempotency_policy` , `attempt_fence` , `attempt_records` ,
`invocation_state` , `outcome` , `result_event_id` , `admission_receipt` , and `cancellation` .
`effect_id` references exactly one committed checkpoint intent. `intent_digest` is the §17.6
complete-intent digest, including its domain, root, and original intent. The `native_handler`
reference is the exact §11.5 provider reference, resolved against host allowlist and dependency
policy. Its binding digest identifies the precise destination and connector configuration without
containing credentials. Configuration generation is a canonical decimal string. The target pins root
instance ID, runtime ID, and exact runtime incarnation; a current alias or a later reactivation
cannot redirect it. The ordered result mapping entries contain exactly `outcome_kind` , `event` ,
`result_slot` , and `operation_token_location` . Every permitted terminal business outcome has one
declared result event and a distinct nonempty slot. Token location is `null` for host-only tokens,
`{"kind":"correlation_id"}` , or `{"kind":"payload","pointer":canonical_json_pointer}` into the
decoded logical declared payload map, before §16.2 typed projection. A token location MUST resolve
to a declared string field; the host, not the worker, inserts the pinned value. A result is admitted
only in the event's declared input mode to the pinned runtime incarnation. Rebinding to a new target
or route requires new work, never a mutation of this record.

Route resolution and authorization occur before the producing core call. The host revalidates the
route generation and exact binding under the producing transaction's commit guard. A changed
generation fails before commit; an equal operation replay returns its saved binding without
resolving current aliases. Later configuration edits apply only to new intents. Every dispatch
checks current scope, worker, handler, and destination authorization. Revocation retains work for
cancellation or reconciliation; it never silently reroutes the pinned intent. Current authorized
configuration supplies secrets only to the handler, outside portable values and journal digests.

### 19.3 Invocation, claims, attempts, and cancellation

`invocation_state` is exactly `unclaimed` , `leased` , `ambiguous` , `outcome_recorded` ,
`result_admitted` , or `closed` . A new record begins `unclaimed` with attempt fence `"0"` , no
reports and null outcome, result ID, admission receipt, and cancellation. Claiming changes it to
`leased` at the next fence. A proved safe `retryable_failure` report returns it to `unclaimed` ; an
`ambiguous` report changes it to `ambiguous` . An authorized reconciliation MAY resolve ambiguity or
issue a new fenced claim only with its recorded external evidence. A terminal report records the
immutable outcome and changes it to `outcome_recorded` ; successful admission changes it to
`result_admitted` . `closed` may follow only after result admission and retains all identity and
outcome evidence required by the declared replay policy. These are business-invocation states
independent of the §17.6 outbox delivery state. Outbox `confirmed` proves only durable adapter
acceptance; it does not prove the native provider succeeded. A remote worker may accept
responsibility while invocation remains outstanding. Runtime/root completion does not silently erase
that outstanding record or late-result policy.

The closed claim shape is `schema/host-effect-claim-v1.schema.json` . A claim has exactly
`scope_identity` , `root_instance_id` , `work_kind` , `work_identity` , `operation_token` ,
`scope_authority_epoch` , `attempt_fence` , `worker_principal` , `expires_at` , and `state` . The
active form MUST match the closed ten-field §18.4 `workerClaim` with canonical signed 64-bit
Unix-epoch nanosecond expiry. Negative zero, float, or overflow is invalid; expiration is determined
from trusted host time at `now >= expires_at` , and unavailable trusted time fails closed.
`scope_authority_epoch` MUST equal the current §18 `authority_epoch` ; a numeric match alone is not
a credential. For this journal `work_kind` is `effect` , `work_identity` is its effect ID, and state
is `active` , `revoked` , or `expired` . The current claim is an authority record outside the
archive; historic claim evidence MAY be retained for audit but grants no authority. Issuing a claim
increments the effect's canonical decimal `attempt_fence` , changes it to `leased` , and commits
both facts under the host authority guard. Only the authenticated current worker principal, active
unexpired claim, current scope authority epoch, and exact attempt fence authorize dispatch or result
submission. Expiry/revocation and new claims serialize under that guard. The authenticated principal
and matching active scope epoch and attempt fence MUST still hold at the journal transaction commit.
Lease expiry can revoke host writes but cannot prove whether external work happened. Import never
restores a live claim; an unresolved inherited attempt becomes ambiguous before any new attempt.

Native handlers receive a declared §16.2 typed portable input, immutable invocation metadata, and an
attempt context. They may build arbitrary SDK, protobuf, or other native objects internally. Such
objects are not portable input, result, checkpoint, or journal values. A handler has no direct
aggregate mutation or host transaction capability. Its report kind is `succeeded` ,
`domain_rejected` , `retryable_failure` , `terminal_failure` , `cancelled` , or `ambiguous` .
`retryable_failure` requires evidence that another attempt is safe; unknown provider acceptance,
unclassified exception, worker disappearance, or expired attempt is `ambiguous` unless definitive
evidence proves no call occurred. A report is immutable and contains exactly `attempt_fence` ,
`report_kind` , `report_digest` , and `reason` . `report_digest` is
`hash(["determa-effect-attempt-report-1", effect_id, operation_token, attempt_fence, report_kind, payload, reason])`
, with the exact §16.2 typed payload and `null` or a stable nonempty reason code. The host retains
external evidence separately under access policy; its digest can be included in the portable payload
only when declared. A duplicate equal report replays and an unequal report for the same fence
conflicts. Reports do not consume the final outcome slot.

Retry after ambiguity requires proved destination deduplication under the same scoped `effect_id` ,
or explicit authorized reconciliation. A new attempt always receives a new fence; an old worker
cannot overwrite current journal or checkpoint state. Neither a lease nor a journal row proves a
provider call did or did not occur. No universal exactly-once claim follows from a host transaction.
`outcome` is null until a terminal business outcome and then is immutable with exactly `kind` ,
`payload` , `digest` , and `attempt_fence` . Its typed portable payload and digest use §16.2 and
`hash(["determa-effect-outcome-1", effect_id, operation_token, kind, payload, attempt_fence])` .
Attempt reports remain separate evidence.

Cancellation is null or exactly `operation_id` , `reason` , and `state` , where state is `requested`
, `prevented_start` , `too_late` , or `reconciliation_required` . Cancellation and outcome recording
serialize. The closed `schema/effect-cancellation-request-v1.schema.json` has exactly `operation_id`
, `effect_id` , `reason` , and typed `payload` ; authenticated scope and principal are transport
context. The closed response is `schema/effect-cancellation-response-v1.schema.json` . It serializes
with claim issuance and outcome recording under the current §18 scope guard. The `operation_id` and
exact normalized request are retained for equal replay or `operation_id_conflict` ; equal replay
returns the exact retained response bytes without mutation. The host validates the request payload
against the pinned `cancelled` event declaration and inserts or verifies the exact pinned token. If
it wins before any claim, the host MUST require an exact pinned `cancelled` result mapping and a
valid declared result payload. In one durable journal transaction it records
`cancellation.state: prevented_start` , an immutable `cancelled` outcome with fence `"0"` , and the
deterministic mapped result event ID; invocation state becomes `outcome_recorded` . No worker
attempt or provider call is made, and no later claim may issue for this effect. The host then admits
the pinned `cancelled` event through §19.4, in an independent atomic checkpoint/journal transaction.
Crash recovery resumes that admission from the stored outcome. If no declared `cancelled` mapping
exists, cancellation fails before mutation; the host MUST NOT invent an event or leave an
unclaimable `unclaimed` invocation. After a call might have occurred, cancellation records
`reconciliation_required` and returns the closed response status of the same name with null outcome
and result ID. Its request and response digest are retained in `operation_response_references` ;
equal replay returns identical bytes. The host cannot claim rollback or erase ambiguity. A recorded
outcome wins over later cancellation; a late report cannot replace it. Equal cancellation returns
retained operation evidence without mutation; a changed request cannot rewrite an immutable outcome.

### 19.4 Authenticated result submission and admission

The closed request shape is `schema/effect-result-request-v1.schema.json` ; the closed response
shape is `schema/effect-result-response-v1.schema.json` . The result request contains exactly
`effect_id` , `operation_token` , `attempt_fence` , `outcome_kind` , and `payload` . The
authenticated transport context supplies principal and scope independently of those fields. Before a
fresh worker report or outcome commit the host validates the current authority epoch, live unexpired
claim and fence at trusted host time, authenticated worker principal, exact scope and pinned route,
outstanding invocation, exact token, allowed outcome and declared portable payload. It computes the
immutable attempt report digest from the normalized request and reason, then records that report
with the outcome when terminal. The worker check remains true through the outcome commit; an expired
or revoked claim cannot create a fresh outcome. A subsequent equal replay requires current scope and
principal authorization, exact retained request/outcome evidence, and the original authenticated
claim principal; it does not require reviving an expired claim or admitting again. An old authority
epoch never gains replay rights across a scope transfer. An expired or revoked worker claim returns
`stale_attempt_fence` before any worker-originated outcome commit or core admission. A wrong
business token or unknown/closed unrelated invocation returns `effect_not_outstanding` ; a stale
fence returns `stale_attempt_fence` ; conflicting content for an already recorded outcome returns
`effect_result_conflict` . Validation failure performs no core call and changes no checkpoint,
journal, or outbox bytes.

A response has exactly `status` , `effect_id` , `attempt_fence` , `attempt_report` , `outcome` ,
`result_event_id` , `admission_receipt` , `checkpoint_revision` , `journal_revision` , and
`error_code` . `status: committed` returns the exact terminal outcome and acceptance receipt; equal
replay returns the same response bytes and revisions. `status: report_recorded` returns the
immutable retry or ambiguity attempt report with null outcome, result ID, and receipt.
`status: rejected` returns only the request effect ID/fence and one closed error code; all evidence
and revisions are null, so unauthorized callers cannot infer whether another scope contains work.
Rejected requests allocate no revision. A retry or ambiguity report leaves the checkpoint revision
unchanged. The journal revision advances only on a fresh report.

For a terminal mapped outcome, result event identity is
`hash(["determa-effect-result-event-1", effect_id, result_slot])` . The host builds the full
normalized input envelope from the pinned event, target and token mapping, including the declared
result payload. Its identity and bytes are immutable across transport retry or response loss. The
host persists the outcome first. Admission of an already committed `outcome_recorded` result is a
host-owned recovery operation: it requires the current authorized scope and §18 guarded commit,
validates the immutable journal outcome, pinned route, target incarnation, declared result schema,
token mapping, deterministic event ID and complete envelope against the stored evidence, and admits
that exact event. It does not require the old worker claim to remain live, create a new claim, or
call the provider. A host-finalized preclaim `cancelled` outcome uses the same path. After a crash,
even if the old claim expired, recovery resumes this operation from the stored outcome. Admission
and journal `admission_receipt` commit atomically. The receipt is the exact §17 accepted event
receipt; it records the result event identity. Processing is a later independent core operation.
Equal submission returns retained outcome/admission evidence without new core admission; unequal
envelope or outcome is `effect_result_conflict` . If admission fails because the pinned runtime is
now ineligible, the outcome remains durable and is explicitly reconciled; the host MUST NOT redirect
it. `result_admitted` and `closed` never permit another result admission.

### 19.5 Required conformance evidence

A host claiming this profile MUST pass positive and negative cases for route generation change
before commit; exact route replay after configuration change; revoked dispatch; SDK-native objects
remaining inside a handler; scope/token/principal/epoch/fence validation before admission; equal
replay versus unequal conflict; stale and duplicate worker reports; cancellation before claim and
after possible call; and terminal outbox acceptance with business outcome still pending. Crash cases
MUST cover intent commit before dispatch, provider acceptance before outcome commit, outcome commit
before admission, and admission commit before response. The first ambiguous window retains
uncertainty; it is not converted to failure. Retrying ambiguous work requires the stated idempotency
or reconciliation proof. Tests MUST compare checkpoint and journal bytes before and after every
rejected operation and prove no unauthorized provider call or core admission. Optional relocation
tests run only when the host advertises its §18 authority capability; otherwise safe relocation is
explicitly refused and staged imports remain inactive. These are public conformance obligations for
any future hosted service claiming the same capability.

## 20. Lossless application projection and embedded transaction facade

This optional host contract binds selected application rows to one root ownership aggregate. It does
not change machine grammar, the §8 foreground result, the §16 portable aggregate, or the §17
execution checkpoint. An application owns row selection, its object-relational mapping, transaction,
and response. A projection extension uses the §11.5 `projection` category and MAY claim
`lossless_projection` only for a configured mapping whose complete supported domain satisfies this
section. Direct injection and named registration have the same validation and capability rules.

### 20.1 Selected rows and exact reconstruction

The mapping identifies an explicit finite set of application rows for one selected root. Other
application rows are outside the projection. The mapping MUST define a total decode from its
selected rows and permitted supplemental storage to exactly one schema-valid aggregate or
checkpoint, and an encode of every reachable committed result back to those locations. For every
supported state `s` , decoding the encoding of `s` MUST reconstruct all logical fields and their
exact canonical typed values, identities, ordering, references, and digests. Physical bytes may
differ only where the portable decoder explicitly permits equivalent encodings; the reconstructed
portable artifact MUST have the canonical §16/§17 bytes and digest. A mapped enum such as `pending`
or `done` cannot replace other state. Supplemental JSON or related rows MAY retain information that
domain columns cannot express, but they participate in the same reconstruction and atomic commit.

The reconstruction obligation includes definition and runtime identities, active configuration,
variables and typed values, counters, history, faults, every ready and deferred entry with its
complete envelope and placement, and, when a checkpoint is used, revision, receipts, migration
audit, replay/retention evidence, root tombstone, pending and terminal effects, effect tombstones,
and outbox ordering. It applies to completed, faulted, unhandled, deferred, migrated, and tombstoned
results as well as the happy path. An application row is never silently archived, deleted, or
rewritten because it is outside the selected set or cannot fit a domain enum. An application archive
participant under §22 is separately identified and declared; this contract does not include
application rows in a Determa-owned archive.

A configured mapping MUST validate its selected row identities, expected shape, declaration/type
correspondence, and lossless capacity before constructing a new caller input from mapped row values
or mutating a row. It MUST verify exact round-trip reconstruction of each proposed result before
commit. If a value or field cannot be represented or reconstructed, it returns
`projection_not_lossless` . Truncation, lossy number conversion, default insertion, sorting of an
accepted noncanonical artifact, omission of an unknown required field, or substituting an enum
summary is forbidden. Concrete table names, ORM models, column layouts, indices, and helper rows are
host configuration, not portable format.

### 20.2 Typed input and external refresh

The application supplies an explicit selected-row snapshot and typed caller request. The projection
MAY read mapped application fields as §16.2 typed values and submit them only through a
declaration-supported create binding, declared input envelope, or the reserved §6 `env` envelope for
declared root external variables. The selected definition determines the accepted name, type,
target, and timing. `env` is the sole undeclared host-input exception. Its selected handler's
`refresh` action copies only the requested fields; a missing `refresh.only` field can produce a
committed `action_fault` under the ordinary RTC rule. A projection cannot edit a variable, active
state, history, fault, mailbox, identity, receipt, effect, or checkpoint field directly. A mapping
MUST reject an undeclared mapped field, type mismatch, or unsupported boundary before constructing a
new core request or writing a row. Native ORM objects, database handles, SDK objects, and
credentials are never machine values. Mapping a host numeric value MUST preserve integer versus
float identity; converting through an untyped JSON number is insufficient. Once an immutable caller
delivery exists, §17.4 admission precedence applies: after batch shape/root/duplicate checks,
retained `event_id` replay or conflict is decided from the exact request digest before declaration
and payload validation. A changed payload reusing a retained `event_id` returns `event_id_conflict`
even if its type is wrong for the current declaration; an equal replay returns its retained receipt
without core evaluation or row mutation. Mapping validation of row-derived values cannot reorder
those checks.

### 20.3 Application-owned transaction

An embedded facade invocation selects one root and mapping, validates the supplied typed request and
configured capability, then reads the selected rows and complete prior Determa artifact under the
application's transaction or an explicitly weaker ephemeral arrangement. It reconstructs and
validates the prior artifact before any core call. It calls the applicable §8 `create` , `admit` ,
or `step` operation on an in-memory candidate; external refresh uses an admitted `env` envelope and
subsequent `step` . It stages the exact operation-specific §8 result: `create` has state, emissions,
lifecycle dispositions, fault, and rejection; `admit` has accepted, state, and rejection; `step` has
disposition, state, emissions, lifecycle dispositions, fault, and rejection. Each also retains its
status. The facade also stages the corresponding complete checkpoint evidence when used and selected
application-row changes. It verifies the proposed round trip before one commit. The result exposed
as committed is returned only after that commit; a precommit result is a candidate, not a durable
receipt.

When the configured execution store proves `shared_application_transaction` and the application uses
its one native transaction, selected row changes and the complete §17 checkpoint replacement
(including receipts and outbox) MUST commit atomically. The root's exact revision/digest guard and
§17 replay/conflict ordering still apply. If the application cannot join the store transaction, the
facade MUST NOT claim `shared_application_transaction` or exactly-once committed application-row
effects; an explicit application inbox/outbox coordination protocol may make a different, separately
proved claim. `lossless_projection` alone implies neither durability, compare-and-swap, shared
transactions, transport acknowledgement, nor authority over a scope. Projected rows grant no access
to another root or scope.

Any failed validation, projection, core invocation, transaction, or revision guard before commit
leaves selected application rows and the prior checkpoint, receipts, and outbox unchanged under the
claimed atomic transaction. A committed `faulted` or `unhandled` core disposition remains an
ordinary complete result under §17 and MUST NOT be mistaken for an invocation failure. A conflict is
reported with the existing `checkpoint_revision_conflict` ; the caller may start a new invocation
from the newly committed state. The facade does not automatically reevaluate or retry. An explicitly
installed impure native runtime provider may perform external I/O during evaluation, before commit.
A database rollback or conflict cannot undo that I/O, so such an invocation cannot claim pure
evaluation, deterministic replay, or safe automatic retry without a separately proved provider and
caller policy (§11.5). Committed intents are delivered only according to the separately selected
outbox and transport profile.

The closed projection-specific failures are `invalid_projection_selection` (missing, ambiguous, or
mismatched selected rows or mapping identity), `invalid_projection_input` (undeclared field, invalid
typed value, or type mismatch), `unsupported_projection_boundary` (no declared create/input/refresh
path for the requested change), `projection_not_lossless` (prior or proposed artifact cannot be
exactly reconstructed), and `projection_transaction_unavailable` (a requested shared transaction
cannot be provided by this configured composition). These errors have no application-row,
checkpoint, receipt, or outbox commit. Existing core, artifact, capability, and checkpoint codes
retain their meanings and precedence: selection and capability validation precede decode; new
row-derived value validation precedes constructing a caller request; an already constructed caller
delivery follows §17.4 identity replay/conflict ordering before declaration/payload checks;
candidate round-trip validation precedes commit; the revision guard is checked at commit. Storage or
provider exceptions are surfaced as host failures without relabeling them as successful
dispositions.

The following cases are normative. `typed` denotes the exact §16.2 projection, and `prior` denotes a
valid complete aggregate or checkpoint for the selected root.

| selected application mapping and invocation | required outcome |
|---|---|
| `status = pending`, supplemental complete `prior`; new declared `amount` input mapped from a row as `typed = ["integer", "7"]` | Construct the valid caller request; preserve the applicable operation-specific §8 result and complete artifact; commit selected row and checkpoint together only when one native transaction is proved. |
| A valid `create` or `admit` completes through the facade | Return its exact §8 result fields; do not invent a `disposition` for either operation or `emissions` for `admit`. Preserve any emissions from `create` and subsequent `step` in their own results and committed evidence. |
| Same mapping reads `typed = ["float", "401c000000000000"]` from a row for a new integer input | `invalid_projection_input` before constructing a caller request; no core call or commit. |
| A valid selected-row snapshot and prior checkpoint retain an `event_id`; its equal complete delivery is presented again after a declaration change | Return its retained §17 receipt read-only before new declaration/payload checks; no core call or row mutation. |
| A retained `event_id` is reused with a changed `amount` payload `typed = ["float", "401c000000000000"]` that is invalid for the current integer declaration | `event_id_conflict` under §17.4 before payload validation; no core call or commit. |
| An admitted `env` envelope has `changed.amount = ["integer", "7"]` for a declared external root variable; the selected handler executes `refresh: {}` | Apply the §6 refresh action during `step` and preserve the exact §8 step result, including any disposition and emissions. |
| An admitted valid `env` envelope omits a field named by the selected handler's `refresh.only` | Preserve the §6 committed `action_fault` step result and its checkpoint evidence; do not recast it as pre-step `invalid_projection_input`. |
| Mapping asks to set `active_state = done` without a declared input/refresh transition | `unsupported_projection_boundary`; no core call or commit. |
| Enum-only `status = pending` has no storage for a deferred envelope, receipt, or pending effect present in `prior` | `projection_not_lossless`; retain `prior` and every row unchanged. |
| Proposed result contains a deferred envelope but configured supplemental storage truncates its payload | `projection_not_lossless`; no row, checkpoint, receipt, or outbox commit. |
| Selected row belongs to a different root or is ambiguous | `invalid_projection_selection`; no core call or commit. |
| Mapping requests one native application/checkpoint transaction from a store without that proved capability | `projection_transaction_unavailable`; no core call or commit. |
| Concurrent writer changes the checkpoint revision after candidate evaluation | `checkpoint_revision_conflict`; roll back all selected rows and checkpoint changes, with no automatic retry. |

## 21. Lossless event delivery profile

### 21.1 Boundary and portable delivery values

This optional profile composes an ingress source, the §8 foreground core, an optional §17 execution
store, and an outbound destination. It changes no machine grammar or core clock behavior. Its closed
wire values are `schema/delivery-v1.schema.json` ; version 1 is the sole version. An application may
use the base in-memory API without a broker, worker, coordinator, daemon, or timer. That API MUST
return every admission rejection, ready/deferred placement, terminal machine disposition, lifecycle
disposal, and emitted intent to its caller. The caller owns persistence and may claim only the
guarantees it actually supplies. The complete core `step` result in §8 and
`schema/core-step-result-v1.schema.json` is the base result shape: its state, disposition,
fault/rejection, emissions, and every `lifecycle_dispositions` member are mandatory fields. A caller
MUST preserve or explicitly decide every returned external intent and lifecycle disposition before
claiming lossless delivery. In a durable host, §17 receipts and records supply the machine evidence;
§18 authority guards, §19 effect journals, and §20 application projections apply only when the host
declares those optional profiles. A delivery adapter MUST NOT infer authority, durable processing,
or application success from a portable envelope or provider name. §19 result-event admission is a
host-owned journal recovery operation. It uses the same §17 aggregate admission boundary but has no
external source item to acknowledge under this section.

Before ingress admission, a source item has an immutable `source_scope` , `source_delivery_id` , and
original content. The content is exactly one of `original_bytes_base64` (RFC 4648 padded standard
Base64 of the original bytes) or `canonical_transport_value` (a §16.2 typed value). A decoded
transport value MUST be canonical before it is used; arbitrary SDK/protobuf/HTTP objects stay inside
the adapter. Base64 is canonical only when strict decoding followed by RFC 4648 standard encoding
reproduces the exact input string, including padding. Non-zero unused pad bits, missing or excess
padding, whitespace, alternate alphabets, and nonalphabet characters are `malformed_delivery` before
digest comparison, admission, or source acknowledgement. An adapter encoding original bytes MUST
produce this canonical form; it cannot hash a different spelling that decodes to the same bytes. Its
content digest is exactly:

```text
source_content_digest = hash([
  "determa-delivery-source-content-digest-1", "1",
  source_scope, source_delivery_id, content_kind, content_value
])
```

`hash` is the §9 SHA-256/JCS construction. The source identity is the pair (`source_scope`,
`source_delivery_id` ); it is distinct from the machine `event_id` . Adapters MUST retain the pair
and digest across attempts. A replay of the pair with different content is
`source_delivery_id_conflict` and cannot overwrite earlier evidence. A successfully normalized
envelope retains its entire §6.1 value, projected with §16.2 typed payloads and decimal-string
integer fields, through every accepted ready/deferred location. A durable adapter binds the source
pair and digest to the checkpoint acceptance receipt or durable terminal-transfer record before
source acknowledgement. The binding is host evidence outside the portable checkpoint; it MUST be
restored with that checkpoint for a broker-integrated profile. For that profile, an admitted binding
and its checkpoint admission MUST commit in one atomic host transaction. If the host cannot commit
both, it cannot claim lossless broker-integrated ingress. The source may continue to hold an
unacknowledged copy after commit, but replay then returns the already committed binding and never
repeats admission. The admitted binding's exact digest is
`hash(["determa-admission-binding-digest-1", "1", binding_without_admission_binding_digest])` . Its
`envelope_digest` MUST equal the matching mailbox entry and acceptance receipt request digest. A
dead-letter record itself is the terminal source binding. The first/replay vectors contain the exact
admitted binding.

### 21.2 Ingress decisions, acknowledgement, and replay

The source retains ownership until either (a) §17.4 admission commits the complete envelope,
acceptance receipt, and admitted binding, or (b) a configured durable terminal transfer commits a
§21.4 ingress dead-letter record. Only then may the adapter acknowledge the source. An
acknowledgement lost after commit may be retried from the retained binding without re-admitting or
re-dead-lettering. An adapter MUST NOT acknowledge on validation, reservation, remote call
initiation, a returned uncommitted core result, or a write whose durability is weaker than its
advertised profile.

The closed pre-commit decision is `source_owned` with one reason: `backpressure` ,
`capacity_unavailable` , `malformed_delivery` , `invalid_delivery` , `target_unavailable` ,
`precommit_failure` , `authority_unavailable` , `source_delivery_id_conflict` , or
`event_id_conflict` . It returns `acknowledge_source: false` and leaves source ownership intact. The
adapter may pause consumption or request source redelivery according to its configured source
contract; it MUST NOT turn a retry exhaustion, overflow, plugin failure, or target removal into
implicit discard. A rejected admission is not a terminal machine disposition. An explicit poison
policy may choose §21.4 instead, if its record can commit before acknowledgement. The decision
response identifies event identity when normalization succeeded; otherwise it identifies source
identity and content digest.

Committed admission returns `admitted` , `acknowledge_source: true` , the exact `event_id` , source
binding, acceptance receipt sequence, and checkpoint revision and digest. Equal source redelivery
returns that evidence unchanged. Equal machine-event replay follows §17.3; unequal machine-event
identity is `event_id_conflict` . For a host claiming §18 `authoritative_scope_fencing` , replay
checks never bypass the current-authority operation guard: a stale claimant may read history through
a separately authenticated read but cannot acknowledge, reactivate, or mutate delivery.
Authentication, authorization, and §18's exact guard failures retain their §18 result codes; a
delivery response MUST NOT replace them with `authority_unavailable` . That delivery reason covers
only an unavailable required guard before a guarded operation begins. An admission batch follows
§17.4's atomic order; a failed member leaves the whole batch source-owned. Broker acknowledgement is
per source item only after the batch commit is known durable.

After admission, exactly one runtime ready/deferred mailbox owns the envelope until a terminal §17.4
receipt owns its outcome. `deferred` is a live placement and cannot trigger a source retry.
`unhandled` , `faulted` , `disposed` , and `migration_disposed` are terminal machine outcomes, never
admission failures. The adapter MUST NOT ask the source broker to retry an admitted event because
the machine did not handle it. Each terminal decision retains event identity, exact reason or fault,
decision authority, receipt, and the profile's declared retention window. The base API returns that
evidence to its caller even without durable storage. The checkpoint-backed `admitted` ,
`machine_disposition` , `mailbox_placement` , and `outbound_decision` responses each name an exact
committed checkpoint revision and digest. A pre-commit `source_owned` response claims no checkpoint
mutation; `ingress_dead_lettered` instead names its separately committed durable terminal record and
source receipt under §21.4. `admitted.evidence.operation_kind` is `acceptance` and its
`receipt_sequence` resolves only to an acceptance receipt at that revision. The receipt's event id,
acceptance sequence, and request digest equal the response's event id, evidence acceptance sequence,
and evidence envelope digest; the complete envelope occupies one ready mailbox entry at that
snapshot. The admitted source binding names the same acceptance evidence. A `machine_disposition`
instead requires `event_terminal` evidence resolving only to a terminal event receipt. Its event id,
acceptance sequence, final queue sequence, request digest, complete outcome, and resulting aggregate
digest equal the response and checkpoint; the event is absent from live mailboxes. Substituting a
creation, acceptance, or other receipt kind is `invalid_delivery_evidence` . The exact first
admission, equal replay, and terminal unhandled witnesses are
`vectors/delivery/delivery-v1-cases.json` and
`vectors/delivery/execution-checkpoint-transfer-v1.json` .

`mailbox_placement` names the committed checkpoint and the exact live entry's event id, envelope
digest, target runtime, acceptance sequence, current queue sequence, and ready/deferred location.
Its `origin` resolves either to that host event's retained acceptance receipt or to its internal
emission's producing operation receipt. A deferred move or structural recall allocates a new queue
sequence but no operation receipt (§17.4). The origin receipt remains unchanged. No adapter may
invent a terminal receipt for a live placement. Wrong origin kind, missing origin, or unequal entry
identity is `invalid_delivery_evidence` . The complete before/deferred/recalled snapshots are
`vectors/delivery/queue-placement-checkpoints-v1.json` . Evidence mismatch rejects the response
without acknowledging or dropping work; durable hosts quarantine a corrupt committed snapshot until
its exact owner evidence is repaired.

### 21.3 Ordering, pressure, and restoration

Within a runtime, committed admissions append in caller batch order and §6.7 owns ready/deferred
order thereafter. The adapter publishes its pre-admission ordering capability separately:
`source_ordered` means it admits one bound source stream in source order; absence of that claim
means `unordered` and promises no source order. Neither implies ordering between roots, different
sources, or an emitted outbound intent and a later external input. A source-order claim requires
proof that concurrent workers, retry gaps, dead-letter transfer, and recovery cannot overtake an
earlier owned source item. Backpressure stops or defers new admission while keeping the source item
available; it cannot evict an accepted mailbox member. The core's deferred-capacity overflow
produces the terminal fault receipt in §10.1 and is not a transport discard.

A durable broker-integrated restore MUST restore checkpoint, source bindings, unacknowledged
external backlog or committed dead-letter records, and unresolved outbox work at one valid
consistency point. Missing any owned item or binding invalidates the claimed profile; the host MUST
quarantine or refuse activation until repaired. A changed source item at a retained identity is a
conflict. An authority provider alone grants no guard. When the host claims §18
`authoritative_scope_fencing` , it MUST hold the §18.3 scope operation guard through the native
commit of admission, source binding, ingress dead-letter transfer, and acknowledgement evidence.
Replay-triggered acknowledgement requires the same current guard. If its §18 host profile reports
`worker_fencing: true` , continuation by a hosted worker checks its active claim, authenticated
principal, epoch, attempt fence, and expiry under §18.4. The trusted host clock is used only for
that optional claim expiry; it is never a core clock or delivery retry timer. A lease alone does not
prove retirement of an old worker. A standalone in-memory host declares no cross-host fencing
guarantee.

### 21.4 Durable ingress dead letters and terminal policy

`durable_ingress_dead_letter` is an ingress-adapter capability, never an execution-store capability.
It requires one immutable record with source scope, delivery id, original content, content digest,
optional normalized event id and envelope digest, reason code, decision authority, retention
profile, and terminal receipt identity. The record digest is:

```text
ingress_dead_letter_digest = hash([
  "determa-ingress-dead-letter-digest-1", "1",
  record_without_ingress_dead_letter_digest
])
```

The dead-letter record and source binding MUST commit durably before the source is acknowledged. Its
content is the original bytes or canonical transport value, never only a payload-free error string.
Equal redelivery returns the same terminal receipt; unequal content conflicts. A dead-letter
destination that merely accepted a send is insufficient unless it durably accepted responsibility
and supplies a recoverable receipt. If durability cannot be established, the source remains owner.
Retention MUST preserve content and replay/conflict evidence for the profile's declared window;
compaction or expiry is allowed only under an explicitly weaker published policy that still
preserves identity and digest evidence throughout its replay window. No lossless profile permits
silent discard, including on fault, queue overflow, retry exhaustion, target deletion, or restore. A
deliberate discard is a terminal decision only under an explicitly weaker policy and MUST have
event/source identity, reason, decision authority, and retained receipt evidence before ownership
transfer.

### 21.5 Outbound responsibility

Each committed external intent enters §17.6 pending outbox state, or the base API returns the
complete intent for caller-owned persistence. A durable worker retries the same `effect_id` and
scope after `retryable_failure` or `ambiguous` ; uncertainty MUST NOT create a new effect identity.
Terminal `confirmed` means the destination durably accepted responsibility under its configured
contract. It does not prove execution or business success. A later declared input event is the only
machine-visible remote result. Definitive rejection, operator cancellation, declared discard, and
dead-letter transfer use §17.6's exact terminal states with reason and complete intent; a durable
dead-letter claim also retains the destination's durable transfer receipt. In the closed §21
outbound decision, `reason_code` is non-null exactly for retryable failure, ambiguity, permanent
rejection, operator cancellation, discard, and dead-letter transfer. `destination_receipt_id` is
non-null exactly for confirmed durable destination acceptance or durable dead-letter transfer. A
`dead_lettered` decision with either field null is invalid and cannot end outbound responsibility.
Retrying an outbox delivery after its terminal state returns its retained record and does not send
that intent again. `outbound_decision.evidence` names the exact committed checkpoint revision and
digest plus the effect's §17.6 outbox location. A pending decision resolves to the complete pending
intent, its `state_revision` , and the same `delivery_state` ; a terminal decision resolves to one
complete terminal outbox record, its `terminal_sequence` and `committed_revision` , and the same
`outcome` . The effect id must resolve to the producing operation receipt's external-outbox emission
reference, and the outbox record retains the complete intent. An outbox state update increments
checkpoint revision but allocates no operation receipt; the destination acceptance or dead-letter
receipt is provider-owned evidence, not a checkpoint operation receipt. For `confirmed` and
`dead_lettered` , the host retains one closed `outbound_destination_receipt` whose root, effect id,
terminal sequence, outcome, reason, destination receipt id, and checkpoint digest agree exactly with
the terminal outbox record and response. Its digest is
`hash(["determa-outbound-destination-receipt-digest-1", "1", record_without_outbound_destination_receipt_digest])`
. The record binds the adapter's proof; the configured destination must actually durably accept
responsibility for the claimed outcome. The two exact records are
`vectors/delivery/outbound-destination-receipts-v1.json` . Missing or wrong effect, location, state
revision, terminal sequence, or destination receipt is `invalid_delivery_evidence` . Exact pending,
confirmed, and dead-letter snapshots are `vectors/delivery/outbound-checkpoint-lifecycle-v1.json` .
When §19's native-effect profile is selected, an outbox `confirmed` record may coexist with an
`unclaimed` , `leased` , or `ambiguous` invocation. It does not supply a terminal business outcome,
cancel the invocation, or authorize a provider retry. The §19 journal retains the business outcome,
cancellation decision, and attempt evidence separately. An `outcome_recorded` result whose admission
was interrupted remains host-owned recovery work: the host admits its exact pinned declared event
under §19.4's current scope guard and checkpoint/journal transaction, without requiring the old
worker claim or calling the provider again. A late or ineligible target follows §19.4's explicit
reconciliation rule, never an ingress source retry or implicit discard. The lossless profile cannot
silently remove unresolved outbound work on completion, cancellation, restore, or worker failure.

## 22. Portable archives and declared participants

### 22.1 Boundary and identity

An archive is a complete snapshot of selected Determa-owned root checkpoints, their referenced
immutable definitions, and the declared participant closure at one host-established consistency
point. It is not an arbitrary database export. Sections 18 (authority), 19 (effects), 20
(application projection), and 21 (delivery) own their respective active protocols; this section
specifies only their explicitly declared portable archive data. An application projection under §20
is a participant only when its separate archive contract is declared. No core helper, timer,
database service, or network worker is made mandatory here. Standalone takeover, cloning,
relocation, rebinding, fencing, and scope transfer require a separate contract and are not archive
import operations.

The version-1 archive is strict UTF-8 JSON with exact `archive_format: "determa.scope_archive"` and
`archive_schema_version: 1` ; its closed schema is `schema/archive-v1.schema.json` . Its component
is the closed `schema/archive-participant-v1.schema.json` . Unknown formats and versions fail before
semantic validation with `unsupported_archive_format` and `unsupported_archive_schema_version` .
Unknown participant formats and versions fail with `unsupported_archive_participant_format` and
`unsupported_archive_participant_schema_version` . A recognized malformed artifact fails
`invalid_archive` ; a wrong digest fails `archive_digest_mismatch` . There is no version-1
compatibility conversion or best-effort unknown-field preservation.

The selection names an ordered set of root identities in one authorized source context, including an
explicit standalone context. The archive `source` records the nonsecret logical source scope
identity and ownership-binding digest when present, plus its generation when one is proved. For
explicit `standalone` provenance, all three are null and profile claims are empty. The
`profile_kind` , sorted `profile_claims` , and digest state what source profile the exporter
actually proved; they are descriptive and confer no access. The `profile_claims` vocabulary is
closed by the archive schema and names only proved §17, §18, §19, or §20 host/profile guarantees.
`durable_native_results` is the §19 source disclosure that makes host journal participation
required; no matching claim is inferred from the mere presence of an outbox intent. The principal,
credentials, authorization policy, and physical scope-isolation key remain host metadata outside
portable bytes. An importer compares this source identity, binding, generation and complete profile
against independently trusted configured source policy; it never trusts a profile claim merely
because archive bytes hash.

The exporter MUST prove that every root in the selection has exactly one checkpoint, including
tombstones, and that every owned runtime and component is represented by its checkpoint aggregate
under §§7.1–7.2. The archive contains the exact complete §17 checkpoint for each selected root. Thus
active configurations, typed variables, histories, owned identities, ready and deferred mailboxes,
counters, faults, external intents, pending and terminal outbox work, receipts, replay retention,
migration audit, and root tombstones survive without inference. An empty root selection is invalid.
Selection completeness is relative to its declared roots; it does not claim a complete deployment or
unrelated application data. The exporter records the checkpoint revision/digest it read and MUST
abort if any selected checkpoint changes before the consistency point is secured. Application and
host-journal participants needing a shared point MUST join the same verified boundary; a readable
snapshot alone does not prove that boundary. When the optional §18 `consistent_scope_inventory`
profile is claimed, its inventory and frozen point MUST be proved against authoritative storage,
including required host journal records. A freeze response or evidence digest is checked by
resolving the linked retained §18 authority-ledger record and its committed evidence; the response
hash alone is never a grant. The archive records no active worker claim, epoch authority, retirement
proof, credential, or destination-bound grant. Import cannot mint or restore any of these from
`consistency_token` , checkpoint bytes, or an authority response.

If the selected source used §19 durable native results, its complete closed §19 host effect journal
for every selected participating root MUST be a required archive participant under a separately
pinned schema and provider. The independent source policy binds that journal participant ID into the
trusted required set; an artifact that deletes the journal or removes the ID from its own contract
still fails. Export and import verify each journal digest and its exact checkpoint revision/digest
pair at the same capture point. The participant payload is a canonical typed value with a closed,
pinned participant schema; decoding it MUST yield exactly one closed §19 journal for each selected
root covered by that source profile, in root-ID order. Each decoded journal MUST validate against
`schema/host-effect-journal-v1.schema.json` , including its root ID, source scope identity, complete
records and references, and journal digest. The same typed participant payload MUST include the
exact normalized public host operation response bytes for every journal response reference, ordered
by operation ID; each reference digest MUST equal
`hash(["determa-host-operation-response-1", normalized_response])` . A response digest alone does
not reconstruct equal replay. An empty journal is valid only when the verified source inventory
proves that root has no §19 native-effect records or response references at the capture point.
Missing, duplicate, or torn journal evidence blocks the operation. §19 active worker claims,
credentials, and authority records remain outside the archive; their historical outcomes and
result-admission evidence remain in the journal payload. A source that has only §17 outbox intents
and does not claim §19 needs no journal participant.

`schema/archive-host-journal-inventory-v1.schema.json` closes the separate source inventory
commitment. Its `source_profile_digest` binds the independently verified source kind, logical scope,
binding, generation, and profile claims; its `consistency_token` binds the capture point. Its
ordered root entries bind each selected checkpoint pair, journal revision and digest, complete
effect IDs, intent digests, attempt-report digests, outcome and result IDs, and operation response
IDs and digests.
`inventory_digest = hash(["determa-archive-host-journal-inventory-1", inventory_without_inventory_digest])`
. Export obtains this inventory from the authoritative source under the same consistent capture
boundary as checkpoints and journals, not by enumerating the archive participant after capture.
Stage compares the complete decoded participant against independently trusted source inventory
evidence configured for that exact source profile, selection token, and root set. An importer MUST
NOT accept an inventory supplied only by the archive or by an untrusted caller. A correctly resealed
deletion or rewrite of a journal record, attempt, outcome, or response reference is
`archive_host_journal_inventory_mismatch` with the journal participant ID. This evidence proves
historical capture completeness; it is not a worker claim, credential, authority grant, or
permission to resume an inherited attempt.

### 22.2 Closed content and digest rules

The version-1 archive root is its closed manifest and has exactly `archive_format` ,
`archive_schema_version` , `source` , `participant_contract` , `selection` , `checkpoints` ,
`normalized_definitions` , `migration_descriptors` , `participants` , `members` ,
`required_determa_capabilities` , `optional_participant_references` , `source_fence_reference` ,
`transfer_reference` , and `archive_digest` . `archive_format` is exactly `determa.scope_archive` ;
there is no reader for the earlier draft spelling. The outer `archive_digest` is the exact content
identity of this archive, rather than a second mutable ID field. The manifest's `source` and
`selection` bind logical scope, source binding and generation when present, profile, and consistency
point. An explicit standalone source has null scope, binding, generation, fence, and transfer
references.
`source.profile_digest = hash(["determa-archive-source-profile-1", source_without_profile_digest])`
. `participant_contract` lists the complete declared required and optional participant IDs, each
sorted by UTF-8 bytes and disjoint. Its digest is
`hash(["determa-archive-participant-contract-1", source.profile_digest, required_participant_ids, optional_participant_ids])`
. Every included participant appears exactly once in that contract and its `required` Boolean agrees
with the corresponding list. Every required participant appears in `participants` ; an optional one
may be absent with an explicit result report. The contract is checked against an independently
trusted configured requirement for this source profile and selection. A resealed archive cannot
delete a required helper, host journal, or application participant by editing its own contract. The
importer MUST refuse source profile or contract mismatch even when all archive hashes verify.

`members` contains one entry for every checkpoint, normalized-definition attachment, migration
descriptor, and included participant record, and no other entry. Identity is respectively
`checkpoint:<root_instance_id>` , `definition:<validated_bundle_fingerprint>` ,
`migration_descriptor:<migration_descriptor_digest>` , or `participant:<participant_id>` . Entries
are strictly sorted by identity UTF-8 bytes and unique. Each entry's `digest` is SHA-256 of the
exact UTF-8 RFC 8785 JCS bytes of the whole member object; `byte_length` is the canonical positive
decimal count of those same bytes. The importer verifies identity, length, digest, complete closure,
and each member's own nested digest. The outer digest includes this member table; it is not a
substitute for comparing it to actual member bytes. For an external participant the member object
includes its external content reference, and staging additionally resolves and verifies the
referenced typed payload bytes.

`required_determa_capabilities` is the unique UTF-8 sorted exact minimum needed to decode and stage
the included Determa-owned content. It includes `portable_archive` , plus `portable_checkpoint` ,
`normalized_definition` , `migration_descriptor` , and `archive_participant` when the corresponding
members exist; a source claiming §19 durable native results also requires `host_effect_journal` .
These are destination requirements, distinct from `source.profile_claims` , which describe source
behavior. The exporter computes them from the captured content and profile; the importer checks
their exact closure and that its independently configured supported-capability set contains them
before any staging write. A missing or invented requirement in an otherwise resealed manifest is
`archive_manifest_mismatch` ; an unsupported genuine requirement is `archive_capability_mismatch` .

`optional_participant_references` is sorted by participant ID and contains one exact ID, §11.5
provider reference, and schema digest for each optional ID in the declared contract, even when its
payload was absent at capture. Its complete list is checked against independently trusted configured
references and source capture evidence; absence still appears in the result.
`source_fence_reference` and `transfer_reference` are null unless the source has an applicable
retained §18 operation. A nonnull fence reference carries only the retained operation kind, ID, and
response digest; a transfer reference additionally carries the destination binding digest. They are
public provenance pointers, checked against exact source profile and trusted retained host evidence
when used. A hash or operation ID in the archive cannot itself prove source retirement, convey a
credential, consume a grant, or authorize activation; §18 and the separate recovery contract perform
those checks.

`selection` contains `root_instance_ids` , ordered strictly by UTF-8 bytes, and `consistency_token`
, an opaque non-empty exporter assertion. The token is not authority, a lease, or a transaction
credential. Checkpoints are ordered by `root_instance_id` , match selection exactly, pass §17
validation and their own digests. The definitions array uses the exact §16.13 attachment shape and
is ordered by fingerprint; descriptors are ordered by digest. Every referenced definition, including
origin/current definitions and fault anchors, every descriptor named by a retained migration audit
record, and every descriptor needed by the exporter's declared recovery route MUST be attached.
Attachments are independently hash-checked and trust-admitted before import. Digest-equal
attachments may appear once; conflicting or missing attachments invalidate the archive. The archive
does not invent a route or migrate a checkpoint while exporting or staging.

Participant records are ordered by `participant_id` and are unique. Each has exactly
`participant_format` , `participant_schema_version` , `participant_id` , `required` ,
`provider_reference` , `participant_schema_digest` , `dependencies` , `storage` , `payload_digest` ,
and `payload` . `provider_reference` is §11.5 exact identifier/version/content digest. Dependencies
are unique participant IDs in ascending UTF-8 order, must exist in the archive, and must form an
acyclic graph. A required participant makes all its dependencies required in effect. `storage` is
`embedded` or `external` . Embedded payload is the complete typed §16.2 projection; external payload
is null and names immutable bytes by `payload_digest` through the importer's configured
content-addressed resolver. Both modes must reconstruct the same declared typed value exactly; an
external reference is never a permission to omit required bytes. The payload is participant-owned
and may represent application rows, a helper, or a host journal only under its separately versioned
closed schema and provider contract. The archive itself does not interpret that data or confer
authority.

All JSON uses §16.2 strict parsing and RFC 8785 JCS serialization. Typed payloads use exactly its
seven typed forms, canonical map ordering, and numeric constraints. `hash(value)` is the lowercase
`sha256:` prefix plus SHA-256 of the UTF-8 JCS bytes of `value` , as in §16. Participant
`payload_digest = hash(["determa-archive-payload-1", participant_id, participant_schema_digest, typed_payload])`
; an external resolver must return the typed payload bytes that reproduce this digest. The
provider's exact schema bytes reproduce `participant_schema_digest` using `hash(schema_json)` and
its registered schema validates the decoded payload. The outer
`archive_digest = hash(["determa-archive-digest-1", archive_without_archive_digest])` . Digest
checks do not replace schema, semantic, provenance, or authorization checks. The archive's schema
bytes are pinned by the importer's supported version-1 schema registry; a changed shape with the
same version is unsupported, even if a JSON parser accepts it.

### 22.3 Export, staging, and declared capability

Export uses the closed `schema/archive-export-request-v1.schema.json` request: exact source
provenance, selected roots, required and optional participant IDs, and consistency token.
`schema/archive-export-source-v1.schema.json` closes the source checkpoint, definition, descriptor,
participant-capture, and inventory-evidence input used in the normative vectors. Its inventory
evidence is a host assertion that must be proved against source storage; the JSON shape alone cannot
prove completeness. The host selects one authorized source context and exact roots. Before reading
data, it resolves every selected participant's exact `archive_participant` provider and checks its
healthy configured-instance `portable_export` , `exact_reconstruction` , and
`consistent_archive_capture` claims. The `consistent_archive_capture` claim means this configured
participant can capture its complete declared payload at the host-selected checkpoint consistency
point; it does not establish that point or provide scope authority. `portable_export` and
`portable_import` mean exact closed payload production and staging, respectively;
`exact_reconstruction` means all declared participant state can be recovered from that payload and
its verified external bytes. A participant MUST declare its dependencies, schema digest, capture and
reconstruction method, and whether its payload includes external bytes. The host expands
dependencies and rejects cycles, duplicate identities, missing required providers, unsupported
schemas, unprovable completeness, or an inconsistent boundary. An optional participant that is
absent is listed in the export result's `absent_optional_participants` ; it is never silently
treated as included. A present but unsupported optional participant is reported as
`unsupported_optional_participants` and excluded only when no included required participant depends
on it. The closed result schema is `schema/archive-result-v1.schema.json` . A success result has
exact format/version, `status` (`exported` or `staged` ), `archive_digest` , `staged` ,
`included_participant_ids` , `absent_optional_participants` , and
`unsupported_optional_participants` . A refusal has exact format/version, `status: "refused"` , one
`code` , nullable `participant_id` , `staged: false` , empty `included_participant_ids` , and the
two optional-report arrays. Reports contain exact `participant_id` and one closed `reason` . All ID
arrays are UTF-8 ordered and unique. Export success is `exported` with `staged: false` ; import
success is `staged` with `staged: true` .

Import uses the closed `schema/archive-import-request-v1.schema.json` request. Its trusted source
profile and participant-contract digests, expected source provenance, and staging identity are
supplied by independent host policy, not copied from the untrusted archive. A caller cannot weaken
that policy by changing request fields. The host verifies these values and authorized staging
selection before any write. Import verifies the complete archive manifest, member bytes and required
Determa capabilities, checkpoint and attachment digests, trust policy, and each included
participant's exact provider fingerprint, transitive executable dependencies, schema bytes/digest,
payload, and configured-instance `portable_import` and `exact_reconstruction` claims before any
write. An included external payload must be resolved and hash-verified during staging; required
missing bytes, providers, schemas, dependencies, or capabilities block the entire import. Optional
participants may be absent only when the request explicitly permits their omission; the result names
each absence or unsupported participant. If a present optional participant is omitted, no dependent
participant may be staged. An importer MUST NOT substitute a provider with the same name and a
different digest or infer that a helper is disposable. An unsupported helper is reported by ID, with
its required/optional status, and cannot disappear without a result record.

Successful import produces an inert staged archive with exact source bytes and verified attachments.
Included participants are fully verified. An omitted optional participant remains an identified
inert record whose provider-dependent data is not available for reconstruction until separately
validated; omission never authorizes activation or deletion of its bytes. Staging creates no active
execution-store scope, root identity claim, credential, ingress subscription, outbox worker, broker
acknowledgement, effect delivery, or authority grant. It does not invoke core `create` , `admit` ,
`step` , or migration, and does not replay a journal or event deltas to reconstruct state. An event
journal or incremental delta MAY supplement a complete snapshot for provenance or later
synchronization, but cannot replace any checkpoint or participant snapshot. Activation and
destination ownership are outside this section. A failed export or stage returns one closed refusal
and leaves source and destination unchanged.

The deterministic refusal precedence is unsupported archive format, unsupported archive version,
invalid archive shape, archive digest mismatch, manifest closure, source provenance, source profile
or participant contract mismatch, invalid checkpoint or attachment, invalid participant closure or
payload, host-journal inventory mismatch, missing required artifact or provider, capability
mismatch, then consistency or staging failure. The result uses one exact code from
`schema/archive-result-v1.schema.json` ; implementations may attach nonportable diagnostics outside
the result. Required absence is never downgraded to an optional report. Conformance MUST compare
complete results and staged bytes, including negative cases for changed checkpoint/mailbox,
receipt/outbox/audit/fault omission, wrong schema or provider digest, resealed required-participant
omission or weaker source claim, absent optional versus missing required, unresolved external
payload, participant dependency cycle, and attempted activation during staging.

An internally wrong source-profile or participant-contract digest is `invalid_archive` . An intact
outer archive digest with a wrong member identity, order, byte length, digest, missing or extra
member, or incorrect required-capability closure is `archive_manifest_mismatch` . An intact archive
whose scope identity, binding, or generation differs from trusted source policy is
`archive_source_provenance_mismatch` ; a kind, claim, or profile digest difference is
`archive_source_profile_mismatch` . An intact, differently declared required/optional participant
set is `archive_participant_contract_mismatch` , even if the artifact was resealed. A required
participant absent from a matching contract is `missing_required_artifact` with its ID. An unknown
selected root or duplicate root selection is `archive_selection_invalid` ; a selected checkpoint
that changes or cannot be captured at the agreed point is `archive_consistency_unavailable` . These
are pre-commit host refusals, never core faults.

The normative `vectors/archives/` vectors include one four-root snapshot with live ready/deferred
entries, a retained fault, a compatible migration receipt/audit pair, and pending, full-terminal,
and compact effect evidence anchored to producing receipts. The closed export cases supply complete
requests and source captures, including selected checkpoint bytes, definition/descriptor
attachments, participant capture, and a host-asserted consistency point. The closed stage cases
supply complete import requests, trusted-policy fixtures, exact staged bytes or refusal, and
JSON-pointer differences from the positive archive. Conformance verifies the source/contract pins
and every nested digest and causal link, not merely the outer archive hash.

## 23. Optional external timer helper

### 23.1 Boundary and clock

This optional version-1 helper implements the §11.2 external timer role. It changes neither machine
format 1 nor core state, checkpoint, `step` , or admission rules. A host with no timer installation
performs no timer record read, due poll, heartbeat, clock call, scheduler start, daemon start, or
background work. The foreground API remains usable without this helper. Installation uses the §11.5
`timer` provider reference and configured-instance capability report. An endpoint scheme, provider
name, or event name confers no authority. Credentials, trusted clock, worker identity, storage
binding, scope authorization, and polling policy are host configuration.

Every absolute time in this interface has `clock_basis: "unix_nanoseconds"` and is a canonical
decimal string in signed-64-bit Unix-epoch nanoseconds `[-9223372036854775808, 9223372036854775807]`
. Every duration is a canonical nonnegative decimal string of nanoseconds at most
`9223372036854775807` . `-0` , a plus sign, a leading zero, fractions, exponents, JSON numbers, and
values outside the range fail `invalid_timer_time` before mutation. A duration schedule samples the
trusted host clock at acceptance and adds with checked arithmetic; overflow fails
`timer_deadline_overflow` . An absolute deadline may already be due. Equality is due:
`now >= deadline` . An unavailable or untrusted clock stops claiming and firing with
`timer_clock_unavailable` ; the core never reads it.

### 23.2 Commands, identity, and cancellation

The closed `schema/timer-helper-operation-v1.schema.json` defines requests and results. Each request
has `interface: "determa.timer_helper"` , `interface_version: 1` , `operation` (`schedule`, `cancel`
, `claim_fire` , `complete_fire` , or `read_timer` ), `operation_id` , `scope_identity` ,
`root_instance_id` , `root_runtime_id` , `timer_id` , `request_digest` , and closed
operation-specific `arguments` . `root_runtime_id` is the immutable §9 root identity of the target
incarnation. The helper verifies it against the authorized target; a recycled root name cannot
receive a prior timer. `operation_id` is unique within the authorized scope. `timer_id` is unique
within the scope/root/incarnation tuple and cannot be reused after a retained terminal record. The
request digest is `hash(["determa-timer-request-1", request_without_request_digest])` using §9.
Before scope lookup or clock access, an unknown interface returns the closed
`unsupported_timer_protocol` early result, then an unknown version returns
`unsupported_timer_protocol_version` , then an invalid recognized request returns
`invalid_timer_request` . These early results have null operation and identities, no record or
result digest, and no mutation. An equal replay returns the exact retained first result before
current preconditions; a different digest fails `timer_operation_conflict` . The host authenticates
and authorizes the scope and target before revealing existence or replay evidence. Unauthorized
requests return `unauthorized_timer_scope` without existence detail. A successful result has
`result_digest = hash(["determa-timer-result-1", request_digest, result_without_result_digest])` .
The helper retains the exact result with its committed operation receipt; this hash alone proves no
transaction fate.

`schedule` supplies exactly one of `deadline_at` or `delay_nanoseconds` , a declared root event
name, its complete §16.2 typed payload map, and a nullable correlation ID. A machine emits
scheduling/cancellation intent through ordinary declared `send` or an external effect integration
selected by the host. No new action keyword exists. The host validates the event declaration on
scheduling and again at admission. Acceptance persists the complete immutable request, computed
deadline, target incarnation, and stable fire identity before acknowledging the intent. The one-shot
`event_id` is
`hash(["determa-timer-fire-event-1", "1", scope_identity, root_instance_id, root_runtime_id, timer_id])`
; deadline, attempt, fence, worker, and clock time do not enter it. A different schedule for an
existing timer ID fails `timer_id_conflict` .

The independent `schema/timer-record-v1.schema.json` artifact has exact
`timer_artifact_format: "determa.timer_records"` , version 1, ordered records, retained operation
receipts, and
`timer_artifact_digest = hash(["determa-timer-artifact-1", artifact_without_timer_artifact_digest])`
. Records are `pending` , `claimed` , `fired` , or `cancelled` , with monotonically increasing
decimal revision and attempt fence. It is never a checkpoint member. `cancel` supplies expected
revision. It atomically changes only `pending` to `cancelled` . Claimed returns
`timer_fire_in_progress` ; fired returns `timer_already_fired` ; cancelled returns the unchanged
terminal result. A cancel cannot retract an admitted event. A conflicting cancel and fire resolve
through one record compare-and-swap or native transaction; the loser cannot claim success.
Independent transport may already hold a copied event.

### 23.3 Fire, recovery, and delivery limits

`claim_fire` requires current revision and an authenticated worker. The helper samples its trusted
clock to determine due status, then atomically changes `pending` to `claimed` and allocates a
strictly increasing canonical decimal `attempt_fence` . A retry after expiration or revocation may
allocate a new fence only after resolving the previous attempt's commit fate. A caller-supplied time
never makes a timer due. Cancellation before claim retains attempt fence `"0"` and no fire attempt,
analogous to §19's proven preclaim cancellation boundary. Once claimed, cancellation cannot assert
that no event was admitted; claim expiration alone gives no such proof. The host selects
`expires_at` in signed-64-bit Unix nanoseconds. At `now >= expires_at` , the claim cannot mutate;
expiry does not prove that an external fire was absent. `complete_fire` requires the current fence,
principal, revision, and exact event ID. A stale or unauthorized claim changes nothing. If §18
authority is installed and its guarded local write capability is proved, the current scope epoch and
`guarded_commit` protect the timer mutation alongside the helper's own fence. The §18 `fence_worker`
/ten-field `workerClaim` is effect-specific and is never serialized as a timer claim; no
`clock_basis` is added to it. An archived or expired helper claim is never portable authority.

For coordinated admission, `admission_receipt_digest` is non-null and equals
`hash(["determa-timer-admission-receipt-1", acceptance_receipt])` for the exact candidate §17
acceptance receipt with matching event ID and `request_digest` equal to the fired envelope digest. A
rejected candidate commits neither receipt nor digest; a successful completion verifies and commits
both. For independent delivery the field is null until that receipt is actually obtained; it cannot
stand for a pending source item. `vectors/timers/timer-committed-admission-v1.json` carries the full
committed checkpoint and this digest for the positive fire case.

A host claiming `coordinated_timer_admission` MUST commit transition to `fired` , timer receipt,
complete §17 acceptance receipt/checkpoint, and any §21 source binding in one native transaction, or
prove a recoverable protocol that reaches the same result after every crash. It acknowledges only
after that boundary. A precommit admission rejection leaves timer and checkpoint unchanged, returns
`timer_admission_rejected` , and preserves the immutable event for authorized retry. It creates no
§17 admission receipt or §21 admitted binding. Recovery after an uncommitted attempt reuses the same
event ID under a newly proved fence. After a committed fire, replay returns retained evidence and
cannot admit it again. The same event ID with different envelope content fails
`timer_event_conflict` . A §21 adapter uses `source_scope = scope_identity` and
`source_delivery_id = hash(["determa-timer-source-delivery-1", root_instance_id, root_runtime_id, timer_id])`
, retains the same canonical event content and source content digest across attempts, and
acknowledges its source only after §21's committed ownership transfer. Direct §17 admission is also
valid and has no §21 source receipt to invent. Both paths apply ordinary declaration, target,
payload, capacity, deferral, fault, and receipt rules.

`independent_timer_delivery` commits fire independently and hands the immutable event to a
configured delivery source. A crash gap is `delivery_ambiguous` until source or admission evidence
resolves it. This profile cannot claim atomic admission, no loss, exactly-once delivery, destination
success, or deadline precision. A durable helper guarantees recoverable accepted records and stable
identities only within its proved topology. It does not guarantee a live target or handled event. An
ephemeral helper may lose schedules on process loss. Closed timer claims are
`ephemeral_timer_helper` , `durable_timer_helper` , `independent_timer_delivery` , and
`coordinated_timer_admission` . Coordinated admission requires durable storage and proved
transaction/recovery integration. The configured instance and composed host prove claims; a
descriptor alone cannot. A configured helper declares exactly one of `ephemeral_timer_helper` and
`durable_timer_helper` , and exactly one of `independent_timer_delivery` and
`coordinated_timer_admission` . A missing or contradictory combination fails
`timer_capability_mismatch` before schedule acceptance.

### 23.4 Archives and conformance

For §22, a durable helper declares a separate `archive_participant` with its exact provider
reference and a schema digest for `schema/timer-record-v1.schema.json` . The archive participant
reference is pinned independently of the installed `timer` provider reference. Its configured report
must prove `portable_export` , `exact_reconstruction` , and `consistent_archive_capture` at export,
then `portable_import` and `exact_reconstruction` at staging. Timer capability claims alone satisfy
none of these checks. Its payload is the complete §16.2 typed projection of the timer artifact:
records, terminal identity, and replay receipts needed by its retention policy. The declared schema
digest is §22's `hash(schema_json)` of the exact timer-record schema bytes. The §22 participant's
`payload_digest` binds its ID, schema digest, and typed payload. Its `participant:<participant_id>`
manifest member binds canonical bytes, SHA-256, and byte length; `required_determa_capabilities`
includes `archive_participant` when it is included. Source provenance, required/optional participant
contract, and any optional participant reference are checked against independent host policy, so a
resealed omission cannot make outstanding timer work optional. Capture uses the same proved
consistency point as selected checkpoints and delivery bindings. The helper must enumerate complete
timer identities and retained replay evidence from authoritative storage for the selected roots; a
caller-provided list or artifact self-description alone is not completeness proof. If outstanding
timers must resume with a selected root, the participant is required in the §22 export request and
trusted source contract; its absence fails export or staging as `missing_required_artifact` or
`missing_required_provider` according to the §22 boundary. External payload storage must reconstruct
the exact bytes. Import stages inert data; it never activates a clock, worker, claim, scope
authority, or pending delivery. The host authorizes and reconciles staged records before resumption.
Unresolved fire fate blocks resumption. A helper is optional when no selected root depends on its
records. A base §16 aggregate or §17 checkpoint export remains available for inspection
independently of timer installation; it MUST NOT be represented as a resumable complete archive for
a root whose outstanding timer participant was omitted. Debug inspection reports timer records only
through the separately authorized helper view and reports no timer state in a base aggregate.
`vectors/timers/timer-archive-export-v1.json` pins the positive scoped export request, source
capture, required timer participant, complete `determa.scope_archive` manifest, member hashes and
lengths, and result. The §22 standalone base export remains applicable with no timer provider or
participant. `vectors/timers/timer-archive-stage-v1.json` pins successful inert staging and a
correctly resealed missing-required-timer refusal against independent trusted source and
participant-contract digests. Staging issues no timer claim or fire.

Under §24, timer participation is conditional on the trusted source contract: neither a base
aggregate nor a source with no installed timer gains a timer participant. Strict restore keeps
imported timer records inert and performs no due poll, clock-triggered fire, cancellation, or
delivery. A standalone takeover treats each inherited pending or claimed timer and each independent
fire without proved admission/source ownership as ambiguous if the old owner may still act. A
retained `pending` state proves no claim had occurred at capture, but does not prove the old owner
stayed inactive afterward. Reconciliation or explicit abandonment under §24 precedes any helper
resumption; the source record remains immutable evidence. A new scope cannot reuse the old scope's
fire event ID or treat its old timer receipt as a new-scope operation receipt. Any authorized new
timer intent uses the new scope and therefore a new §23.2 fire identity. A clone likewise resolves
or explicitly cancels inherited pending/ambiguous timer work and proves provider isolation before
activation. A proved same-authority transfer preserves the logical scope and stable fire identity
only after revoking active helper claims, resolving prior fire commit fate, and completing §24's
retirement and guarded activation. A committed admitted fire remains terminal in every mode and is
never automatically admitted again.

`vectors/timers/timer-helper-cases-v1.json` , `vectors/timers/timer-clock-cases-v1.json` ,
`vectors/timers/timer-records-v1.json` , and `vectors/timers/timer-cancelled-records-v1.json` ,
`vectors/timers/timer-fired-records-v1.json` , and
`vectors/timers/timer-committed-admission-v1.json` are normative cases for first execution, replay,
collision, cancellation/fire races, clock boundaries, stale claims, crash recovery, and delivery
limits. A helper passes all cases applicable to its advertised claims. An implementation without the
profile needs no background timer or store.

## 24. Recovery, fresh-scope takeover, cloning, and optional relocation

### 24.1 Contract and precedence

This optional host contract consumes a verified, inert §22 archive stage. It neither changes the §16
aggregate nor makes a portable archive an authority credential. The closed version-1 request and
result are `schema/recovery-operation-v1.schema.json` . A host authenticates the principal outside
the portable request, binds it to the requested operation and destination, and durably records the
first complete result under `(destination authority, operation_id)` . An equal canonical request
replays that complete result without repeating work; a changed request at the same ID returns
`scope_operation_conflict` . An operation ID is never a source checkpoint ID, event ID, or external
idempotency key. A missing/unsupported format or version, malformed request, unauthorized principal,
duplicate conflict, then operation-specific precondition failure is the refusal order. No refusal
stages a partially active scope. The result reports `safe_relocation` and
`no_duplicate_external_work` independently; neither is inferred from a valid archive or a successful
activation.

Recovery requires a complete §22 archive, including every selected root checkpoint and its owned
runtimes, active configuration, variables, ready and deferred mailboxes, queue and logical counters,
fault state, pending and terminal effects, intents, outbox, receipts, replay retention, tombstones,
exact definitions, migration descriptors and required participants. The host checks the archive
source profile, provenance, participant contract and digest against the trusted §22 import request
and stage receipt. It verifies required helper/application/host journal participants and any
external bytes. An individual checkpoint, selected-root archive presented as complete scope
inventory, missing participant, untrusted source, or altered source identity fails with
`scope_archive_incomplete` or `scope_archive_digest_mismatch` . The host MUST NOT recreate missing
queues or receipts from audit trails or omit terminal work. The source identity and binding digest
are public provenance, not credentials.

### 24.2 Strict quarantine

`strict_restore` always creates an immutable, inactive read-only quarantine for inspection and
reconciliation. Even a proved retirement does not change this operation into activation. Proved
continuation uses the separate tested transfer path and `activate_import` after its guarded commit.
Quarantine grants no admission, processing, core step, runtime-provider evaluation, helper
firing/dispatch/cancellation, ingress acknowledgement, effect dispatch/result acceptance, promotion
or clock-triggered timeout work. A lease expiry, copied database, archive digest, changed endpoint
or matching operation token cannot promote it. Its result records `state: "quarantined"` ,
`safe_relocation: false` , `no_duplicate_external_work: false` , and the missing proof. A request
for strict activation returns `scope_restore_requires_quarantine` and leaves the destination
inactive regardless of proof; strict restore has no activation branch. Quarantine can later be used
only by a separately requested operation whose full preconditions are checked afresh.

### 24.3 Explicit standalone takeover

`standalone_takeover` requires an authenticated, expressly named request with
`acknowledge_old_owner_risk: true` , a validated complete staged archive, an empty reserved
destination, a never-used logical scope identity and a never-used external idempotency namespace.
The new scope and namespace are permanently consumed even if the scope later terminates. The result
retains the exact source identity, source binding digest, archive digest and participant contract as
provenance. Source checkpoints and historical request/result receipts remain intact as evidence, but
cannot authenticate or replay as new-scope operations. The host initializes a fresh operation
ledger, bindings, authority epoch where supported, and worker claims; it never imports live source
claims or source authority marks. Old worker/result submissions are rejected under new-scope
authentication and fence checks even if root, effect, token or ID strings match. New intents use the
new namespace. No old namespace key is reused or inferred to deduplicate prior provider activity.

Every inherited submitted, accepted, pending, or attempt-unknown external effect, delivery, or
helper operation becomes an identified `ambiguous` inherited-work record unless exact retained
evidence proves that attempt unstarted. Terminal work remains terminal and is never automatically
replayed. Accepted ready and deferred machine events remain in their original order. The takeover
begins inactive. A separate `resume_takeover` requires a second explicit risk acknowledgement and
records the warning that the old owner may still act. It may enable pure machine processing, but
blocks every external dispatch, helper operation, ingress acknowledgement tied to ambiguous work,
and impure native runtime-provider evaluation until every inherited ambiguous item has a retained
`reconcile_work` outcome or an explicit `abandoned` outcome. Abandonment records the
operator/principal, item, evidence, decision, and risk; it does not assert that provider work was
undone. Pure processing must pause before any potentially impure evaluation or new external side
effect. The host MUST NOT automatically retry or redispatch an ambiguous operation. All standalone
results state `safe_relocation: false` , `no_duplicate_external_work: false` ; they disclose
old-owner and duplicate-work risk even after reconciliation. This is an availability choice, not
continuation of the old logical scope. Failure never falls back from strict restore or relocation to
this operation.

### 24.4 Clone as independent execution

`clone_scope` also allocates a never-used logical scope and external namespace, retains exact source
provenance and terminal evidence, and starts inactive. An `activate_clone` requires an explicit
declaration of independent business execution, proved isolated destination and provider bindings for
every side-effecting channel, and resolution or explicit cancellation of all inherited
pending/ambiguous external work and required helper state. Binding isolation is checked against the
source's actual provider/environment identity; a credential, alias or endpoint label change alone
does not prove isolation. Failure returns `clone_isolation_unproven` or `clone_has_unresolved_work`
, leaving the clone inactive. A clone-side cancellation records the decision never to deliver that
inherited work in the clone; it does not assert a source provider call was cancelled or undone.
Terminal effects stay terminal and are never redelivered. Ready/deferred events may remain; their
future execution and the new namespace are recorded in the activation evidence. A clone never
asserts source retirement, safe relocation or prevention of duplicate business work across scopes.

### 24.5 Optional same-authority relocation

A host MAY implement `prepare_transfer` , `stage_transfer` , `commit_transfer` , and
`activate_import` only for a topology positively advertised by its §18 profile and proved within its
single trusted authority domain. The stock profile defaults `safe_relocation: false` ; this
specification supplies no distributed coordinator, leader election or cross-authority grant.
Unsupported topology returns `host_capability_mismatch` and leaves the destination inactive.

`prepare_transfer` resolves the retained §18 freeze evidence, complete scope inventory and required
participant snapshots, all active claim revocations, known transaction fate, exact epoch and
generation, and destination binding. It records one destination-bound prepared transfer record under
the guarded authority transaction. This record proves a frozen source and a reserved destination,
never retirement or an activation grant. `stage_transfer` checks that prepared record, the exact
archive, source binding, participant contract, destination binding, epoch/generation, transfer ID
and an empty inactive destination; staging confers no writer rights. `commit_transfer` atomically
proves the source retired under §18, consumes the destination-bound single-use grant, advances
authority epoch/generation and binds one staged destination. Only this committed record is
retirement proof. `activate_import` checks that committed record and its exact destination, archive,
new epoch and generation under the guarded transaction before exposing admission, workers, helpers
or writes. Equal operation replay returns the first complete result. A conflicting transfer ID,
destination, consumed grant, stale generation or unresolved transaction fate fails closed before
stage or commit. An uncertain source transaction or commit returns `scope_transaction_in_doubt` and
remains inactive until its fate is resolved from trusted authority records; a timeout never selects
a winner and the source is never unsafely rolled back. Imported pending external work still requires
destination idempotency evidence or reconciliation before retry. Fencing prevents stale Determa
writes but cannot unsend a provider call. Only a profile that proves retirement, single use,
transaction fate, complete import, worker fences and absence of concurrent writers may report
`safe_relocation: true` .

Each logical scope is independently authorized and has its own receipts and archive selection. A
multi-scope request returns per-scope results, including explicit partial success, and makes no
global atomicity claim. Callers cannot combine partial proofs into a scope-wide or multi-scope
safe-relocation assertion. The normative `vectors/recovery/recovery-cases-v1.json` fixes complete
response and rejection bodies, including stale workers, quarantine, ambiguity, clone isolation and a
positively negotiated single-authority local transfer. Its separate guarded-source archive is
`vectors/recovery/archive-local-transfer-v1.json` . This test profile binds only an implementation
that advertises that exact topology; the stock profile may continue to refuse transfer. No fixture
implies a distributed grant service.

### 24.6 Records, digests, and denial behavior

`schema/recovery-record-v1.schema.json` is the closed durable destination record. It lists every
inherited external work identity and source attempt disposition, the exact retained checkpoint
digests, source provenance, fresh namespace, mode, current state, destination binding, fresh
operation-ledger identity, empty imported-claim set, risk decision and isolation evidence. A strict
quarantine has null namespace and ledger identity; fresh-scope modes require both.
`record_digest = hash(["determa-recovery-record-1", record])` . For an operation request,
`request_digest = hash(["determa-recovery-request-1", request_without_request_digest])` using §16.2
JCS. The host verifies this before ledger lookup; a mismatch is `invalid_recovery_request` . A
result's `record_digest` resolves the complete retained record and is not itself an authority proof.
Work identities are unique by `(work_kind, work_identity)` , ordered by UTF-8 bytes, and retain
source evidence across every resolution. An apparently unattempted operation requires affirmative
unattempted evidence; absence of a receipt means `unknown` . `reconcile_work` can change only an
ambiguous item and appends a durable decision; it cannot mutate source terminal evidence. A result
with `state: "active"` is valid only after the operation's activation guard committed. A refusal
retains `state: "unchanged"` and `record_digest: null` . Equal replay preserves exact warning and
source provenance fields. No error or retry performs an implicit takeover.

The host checks source archive/stage and authenticated destination rights before allocating a scope;
then checks fresh identity, required participants, unresolved work and, if requested, authority
proof. A destination reservation, existing root, prior namespace use or ledger collision fails
`scope_destination_not_empty` or `standalone_takeover_requires_fresh_scope` with no activation. An
unsupported transfer uses `host_capability_mismatch` ; a purported proof that does not resolve to
the trusted authority ledger uses `scope_fence_unproven` . A stale worker result uses
`stale_scope_authority` or `stale_attempt_fence` at §18/§19 before its payload can change the new
scope. Responses must use exactly the operation's closed code set and preserve the first result. The
examples fix the relevant precedence and full results.

The version-1 `guarded_action` probe gives a complete, read-only admission decision for
`admit_event` , `process_event` , `dispatch_effect` , `submit_effect_result` , `fire_helper` ,
`cancel_helper` , `clock_timeout` , `ack_ingress` , `evaluate_impure_provider` , or `promote` . It
does not perform the action. The action itself must repeat the same guard at its native commit
boundary; a successful probe is no transferable permit. This lets conformance assert quarantine and
old-worker denials without invoking an external provider. Inactive scope denial precedes payload
validation. A worker-scope mismatch is `stale_scope_authority` even when a token string matches;
within the current scope, a stale attempt fence is `stale_attempt_fence` . Standalone ambiguous work
denies external dispatch and impure evaluation with `standalone_takeover_ambiguous_work` until every
relevant item is resolved.

A host that offers a batch convenience interface returns the closed
`schema/recovery-batch-v1.schema.json` result. Its scope results are individually ordered by scope
identity and each is the complete first result from its own receipt ledger. `status: "partial"`
means at least one scope succeeded and one refused; `atomic_across_scopes` is always false. The
caller must authorize every scope independently. Batch failure never rolls back a committed
per-scope result or turns one scope's proof into another's grant.

The trusted authority ledger stores a closed `schema/recovery-transfer-proof-v1.schema.json` record
for each prepared or committed transfer.
`proof_digest = hash(["determa-recovery-transfer-proof-1", proof_without_proof_digest])` . The
`prepared` phase binds frozen source state, revoked active claims, known freeze transaction fate,
one transfer ID, destination reservation, archive and participant contract, and the exact old
epoch/generation. Its destination epoch and generation are reserved proposed values, not active
authority. It is not a retirement proof or activation grant. The `committed` phase binds the §18
retired source, consumed single-use grant, known commit fate, old-write fence, and new destination
epoch/generation. A request's public proof digest is only a lookup key: the host resolves the
phase-correct record in its guarded ledger, checks its digest and all fields against current
authority state, and consumes the grant atomically. A copied proof JSON or digest cannot grant
authority. Unknown transaction fate cannot satisfy either phase. No cross-authority host may
advertise this profile unless its own topology proves these properties across both domains.
## 25. Public client and execution-host protocol

### 25.1 Named bindings and authority

The remote public protocol is `determa.execution_host` version `1` . It is the common client
boundary for the open-source reference host and a later hosted service. HTTP routes, MCP tool names,
SDK objects and database layouts are adapters, not alternative semantics. Clients may configure
multiple named endpoint/scope bindings and switch a selected binding without editing a format-1
machine. Endpoint URLs, credentials, tenancy and routing are deployment configuration and never
portable state or machine declarations. The embedded §20 facade may implement the same operations in
process with application-owned rows and native transaction composition; it needs no remote user
database transaction, privileged URI scheme, or authority coordinator.

A fresh client selects a named endpoint in its deployment configuration and sends the ordinary
authenticated `capabilities` request with `scope_binding_identity: null` , null target and null
precondition. Its closed arguments contain `scope_alias` , the configured name of the requested
logical scope. The selected endpoint and authenticated transport principal are invocation context;
the alias is a public deployment selector, not a machine field or credential. The host resolves the
alias at that endpoint to one authorized logical scope and immutable public binding, then returns
that identity in the closed capabilities value. It returns no identity or scope existence detail on
denial. A later `capabilities` request MAY present that non-null identity to recheck health; every
other operation MUST present it. This is the protocol bootstrap, with no separate private or
SaaS-only handshake. Before its first mutation for an operation, the client discovers the configured
name's immutable public `scope_binding_identity` and endpoint authority. It durably saves the
resolved endpoint, logical scope, complete canonical request, digest and operation ID. The binding
identity is public equality evidence, not a credential. The host authenticates the transport
principal and authorizes the scope before any root-existence lookup or existence-disclosing
response, capability report or replay evidence. A valid unauthorized request produces the same
denial whether the root exists or not. Structural/version and digest errors may be rejected before
scope lookup, but MUST reveal no scope facts. Credentials and principal claims remain outside
protocol bytes and hashes.

After a lost response, the client queries or retries only its saved endpoint, scope binding and
operation identity. An alias remapping never retargets saved work. An unavailable or changed
endpoint identity yields `binding_unavailable` /unknown fate; the client MUST NOT resolve the new
alias, automatically fail over, allocate a replacement operation ID or treat a missing receipt as
rollback proof. Cross-authority relocation requires separately verified §18/§24 continuity and
participant evidence. An explicit fresh-scope standalone takeover uses a new namespace and weaker
guarantees and is not a retry.

### 25.2 Closed request, response and digest

`schema/public-host-request-v1.schema.json` defines exactly eight request fields: `protocol` ,
`protocol_version` , `operation_id` , `scope_binding_identity` , `operation` , `target` ,
`precondition` , and `arguments` . The protocol is `determa.execution_host` , version is integer `1`
, and operation is one of `capabilities` , `create` , `admit` , `process` , `read` , `inspect` ,
`receipt` , `effect_result` , `cancel_effect` , `timer_command` , or `scope_operation` . `target`
has exact `root_instance_id` , `runtime_id` , and `runtime_incarnation` , each null when
inapplicable; a non-null incarnation is §16's exact immutable identity origin. `precondition` is
null or an exact revision/checkpoint-digest pair. The selected operation has closed `arguments` .
Only a `capabilities` bootstrap may carry null `scope_binding_identity` ; its successful value names
the resolved non-null binding. Typed values use §16.2 before hashing; omitted, null and empty fields
are different. An operation ID is a nonempty caller-chosen identity unique within its authenticated
logical scope. No effect ID, event ID, creation ID or operation token may substitute for it. The
`create` target root MUST equal `arguments.root_instance_id` ; `process` requires the exact non-null
runtime ID and incarnation; `inspect` target MUST equal its candidate runtime and incarnation; every
`admit` delivery MUST address a runtime owned by the target root. A nested cancellation request MUST
reuse the outer `operation_id` ; a nested authority request MUST name the authenticated scope bound
by `scope_binding_identity` . These cross-field equalities are checked before mutation.

```text
request_digest = hash(["determa-public-host-request-digest-1", "1", complete_request])
```

The hash uses §9 JCS/SHA-256. The host identity key is
`(authenticated_scope_identity, operation_id)` ; equality requires the same request digest and saved
`scope_binding_identity` . Once authentication and current scope/authority checks pass, the host
checks that key before resolving any mutable effect route or endpoint alias. A host receipt binds
exactly `scope_binding_identity` , `operation_id` , `request_digest` , `receipt_kind` ,
`acceptance_receipt` , and `evidence_digest` . For `receipt_kind: committed` , `acceptance_receipt`
is null and the evidence digest is
`hash(["determa-public-host-evidence-1", "1", complete_operation_tagged_value])` . A non-null
receipt on a `committed` public response MUST have `receipt_kind: committed` ; `accepted` is
reserved for the durably evidenced `pending` response. For `receipt_kind: accepted` ,
`acceptance_receipt` is the existing complete §17 input-acceptance receipt, with its durable
`receipt_sequence` , `acceptance_sequence` , and envelope request digest; the evidence digest is
`hash(["determa-public-host-acceptance-evidence-1", "1", scope_binding_identity, operation_id, request_digest, acceptance_receipt])`
. The host must resolve that receipt and the exact retained caller request at the pinned binding
before returning `pending` . This defines no new sequence allocator or alternate admission boundary.
A §17 acceptance receipt proves the event was admitted, not that a separate `process` , timer,
effect or scope request was accepted. `pending` is available only when the configured host can
additionally link that exact public request to a durable accepted-work record; the minimal reference
profile need not emit `pending` . Without that proof the host returns a committed/rejected result or
no response on unknown fate. Neither digest alone is commit proof. The host resolves it to the
retained atomic receipt and storage evidence. It stores the first exact response and complete
request digest with the authoritative mutation receipt; a linked journal or helper ledger may hold
the response when the portable checkpoint cannot. Equal replay returns those saved bytes, including
original artifact/inspection/result bindings, without a core call, alias resolution, outbox dispatch
or new counter. Unequal reuse returns `operation_id_conflict` without mutation. A response-reference
hash alone is insufficient unless the exact first response is retained or reconstructible. Read
operations are not inserted into the mutation ledger; each read returns its observed artifact
digest/revision.

`schema/public-host-response-v1.schema.json` defines exactly seven response fields: `protocol` ,
`protocol_version` , `operation_id` , `status` , `receipt` , `value` , and `error` . `committed` has
a complete operation-tagged `value` and null `error` ; for a mutation its receipt binds the
committed request and evidence, while an observational read, accepted authority read, or
empty-mailbox `not_runnable` process may have null receipt. A `process` disposition of `handled` ,
`deferred` , `unhandled` , or `faulted` requires the public committed receipt even when no terminal
event receipt exists; deferral and recall alter the durable mailbox/checkpoint. A successful
mutating §18 `guarded_commit` , `freeze_scope` , `fence_worker` , or `prove_retirement` likewise
requires it. A nested §18 `rejected` result cannot appear in a committed public value. It means the
named operation's authoritative transaction committed, or an observational operation returned a
bound observation; it never means merely that a handler finished. `pending` has no value/error and
requires a durable accepted receipt queryable at the pinned binding; an in-memory promise cannot
produce it. `rejected` has a closed operation-tagged error and null receipt/value, with no claim
that an earlier request of unknown fate rolled back. Closed §18/§19/§22/§23/§24 nested refusals
appear in the operation's `error.authority_result` , `error.effect_result` ,
`error.cancellation_result` , `error.timer_result` , `error.archive_result` , or
`error.recovery_result` respectively, with the same outer and inner error code and no claimed
commit. Accepted authority operations and committed effect, archive, or recovery operations retain
their complete nested result in `value` and the transaction receipt. A transport timeout has no
protocol response and means outcome unknown. Expired or absent replay evidence MUST be reported
honestly; neither proves rollback. Reconciliation uses retained checkpoint, journal, ingress or
helper evidence.

### 25.3 Operation bindings

`capabilities` authenticates the scope, checks configured instance health/topology and returns the
resolved binding identity, exact supported operations, supported scope-action/timer-command
variants, the independently configured `supported_determa_capabilities` set, and separate guarantee
claims. That set uses exactly `schema/archive-v1.schema.json#/$defs/capability` ; it is destination
decoder/staging support, not source `profile_claims` or host authority. On import, the host compares
the manifest's exact content-derived `required_determa_capabilities` against its current configured
set before any staging write. A required missing feature produces §22's complete
`archive_capability_mismatch` refusal. The discovery set does not waive §22's source, participant,
optional-reference, or trusted inventory checks. Provider names, schemes, descriptors and report
digests do not prove a configured claim. A required missing or unhealthy capability or
archive/helper participant produces `host_capability_mismatch` before mutation. A host MAY advertise
an operation only for an implemented, tested profile. Every operation still rechecks its
dependencies at invocation. The discovery `profile_digest` is
`hash(["determa-public-host-profile-1", "1", scope_binding_identity, capability_result_without_profile_digest])`
; it binds the returned configured report and claim list, but does not itself authorize another
scope or prove a later health state.

`create` , `admit` , and `process` bind the §8 foreground operations and §17 checkpoint boundary.
Creation arguments include exact definition fingerprint, machine identity, root ID, creation ID and
normalized typed bindings. Admission contains complete ordered
`(delivery_mode, envelope, envelope_digest)` entries; each digest uses §16.15 and is verified before
commit. Processing targets one exact runtime and incarnation under the supplied checkpoint
precondition. Successful values include the complete resulting checkpoint, exact §8
operation-specific result and durable creation/acceptance/terminal receipt evidence as applicable.
They never invent a disposition for `create` or emissions for `admit` . Checkpoint, outbox and
receipt facts come from one authoritative commit. §17 creation/event replay and conflict precedence
still apply; the host operation ID is additional identity. A §21 ingress adapter preserves its
separately closed source-owned, admitted, dead-letter, machine-disposition, mailbox-placement and
outbound decisions. Only actual admission and terminal processing produce their respective
checkpoint receipts; deferral/recall and outbox updates bind their checkpoint or outbox evidence
without inventing an operation receipt. A source-owned precommit decision or ingress dead letter is
not represented as a committed checkpoint receipt.

`read` returns one complete validated checkpoint and observed digest, or an exact absent result.
`inspect` submits the exact §12 candidate and returns its complete outcome bound to the observed
aggregate/checkpoint. Neither mutates. `receipt` queries the saved operation ID and request digest
at its pinned binding and returns the full first response or a precise retained/expired/unknown
status; `unknown` is not rollback proof. `effect_result` and `cancel_effect` are conditional §19
operations. They require authenticated worker/route/claim checks and exact token/fence/journal
evidence; mere possession of an effect ID grants no authority. Resuming a §19 stored outcome after
worker expiry is a host recovery step over committed journal evidence, not a new worker
`effect_result` submission; it does not fabricate or require a live worker claim and cannot call the
provider again. `timer_command` is conditional on the §23 helper and its durable operation receipt;
the core has no clock or `poll_due` . It wraps the exact closed
`schema/timer-helper-operation-v1.schema.json#/$defs/request` and returns the exact accepted helper
result, including its computed `result_digest` . The public operation ID equals the nested one; the
authenticated scope and target root/root-runtime equal the nested scope and immutable root-runtime
identity. The helper checks its own canonical request digest, deadline clock basis, due/fence
conditions and worker rights. A timer success that mutates the helper ledger is outer `committed`
only with a retained public receipt backed by that durable helper commit. `read_timer` is
observational and may have null receipt. A nested helper `rejected` result is outer `rejected` with
the complete helper result in `error.timer_result` , the same code, and null receipt/value. A clock
outage, stale fire fence, or ambiguous delivery cannot be reinterpreted as a successful fire. Timer
records and admission evidence stay with the §23 archive participant; a public result digest alone
does not reconstruct or prove them. `scope_operation` carries the exact nested §18 authority
request/result when that capability is offered. Its `archive_export` and `archive_import` variants
carry the exact closed §22 requests and results. Export success returns the complete
`determa.scope_archive` bytes and result in the retained first public response; import carries the
complete archive bytes with the §22 import request and returns the exact staged result. An import
only creates inert staging: it does not activate roots or convey ownership. §22's complete precommit
`refused` result maps to outer `rejected` with null receipt/value and the exact result in
`error.archive_result` , where `error.code` equals its refusal code; it claims no transaction
commit. A successful export or stage maps to outer `committed` only when the named operation
transaction and its complete response receipt are durably retained. The same rule applies to nested
§18 and §19 refusals: a definitive nested response is not proof of an outer transaction commit. The
`recovery` scope variant carries the exact closed §24 request and complete result, plus its complete
durable `recovery_record` when the result names one. The outer operation ID equals the inner one;
the authenticated destination scope bound by the selected endpoint equals
`destination_scope_identity` . The host verifies the nested §24 request digest and independently
trusted inert stage, source archive, participant closure, destination rights, prior destination
record, and any source-retirement or clone-isolation proofs before a mutation. A recovered record
digest resolves to the complete retained record; it is not an ownership credential. Strict restore
remains quarantined; fresh-scope takeover and clone require their separate risk and isolation
conditions. Only a positively configured and verified same-authority transfer may claim
`safe_relocation` ; unsupported topology returns the exact §24 `host_capability_mismatch` refusal in
`error.recovery_result` . A §24 `refused` result maps to outer `rejected` without receipt or value.
A successful mutation maps to outer `committed` only with its retained public receipt and exact
complete result/record; read-only `guarded_action` may have a null receipt and must recheck the
guard at a later native commit. Recovery uses the same public protocol for local and future hosted
profiles, with no service-private activation path. Unsupported actions fail before mutation.

Inspection, retained history, saved-response replay and re-execution are distinct claims. Structural
inspection is available for valid executable definitions; semantic inspection requires §12's proved
bounded nonmutating entrypoint. History exposes retained committed facts subject to its retention
profile. Replay returns saved bytes with no execution. Re-execution may compare exact machine
outcomes only for deterministic portable provider closures. Weak, impure, nonportable or
nondeterministic profiles compare canonical artifacts, identities, retained evidence and safety
refusals but MUST NOT claim repeat-execution equality. An API or MCP adapter validates the same
closed message, scope and capability rules, and returns the same semantic response; transport-only
credentials and tracing do not enter the protocol object.

### 25.4 Public compatibility gate

`schema/public-host-contract-v1.json` records the reviewed boundary sources and SHA-256
fingerprints, hash domains, capability meanings and helper participant rules. It MUST cover
`SPEC.md` , `VERSION` , every `schema/*.schema.json` , and the cited golden source artifacts with no
duplicate path; each `sha256` is the raw file-byte digest, not a semantic substitute for review. Its
`hash_domains` list MUST enumerate every named version-1 Determa hash domain in this specification
exactly once, including effect, delivery, archive, timer and recovery domains when defined. Its
closed shape is `schema/public-host-contract-v1.schema.json` . The corresponding closed
change-record shape is `schema/public-host-change-record-v1.schema.json` . `vectors/public-host/`
contains complete positive and negative golden messages and canonical request hash operands. A
change to a recorded protocol, artifact, effect, capability or helper source requires a reviewed
change record naming old/proposed fingerprints, affected claims and the positive/negative
cross-language fixture matrix. Specification and conformance checks mechanically compare closed
schemas, source fingerprints, canonical hashes and exact fixture bytes. Python and Rust clients and
the local reference host run the matrix for every advertised operation/profile; a later hosted
service runs that same suite for each capability it claims. A pre-alpha breaking redesign updates
this single current version-1 contract and coordinated pins, without a provisional compatibility
layer or service-specific machine grammar.
