# Determa State

**Determa State** is a language-agnostic statechart specification in the Harel/UML
lineage. A bundle declares typed event contracts and one or more machines in YAML or
JSON. Independent engines agree on behavior by passing one shared normative
conformance suite.

The current document grammar is a pre-release alpha:

```yaml
format: 1
namespace: example.turnstile

events:
  coin:
    direction: input
    payload:
      amount: { type: int, required: true }

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
        unlocked: {}
```

Format 1 is intentionally not compatibility-stable yet. Earlier draft documents may
become invalid while the model is being designed.

Releases 0.0.1 through 0.0.6 used a frozen legacy grammar and snapshot format. They
cannot be converted to format 1: rewrite definitions as format-1 bundles, and do not
carry legacy snapshots forward. Release 0.0.7 introduced format 1, and its definitions
remain subject to ordinary current validation. Repository/package versions and machine
formats are separate; loaders never guess either from document shape. See
[SPEC.md §2](SPEC.md#2-conformance-parsing-and-format-identity).

## Unreleased 0.3.0

This alpha specification adds portable event deferral in aggregate-owned ready and
deferred mailboxes, version-1 maintenance migration and operation receipts, and exact
structural inspection of a selected runtime and envelope. Bounded semantic inspection
requires a safe nonmutating provider. Format-1 definitions may use exact runtime
guard/action provider slots beside CEL and structured actions; optional source
compilers emit a version-1 provenance manifest. Provider closure and configured
capabilities are checked explicitly. Native I/O during evaluation is a weaker
embedded profile whose guarantees must be reported.

The execution-checkpoint profile, lossless application projection, and coordinator-free
embedded facade bind complete dispositions and intents to an application-owned
transaction. Optional native effects retain committed routes, authenticated outcomes,
and explicit ambiguity, cancellation, and reconciliation evidence. Lossless delivery
binds ingress ownership, committed admission, terminal disposition, and outbound
responsibility; external brokers and dead-letter stores remain host integrations.
Optional host authority uses scoped epochs and refuses unproved topology without a
distributed coordinator. Optional external timers use a separately installed durable
helper; the core has no clock, scheduler, or wakeup.

Portable archives capture full selected checkpoints, immutable definitions, and
declared required or optional participant evidence, including trusted native-effect
journal inventory when applicable. Import stages inertly. Recovery distinguishes
read-only quarantine, explicit fresh-scope takeover, independent clone, and verified
local transfer; safe relocation is claimed only when the configured host proves it.
The common version-1 public client/host protocol pins named endpoint and scope
bindings, complete first responses, and a mechanical compatibility manifest and
change record for local and later hosted implementations.

Portable persistence artifacts and the public protocol use their sole version 1;
earlier draft artifact representations are unsupported. Machine documents retain
`format: 1`, independently of the unreleased package SemVer and artifact versions.

## Core model

- hierarchical states, entry/exit behavior, choices, and shallow/deep history;
- typed state-scoped variables;
- guards and computed action values in [CEL](https://cel.dev/);
- structured assign, send, refresh, spawn, cancel, and stop actions;
- isolated lifecycle-bound components and owned spawned runtimes;
- one-envelope atomic run-to-completion processing;
- deterministic event/effect identities and fault rollback; and
- host-facing typed input/output contracts with explicit correlation;
- canonical portable aggregate-state serialization; and
- portable durable execution checkpoints with capability-checked adapters; and
- explicit deterministic lazy migration between validated definitions.

Determa transition actions run while the source state and its variables are still
active, before source exit. The triggering event is visible to the selected handler,
guard, and transition action, but not to entry or exit actions.

## Host and plugin boundary

The default core evaluation is a pure foreground transform from prior logical state
plus one envelope to new logical state plus ordered emissions. Accepted events live in
aggregate-owned ready or deferred mailboxes. The core has no external broker, scheduler,
clock, timer helper, dead-letter store, database, background worker, or external I/O.
An explicitly installed native runtime provider may perform external I/O during an
embedded evaluation under the weaker capability profile in §5.4.

The public extension boundary uses exact provider references and configured-instance
capability reports. Hosts may inject objects directly or register bundled and third-party
factories through the same public path. Requested capabilities are checked before
mutation, and URI lookup never grants authority. A future Determa SaaS uses the same
public machine, client/host, checkpoint/archive, and conformance contracts; endpoints
and credentials stay in deployment configuration. See [SPEC.md §11.5](SPEC.md#115-public-extension-identity-registration-and-capabilities).

A host chooses queue and effect plugins appropriate to its deployment: for example,
in-memory delivery, Redis, GCP Pub/Sub, or a transactional database inbox/outbox. The
optional [lossless delivery profile](SPEC.md#21-lossless-event-delivery-profile)
defines source ownership, commit-before-acknowledgement, explicit retry and
backpressure, durable ingress dead letters, and retained terminal evidence. Accepted
events follow core mailbox order and disposition; an unhandled event has a terminal
receipt. In-memory use returns every decision to the caller for persistence.

The optional §22 archive profile snapshots selected complete checkpoints and exact
immutable definition attachments with separately declared application or helper
participants. Import verifies the closure and stages it inertly; activation and scope
transfer require separate protocols.

The optional execution-checkpoint profile standardizes the durable transaction boundary
around one root aggregate: accepted host/internal deliveries, operation receipts,
pending/terminal/compact outbox work, replay retention, root tombstones, migration
audit, and revision. Built-in and third-party execution stores use the same registration
path and advertise only capabilities their configured instance can prove; broker
integration is a composed host profile, not a storage capability.

The optional [host authority interface](SPEC.md#18-optional-host-scope-authority)
defines scope ownership, epochs and guards for configured hosts and plugins. A host
reports its proven topology and refuses safe relocation when it cannot prove source
retirement and transaction fate. The pure core remains coordinator-free; an embedding
application may commit its result in an application-owned transaction. Standalone
checkpoint export and import remain available without claiming safe relocation.
An optional [lossless application projection](SPEC.md#20-lossless-application-projection-and-embedded-transaction-facade)
loads explicitly selected application rows, admits declared typed input, and maps the
complete resulting state back into application-owned storage. Shared transaction
claims require one proved native transaction covering those rows and the checkpoint.

Time-based behavior uses an external event-producing extension. A machine emits a
declared scheduling request and may later receive a declared correlated event. The
optional [external timer helper](SPEC.md#23-optional-external-timer-helper) specifies
closed schedule, cancel, fire, clock, durability, and archive contracts. It is
installed explicitly and is never part of core evaluation. Durable timer records
join a complete §22 archive through a separately declared required participant
when selected roots depend on them.

## Repository

This repository holds only the specification:

- [`SPEC.md`](SPEC.md) — normative semantics;
- [`schema/delivery-v1.schema.json`](schema/delivery-v1.schema.json) and
  [`examples/delivery/delivery-v1-cases.json`](examples/delivery/delivery-v1-cases.json)
  with [admission](examples/delivery/execution-checkpoint-transfer-v1.json),
  [queue placement](examples/delivery/queue-placement-checkpoints-v1.json), and
  [outbound](examples/delivery/outbound-checkpoint-lifecycle-v1.json) checkpoints
  with [destination receipts](examples/delivery/outbound-destination-receipts-v1.json)
  — closed ownership, replay, disposition, and delivery evidence vectors;
- [`schema/machine.schema.json`](schema/machine.schema.json) — structural JSON Schema;
- [`schema/provider-reference-v1.schema.json`](schema/provider-reference-v1.schema.json)
  — exact executable provider identity;
- [`schema/runtime-provider-descriptor-v1.schema.json`](schema/runtime-provider-descriptor-v1.schema.json)
  — exact registration and slot-kind contract;
- [`schema/runtime-action-output-v1.schema.json`](schema/runtime-action-output-v1.schema.json)
  — checked native action proposals;
- [`schema/language-source-v1.schema.json`](schema/language-source-v1.schema.json) and
  [`schema/compilation-manifest-v1.schema.json`](schema/compilation-manifest-v1.schema.json)
  — optional exact source compilation and provenance;
- [`schema/inspection-v1.schema.json`](schema/inspection-v1.schema.json) — exact candidate inspection request and outcome;
- [`examples/inspection/`](examples/inspection/) — precedence, invalid-shape, and fuel-boundary vectors;
- [`schema/aggregate-state-v1.schema.json`](schema/aggregate-state-v1.schema.json) —
  portable queue-bearing aggregate-state envelope;
- [`schema/migration-descriptor-v1.schema.json`](schema/migration-descriptor-v1.schema.json)
  — declarative definition migration;
- [`schema/aggregate-state-package-v1.schema.json`](schema/aggregate-state-package-v1.schema.json)
  — self-contained transfer package;
- [`schema/execution-checkpoint-v1.schema.json`](schema/execution-checkpoint-v1.schema.json)
  — portable durable-host checkpoint;
- [`schema/archive-v1.schema.json`](schema/archive-v1.schema.json),
  [`schema/archive-participant-v1.schema.json`](schema/archive-participant-v1.schema.json),
  and [`schema/archive-result-v1.schema.json`](schema/archive-result-v1.schema.json)
  — closed portable archive, participant, and result;
- [`schema/archive-export-request-v1.schema.json`](schema/archive-export-request-v1.schema.json),
  [`schema/archive-import-request-v1.schema.json`](schema/archive-import-request-v1.schema.json),
  and [`schema/archive-export-source-v1.schema.json`](schema/archive-export-source-v1.schema.json)
  — closed archive requests and export-source evidence;
- [`examples/archives/`](examples/archives/) — complete staged and refusal vectors;
- [`schema/core-step-result-v1.schema.json`](schema/core-step-result-v1.schema.json) —
  closed core step result schema;
- [`schema/provider-reference-v1.schema.json`](schema/provider-reference-v1.schema.json)
  and [`schema/extension-descriptor-v1.schema.json`](schema/extension-descriptor-v1.schema.json)
  — exact provider identity and public registration descriptor;
- [`schema/extension-capability-report-v1.schema.json`](schema/extension-capability-report-v1.schema.json)
  and [`schema/extension-capability-requirement-v1.schema.json`](schema/extension-capability-requirement-v1.schema.json)
  — configured-instance claims and exact requirements;
- [`examples/`](examples/) — schema-valid machine documents and normative vectors; and
- [`examples/providers/`](examples/providers/) — mixed CEL/native slots, source
  compilation and output validation examples;
- [`schema/public-host-request-v1.schema.json`](schema/public-host-request-v1.schema.json) and
  [`schema/public-host-response-v1.schema.json`](schema/public-host-response-v1.schema.json)
  — closed public client/reference-host protocol;
- [`schema/public-host-contract-v1.json`](schema/public-host-contract-v1.json) with [`schema/public-host-contract-v1.schema.json`](schema/public-host-contract-v1.schema.json),
  [`schema/public-host-change-record-v1.schema.json`](schema/public-host-change-record-v1.schema.json), and
  [`examples/public-host/`](examples/public-host/) — compatibility boundary and golden messages;
- [`VERSION`](VERSION) — synchronized specification/package SemVer.

The executable correctness target lives in
[`fruwehq/determa-state-conformance`](https://github.com/fruwehq/determa-state-conformance).
Engine work follows the specification and conformance work in separate repositories.

The optional [§24 recovery contract](SPEC.md#24-recovery-fresh-scope-takeover-cloning-and-optional-relocation)
provides strict inactive quarantine, explicit fresh-scope standalone takeover,
independent clone activation, and guarded same-authority relocation where a host
positively proves it. See the closed [operation schema](schema/recovery-operation-v1.schema.json)
and [normative cases](examples/recovery/recovery-cases-v1.json).
