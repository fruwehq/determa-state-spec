# Determa State decision record

This file records settled design rationale and review guidance. It is **not normative**
and is not a second specification. [`SPEC.md`](SPEC.md), the schema, and applicable
conformance cases define behavior; if this record conflicts with them, correct this
record rather than treating it as an alternative rule source.

## Transition boundaries are relationship-specific

Relevant specification: [§6.4](SPEC.md#64-transition-execution-order).

**Decision.** Format 1 enumerates every targeted source/target relationship:

| relationship | transition boundary |
|---|---|
| source equals target | parent of a non-root source; root self-target is invalid |
| target is a strict descendant, unmarked | parent of a non-root source; root remains invariant |
| target is a strict descendant, `local: true` | source itself |
| target is a proper ancestor | target itself, without re-entry |
| source and target are unrelated | ordinary least common ancestor |

An action without `transition_to` is an internal reaction and changes no configuration.

**Rejected alternative.** Apply the ordinary least common ancestor in every topology.

**Reason.** When a composite source contains its target, the ordinary least common
ancestor is the source. That made `local: true` a no-op in the only topology where it
is legal and removed the reset-in-place form.

## The machine root is an invariant boundary

Relevant specification: [§6.4](SPEC.md#64-transition-execution-order) and
[§13](SPEC.md#13-deliberately-unsupported-in-format-1).

**Decision.** Ordinary transitions never exit or re-enter the machine root.

**Rejected alternative.** Give the root a virtual parent and apply ordinary external
transition boundaries through it.

**Reason.** A root handler targeting a descendant would reset the complete runtime,
including root input and external variables, which conflicts with their creation and
refresh boundaries.

## Cross-root external re-entry is deliberately unexpressible

Relevant specification: [§5.1](SPEC.md#51-load-time-validation),
[§6.4](SPEC.md#64-transition-execution-order), and
[§13](SPEC.md#13-deliberately-unsupported-in-format-1).

**Decision.** A root-to-descendant transition preserves the root, while a target that
would re-enter the root is rejected. External re-entry across the root/descendant
boundary in either direction is not expressible in format 1.

**Rejected alternative.** Add another external-transition marker or a virtual
root-parent escape.

**Reason.** Format 1 accepts the smaller local-only root semantics to preserve root
identity and variable lifetime. A later format may add an explicit reset operation
without overloading ordinary transitions.

## Transition actions execute before source exit

Relevant specification: [§5.1](SPEC.md#51-load-time-validation) and
[§6.4](SPEC.md#64-transition-execution-order).

**Decision.** Transition actions run in the source configuration and lexical scope
before any state exits.

**Rejected alternative.** Use UML-style ordering in which the transition effect runs
after source exit and before target entry.

**Reason.** Preserving source context makes transition code direct and predictable.
The consequence is deliberate: §5.1 rejects writes and bindings whose destination is
destroyed by the selected transition.

## Cancellation is total over reference states

Relevant specification: [§7.2](SPEC.md#72-owned-spawned-instances) and
[§10.1](SPEC.md#101-engine-faults).

**Decision.** `cancel` disposes a live or retained-faulted owned runtime and succeeds
without effect for null or already-disposed references.

**Rejected alternative.** Accept cancellation only for a currently live target.

**Reason.** Automatic scope cleanup runs before state exit actions. A live-only rule
would make an exit-action cancellation fault merely because cleanup had already
disposed the same child.

## A holding reference bounds owned-child lifetime

Relevant specification: [§7.2](SPEC.md#72-owned-spawned-instances).

**Decision.** A child bound through `bind_to` is cancelled when the declaring scope of
that reference exits. Unbound children and children held by surviving scopes remain
owned until later completion or cancellation.

**Rejected alternative.** Require every transition leaving a holder scope to contain
an explicit cancellation action validated at load time.

**Reason.** Nullable references make mandatory explicit cancellation incorrect: a
valid transition would fault whenever no child had been spawned. Scope cleanup gives a
total, deterministic lifetime rule.

## Conformance has three authority tiers

Relevant specification: [§2](SPEC.md#2-conformance-parsing-and-format-identity) and the
[conformance-suite policy](https://github.com/fruwehq/determa-state-conformance#readme).

**Decision.** Core cases bind every conforming implementation. Profile cases bind only
implementations declaring that profile. Driver and harness mechanics bind no public
implementation interface.

**Rejected alternative.** Treat every file and helper in one flat suite as universally
authoritative whenever it differs from prose.

**Reason.** Portable state semantics, optional host surfaces, and test orchestration
have different consumers. Keeping their authority explicit prevents a harness detail
from becoming an accidental API.

## The command-line interface is a profile

Relevant specification: [§11.4](SPEC.md#114-hosting-profiles) and
[§13](SPEC.md#13-deliberately-unsupported-in-format-1).

**Decision.** Command-line behavior is optional profile work, not core format
semantics. The former in-process first-in-first-out fixtures were dropped;
`run_cli.py` remains non-normative infrastructure for a future profile.

**Rejected alternative.** Standardize the old command output and core-owned queue
model as universal conformance.

**Reason.** Queue ownership, stepping, and serialized command results are host
concerns. Standardizing the old fixtures would contradict the pure foreground core.

## Enabled-event inspection remains undefined

Relevant specification: [§12](SPEC.md#12-inspection-and-visualization).

**Decision.** Format 1 does not define a portable enabled-event inspection result.

**Rejected alternative.** Derive an enabled-event list from active configuration
alone.

**Reason.** A false guard is non-consuming and permits ancestor search. Enabledness is
therefore a property of the configuration, current variables, and a candidate
envelope—not the configuration alone. Any future inspection contract must preserve
that dependency.

## Portable scalar spellings are canonical

Relevant specification: [§2](SPEC.md#2-conformance-parsing-and-format-identity).

**Decision.** The only plain Boolean/null spellings are lowercase `true`, `false`, and
`null`. Plain `yes`, `no`, `on`, `off`, `y`, and `n` remain strings; noncanonical
Boolean/null spellings are rejected, and quoted tokens remain strings.

**Rejected alternative.** Delegate implicit scalar resolution to whichever YAML
library a host uses.

**Reason 1: identity preservation.** Treating `yes`/`no` and `on`/`off` as Booleans
collapses distinct YAML 1.2 state and event identifiers under YAML 1.1 tooling.

**Reason 2: canonicalization.** Rejecting `True` and `~` gives one source spelling per
portable Boolean/null value. This follows the same canonical-surface principle as
rejecting `local: false` instead of treating it as an alias for omission.

## Review guidance: enumerate quantified domains

This guidance is non-normative.

If a rule says “property P is valid for Q,” enumerate every member of Q and state the
otherwise behavior. A missing otherwise rule is a latent defect, especially for
lifecycle states, reference states, source/target relationships, and parser token
classes. Write the failing fixture first, then close the prose, schema, and assertion
contract around that exhaustive domain.

## Portable state references content-addressed definitions

Relevant specification: [§16.3](SPEC.md#163-complete-root-ownership-aggregate) and
[§16.5](SPEC.md#165-content-addressed-definition-registry).

**Decision.** An ordinary aggregate envelope stores the exact validated-bundle
fingerprint. The normalized definition is stored once in a content-addressed registry.
A separate package can carry verified definitions for transfer or recovery.

**Rejected alternative.** Embed the complete normalized definition in every aggregate
row.

**Reason.** Long-lived dormant aggregates need enough old declarative definition data
to validate and migrate, but duplicating it per row makes every deployment and backup
needlessly expensive. Content addressing preserves integrity and allows lazy migration
without retaining old host executable logic.

## Migration preserves target identity

Relevant specification: [§16.4](SPEC.md#164-immutable-identity-and-mutable-definition-binding).

**Decision.** Runtime identity origin and target identity are immutable. Migration
updates only the current definition and relation pointers.

**Rejected alternative.** Recompute component and spawned identities from the target
definition.

**Reason.** Already-created envelopes, outbox rows, nominal references, and external
correlation may contain the old target. Rekeying would silently invalidate committed
work unless every external reference were transactionally rewritten, which is outside
the root aggregate.

## Migration is declarative and action-free

Relevant specification: [§16.7](SPEC.md#167-immutable-declarative-migration-descriptors)
and [§16.9](SPEC.md#169-total-transform-matrix).

**Decision.** Descriptors are immutable data with a bounded closed CEL value profile.
Migration never executes machine actions, lifecycle behavior, arbitrary code, host
callbacks, or I/O.

**Rejected alternative.** Run entry/exit/transition code or an implementation-language
migration callback.

**Reason.** Portable migration must be retry-identical across languages and safe inside
one host transaction. Author behavior can emit effects or depend on runtime facilities,
and arbitrary callbacks cannot be independently validated by the conformance suite.

## Migration routes are exact and pinned

Relevant specification: [§16.8](SPEC.md#168-exact-route-and-migration-algorithm).

**Decision.** Deployment supplies one exact ordered list of trusted descriptor digests.
The engine never searches a migration graph.

**Rejected alternative.** Select the shortest or latest available path at runtime.

**Reason.** Registry contents and path-selection algorithms are host-dependent. Two
available valid paths must not make aggregate results nondeterministic.

## Incompatible state quarantines instead of guessing

Relevant specification: [§16.9](SPEC.md#169-total-transform-matrix) and
[§16.12](SPEC.md#1612-failure-rollback-quarantine-and-audit).

**Decision.** Deleted or incompatible active state requires a complete explicit
mapping. Otherwise the original aggregate is retained and quarantined. Version 1 has
no destructive reset migration.

**Rejected alternative.** Match by name, choose a surviving ancestor/initial state, or
restart the aggregate automatically.

**Reason.** Each fallback can discard business state or rerun side effects while
appearing to be a routine definition upgrade. Quarantine makes the unresolved case
observable and auditable without corrupting the committed source.

## Lazy migration shares the dispatch transaction

Relevant specification: [§16.11](SPEC.md#1611-lazy-transactional-host-ordering).

**Decision.** The complete route, optional dispatch, aggregate replacement, inbox
result, outbox emissions, and audit commit atomically under one aggregate lock.
Artifacts are resolved before the transaction.

**Rejected alternative.** Commit migration first and dispatch later, or process owned
children in separate transactions.

**Reason.** A committed definition advance with a rolled-back core result violates
fault and retry semantics. The root ownership aggregate remains one transactional
state boundary.
