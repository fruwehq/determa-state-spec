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

The core is a pure foreground transform from prior logical state plus one envelope to
new logical state plus ordered emissions. It has no built-in queue, clock, timer,
dead-letter store, broker, database, background worker, or external I/O.

A host chooses queue and effect plugins appropriate to its deployment: for example,
in-memory delivery, Redis, GCP Pub/Sub, or a transactional database inbox/outbox. Those
plugins own ordering, retries, acknowledgements, discard, capacity, and dead-letter
policy. Unhandled events do not accumulate in core state.

The optional execution-checkpoint profile standardizes the durable transaction boundary
around one root aggregate: accepted host/internal deliveries, operation receipts,
pending/terminal/compact outbox work, replay retention, root tombstones, migration
audit, and revision. Built-in and third-party execution stores use the same registration
path and advertise only capabilities their configured instance can prove; broker
integration is a composed host profile, not a storage capability.

Time-based behavior uses an external event-producing extension. A machine emits a
declared scheduling request and may later receive a declared correlated event. The
extension determines its clock, durability, delivery, cancellation, and credentials.

## Repository

This repository holds only the specification:

- [`SPEC.md`](SPEC.md) — normative semantics;
- [`schema/machine.schema.json`](schema/machine.schema.json) — structural JSON Schema;
- [`schema/aggregate-state.schema.json`](schema/aggregate-state.schema.json) — portable
  aggregate-state envelope;
- [`schema/migration-descriptor.schema.json`](schema/migration-descriptor.schema.json)
  — declarative definition migration;
- [`schema/aggregate-state-package.schema.json`](schema/aggregate-state-package.schema.json)
  — self-contained transfer package;
- [`schema/execution-checkpoint.schema.json`](schema/execution-checkpoint.schema.json)
  — portable durable-host checkpoint;
- [`examples/`](examples/) — schema-valid machine documents and normative vectors; and
- [`VERSION`](VERSION) — synchronized specification/package SemVer.

The executable correctness target lives in
[`fruwehq/determa-state-conformance`](https://github.com/fruwehq/determa-state-conformance).
Engine work follows the specification and conformance work in separate repositories.
