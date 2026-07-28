# Determa State — specification

Status: **pre-release alpha**. Normative unless a section says informative.
Document format: **1**.
Spec version: **0.0.6** (see `VERSION`; synchronized across the Determa State
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
- `version` — positive integer, optional, default `1`.
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

Payload fields use `string`, `int`, `float`, `bool`, `map`, or `list`. A required field
must be present; an optional field may declare a literal `default`. Extra fields are
invalid.

`correlates_to` is valid only on a bundle `input` event and MUST name a bundle `output`
event. Success, rejection, failure, cancellation, or no-response outcomes from an
external effect become machine behavior only through later declared input events.

The names `env`, `done`, `determa.component_completed`,
`determa.component_failed`, and `determa.spawned_instance_failed` are reserved and
cannot be author declarations.

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
- `init` — optional literal value used on entry.
- `input: true` — permits a typed value when creating this runtime.
- `external: true` — declares a host-source value copied into machine state.
- `nullable: true` — permits null.
- `machine_id` — optional nominal constraint for `instance_reference`.

A declaration cannot be both `input` and `external`. Both flags are valid only on root
variables, which makes creation and refresh bindings unambiguous. External variables
are read-only to `assign`; they change only through a successful `env`/`refresh` step.
An `instance_reference` cannot be external and cannot be constructed from an arbitrary
string.

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
event data into a variable whose scope survives the transition.

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

A choice pseudostate is represented by an ordered `choice` branch list. It is transient
and mutually exclusive with active-state fields.

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
- `local: true` — for a composite source targeting its strict descendant, preserve the
  source instead of applying the unmarked transition's source reset (§6.4).
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
| assign | `{ assign: { variable: CEL, ... } }` | typed writes in the executing runtime |
| send | `{ send: { event, to? \| targets?, payload?, correlation_id? } }` | ordered immutable emissions |
| refresh | `{ refresh: { only?: [name, ...] } }` | adopt validated `env` values |
| spawn | `{ spawn: { machine_id, bindings?, bind_to? } }` | create an owned runtime |
| cancel | `{ cancel: { instance: CEL } }` | cancel an addressed owned runtime; null or non-targetable is a no-op |
| stop | `{ stop: {} }` | complete the executing runtime |

`stop` MUST be the final action in its list and is invalid in exit behavior. Once it
executes, any transition target is abandoned and deterministic runtime completion
replaces normal target entry.

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
direction/target rules are checked at load time.

## 5. Static validation and CEL

### 5.1 Load-time validation

A bundle is rejected before any runtime is created when it has:

- duplicate machine, component, state, variable, or event identities;
- unresolved machine, state, event, or variable references;
- unreachable states;
- an initial transition targeting anything outside its owning composite;
- a reachable composite without an initial transition;
- a guard or trigger on an initial transition;
- an `input` or `external` variable outside a machine root;
- an unguarded branch before a later branch;
- a cycle in state nesting or component placement;
- an invalid public/private event direction or correlation;
- a send whose target, event direction, or correlation is inconsistent;
- `local: true` without a target, with a non-composite source, or with a target that is
  not a strict descendant of its source;
- a transition selected on the machine root whose target is the machine root, or any
  history target that resolves to the machine root (`root_reentry`);
- a history target naming a non-composite state, a composite whose `history` is `none`,
  or a composite that strictly contains the transition source;
- an `assign` or `refresh` whose destination variable belongs to a state exited by
  that same transition (`destroyed_variable_write`);
- a spawn `bind_to` whose destination reference belongs to a state exited by that same
  transition (`destroyed_reference_binding`);
- invalid creation bindings or `instance_reference` constraints;
- an initial/choice cycle that cannot reach a stable state;
- a transition inside entry, exit, or initial behavior;
- a `stop` action in exit behavior;
- a `stop` action that is not last in its action list; or
- a CEL parsing, name-resolution, or type-checking failure.

For the destroyed-destination rules above, exits performed by root, component, or
spawned-runtime completion after reaching a final state or executing `stop` belong to
the same transition or action outcome.

### 5.2 CEL environments and compile-time types

Every CEL expression MUST be parsed and type-checked at bundle load with the exact
variables, current event schema, owner binding environment, and expected result type
for its location.

- A guard must infer `bool`.
- An `assign` expression must be assignable to the destination variable.
- Send payload expressions must satisfy the target event payload declaration.
- `correlation_id` must infer `string`.
- Component and spawn bindings must satisfy the target root input and external
  declarations.
- Instance targets and cancellation expressions must infer `instance_reference`.
- A dynamic value cannot flow into a concrete destination without an explicit checked
  conversion.

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

An input envelope is:

```text
{
  event: event_name,
  event_id: non_empty_string,
  target: root | live_instance_reference,
  payload: typed_map,
  correlation_id?: non_empty_string
}
```

The caller owns the envelope. The core neither enqueues nor stores it. Before starting a
step, the core atomically validates event declaration, direction, payload, correlation,
and target eligibility. Rejection returns the prior state unchanged and allocates no
logical identity.

An ordinary host envelope MUST name a bundle `input` event and may target only a
running root or running spawned runtime. If the declaration has `correlates_to`, the
envelope MUST carry a non-empty `correlation_id`. An internal emission later presented
by a queue plugin may name a bundle or machine-local `internal` event and may target a
running root, spawned runtime, or component.

A completed, faulted, or pending-completion root/spawned runtime rejects ordinary input
with `invalid_instance_target`. A pending-completion, completed, or faulted component
rejects an internal send with `inactive_component_target`; it cannot accumulate work
that will never run. Rejection is atomic. Calling `dispatch` with no envelope is not a
processing operation; read-only inspection of a terminal aggregate returns its existing
status and no emissions.

Reserved `env` is the only undeclared host-input exception:

```text
{
  event: "env",
  event_id: non_empty_string,
  target: root | live_instance_reference,
  payload: { changed: { external_name: typed_value, ... } }
}
```

It has no correlation id. `changed` is non-empty and every field must match a declared
root external variable on the targeted runtime. A `refresh` action is valid only in the
selected `env` handler and copies the requested changed values into those root
variables atomically. Changed fields not selected by `refresh` are not retained by the
core.

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
2. execute transition actions in the source configuration and source variable scope;
3. exit the active source path from innermost to outermost, stopping below the
   transition boundary defined below;
4. enter the target path below that boundary, from outermost to innermost; and
5. restore explicitly targeted history or follow nested initial transitions until a
   stable leaf is active.

Because step 2 precedes exit, a write to a variable or `bind_to` reference destroyed by
step 3 would have no observable result. §5.1 therefore rejects such a transition at
load time. A write to an ancestor-scoped destination that survives the exit remains
valid. For a choice chain, every possible selected branch must satisfy this rule.

This differs from canonical UML ordering, which treats the transition effect as behavior
of the edge after source exit and before target entry. Determa instead keeps the source
context intact while the transition action runs. Authors familiar with UML MUST rely on
the order above for Determa definitions.

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
  lifecycle actions untouched.
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

A choice is transient. Prior transition actions run first; choice guards then evaluate
against the resulting variables, and the first true branch is taken. The final branch
MUST be unguarded.

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

## 7. Components, spawning, and lifecycle

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

Entering a parallel state is part of the owner step:

1. allocate component runtime identities in declaration order;
2. initialize the parallel state's variables and run its entry actions;
3. evaluate `with` using owner variables only;
4. create and initialize each component in declaration order; and
5. reach a stable configuration in every component before the owner step commits.

The triggering event is unavailable to entry actions and `with`. A transition action
must first copy required payload into an owner variable.

Components have isolated configurations and variables. They do not share an event queue
because queues are outside core. An event reaches a component only through an explicit
emission targeting its nominal component runtime identity.

The `{component: component_id}` syntax resolves at emission time to the currently
running placement identity, including its activation sequence. The immutable envelope
stores that resolved runtime identity. Delayed delivery to a disposed placement
therefore cannot accidentally reach a later re-entry incarnation with the same
`component_id`.

A component reaching its root final state or executing `stop` becomes completed and
emits one `determa.component_completed` envelope to its owner:

```text
{ component_id, component_runtime_id }
```

When every component placement is complete, the same successful step also emits the
reserved `done` event to the owner with:

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

Pending-completion, completed, and faulted component runtimes remain inspectable until
their parallel owner exits, but are terminal and non-targetable. Only pending-
initialization and running components accept internal sends.

Exiting the parallel state synchronously cancels its retained component runtimes in
reverse declaration order, performs deepest-first owned-child cleanup, runs exit
actions for running components, and disposes their logical state atomically with the
owner transition. Cleanup of a retained-faulted component skips its author exit actions
and disposes the frozen subtree deepest-first.

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

The action allocates a deterministic child identity, validates and copies root input
and external bindings, initializes the child to a stable configuration, and optionally
stores its nominal `instance_reference` in `bind_to`. Creation and binding are atomic
with the parent step. `bind_to` MUST name a compatible nullable `instance_reference`
whose current value is null; otherwise the step faults with `binding_not_empty`.

The declaring scope of the `instance_reference` used by `bind_to` defines the bound
child's maximum lifetime. Binding records that declaration as the child's lifetime
holder; copying or later replacing the nominal reference value neither transfers nor
erases that association. When the holder's state exits, every running or
retained-faulted associated child is synchronously cancelled and disposed before the
declaring state's exit actions run. The engine orders holders by ascending identifier
byte order and children of one holder by ascending spawn sequence, then cancels each
descendant subtree deepest-first. The cleanup is atomic with the owner transition.
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
directly or transitively owned instance, it synchronously cancels descendants
deepest-first, runs remaining exit actions, disposes logical child state, and
invalidates the reference as a target. A retained reference remains serializable and
comparable but no longer addresses a live runtime. Faulted instances accept
cancellation only for this cleanup; they reject ordinary events and sends. Cancellation
of a retained-faulted instance skips its author exit actions and disposes the frozen
subtree deepest-first. Every other resolved value succeeds without effect as described
below.

If the cancellation expression evaluates to null, or does not currently address a
running or retained-faulted directly or transitively owned instance, the action succeeds
as a no-op. The action itself changes no ownership or counters and produces no fault or
emission; the surrounding RTC step continues normally.

Natural child completion performs the same descendant cleanup and emits one reserved
`done` envelope to its immediate owner:

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

The `parallel` and `spawned_instance` maps above are the complete tagged-union payload
schema for reserved `done`.

The completed child subtree is then disposed and its `instance_reference` becomes
non-targetable. The completion envelope retains the nominal identity needed by the
owner. A faulted child subtree is instead retained for diagnostics and may be disposed
only by explicit owner cancellation or owner/root cleanup.

Remote provisioning is never core `spawn`. A machine requests it through an external
output intent and receives declared correlated input events from a host extension.

### 7.3 Runtime and aggregate-root completion

When any runtime reaches its root final state or executes `stop`, it synchronously
cancels all retained owned descendants, runs its active exit actions deepest-first, and
becomes completed before the RTC commits. Component and spawned-runtime retention and
notifications then follow §7.1 and §7.2.

For the aggregate root, the engine retains terminal identity/status and fault-history
diagnostics and returns `completed`. No new ordinary envelope may target it.

## 8. Pure foreground interface and logical state

Language APIs may use idiomatic names, but every implementation must provide behavior
equivalent to:

```text
create(bundle, machine_id, root_instance_id, creation_id, bindings)
  -> { status, state, emissions, faults }

dispatch(bundle, prior_state, envelope)
  -> { status, disposition, state, emissions, faults }
```

`status` is `running`, `completed`, or `faulted` for an existing aggregate. A creation
rejected before an aggregate exists returns `status: rejected` and `state: null`.
Dispatch rejection or an unhandled envelope preserves the prior aggregate status.

`disposition` is exactly:

- `handled` — the accepted envelope completed a successful RTC step;
- `unhandled` — no enabled handler existed;
- `rejected` — validation failed before an RTC step; or
- `faulted` — an engine fault occurred during the RTC step.

The root ownership aggregate contains:

- the root runtime;
- every retained component runtime;
- every live or retained-faulted owned spawned descendant;
- each runtime's definition identity, configuration, variables, history, lifecycle
  status, source maps, and fault records;
- ownership and `bind_to` lifetime-holder associations, plus placement, activation,
  state-entry, spawn, logical-step, and output identity counters.

It contains no event queue, deferred queue, dead-letter collection, timer, broker
acknowledgement, transport receipt, or plugin configuration.

The host may store one aggregate in one row/document or normalize it, provided every
dispatch sees serializable prior state and commits an observably equivalent result.
The core itself performs no persistence.

Creation initializes the aggregate and every root initial/entry action atomically.
Creation bindings contain separate `input` and `external` maps. Missing required,
extra, or wrongly typed values reject creation without state or emissions.

A value-dependent root initialization fault rolls author behavior back, commits only
the deterministic root fault record in a terminal faulted aggregate, and returns no
author emission. A contained component initialization fault follows §10.2 and may
commit a running owner aggregate with the contained fault and its one failure emission.

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
  "determa-root-runtime-identity-1",
  "1",
  namespace,
  machine_id,
  canonical_decimal(machine_version),
  root_instance_id
])
```

Normative root-identity vector:

```text
JCS:
["determa-root-runtime-identity-1","1","example.turnstile","turnstile","1","turnstile-42"]

hash:
sha256:c5d32baf8704e310e39e65536c84cbf1d5d33ce8276f4aaaf9878e5b5590fd58
```

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
["determa-cause-identity-1","1","root_initialization","turnstile-42","sha256:c5d32baf8704e310e39e65536c84cbf1d5d33ce8276f4aaaf9878e5b5590fd58","sha256:c5d32baf8704e310e39e65536c84cbf1d5d33ce8276f4aaaf9878e5b5590fd58","create-7","0","/machines/0/root","0"]

hash:
sha256:063ca0e80bec3ffc23fc8754de4275f64dfc3fa5c14ae1cf24eaabc99fb62889
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
- `invalid_binding`;
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
    fault: fault_record
  }
  ```

- `determa.spawned_instance_failed` with:

  ```text
  {
    instance: instance_reference,
    instance_id: non_empty_string,
    machine_id: identifier,
    machine_version: positive_integer,
    fault: fault_record
  }
  ```

The `fault` field is a value-copy of the committed record above. Each failed runtime
emits its notification once. Source is the failed runtime; target is its immediate
owner; the emission retains the faulting cause and uses the corresponding system
locator from §9.

A reserved failure event that reaches its owner unhandled faults the owner with
`contained_runtime_fault`; its source locator is the owner's selected handler-search
path and its cause id is the failure envelope's `event_id`. A queue plugin can discard
or delay the notification; the core does not claim otherwise.

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
| definition migration/hot-swap | unsupported |
| observers and export | read-only inspection and visualization are recommended, not executable core behavior |
| snapshots | no portable wire format in this alpha |
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
- machine definition migration or hot-swap;
- portable snapshot wire format;
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
- CEL name/type checking and event visibility;
- leaf-to-ancestor dispatch and false-guard fallback;
- internal, self, local, and external descendant-reset transition traces;
- proper-ancestor transition bounds and the absence of external ancestor re-entry;
- schema rejection of non-canonical internal/local transition shapes;
- least-common-ancestor exit/entry paths;
- transition-action-before-exit ordering;
- load-time rejection of transition writes to destinations that the transition exits;
- root-boundary preservation plus root self/history rejection;
- destroyed `refresh` rejection and owned-child cancellation on reference scope exit;
- initial descent and ordered choice;
- explicit history resume/restart, self-history lifecycle replay, first-entry fallback,
  capture timing, shallow/deep restoration, local history targeting, and variable
  reinitialization;
- immutable envelope validation and each disposition;
- no recursive delivery of internal sends;
- component creation/routing/completion/disposal;
- spawn, nominal reference, completion, cancel and cascade, including no-op
  cancellation of null and disposed references;
- deterministic event/effect identity vectors across languages;
- full RTC rollback and contained-runtime failure propagation;
- queue-independent behavior for a fixed delivery trace; and
- explicit absence of timer, queue, and dead-letter fields from core state.

Golden-trace cases SHOULD make every action emit a trace token so ordering is directly
reviewable.

## 15. Future example repositories

After conformance and engine support, runnable examples should validate:

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
- real-time-oriented hosting.

Those examples are empirical design validation. They are not part of this
specification-only change.
