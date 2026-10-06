# Determa State

Determa State specifies portable statecharts: typed events, hierarchical states, CEL guards and
ordered actions. A developer writes the machine in readable YAML. Language implementations are
correct when they satisfy the same specification and language-neutral conformance tests.

This is the **unreleased, pre-alpha 0.3.0** specification. The only current machine format is
numeric `format: 1` . Superseded draft syntax has no compatibility reader.

## Start with a machine

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
        unlocked: { type: simple }
```

The small authored example set covers:

-  [A minimal guard](examples/minimal.yaml) .
-  [Hierarchy and history](examples/hierarchy-history.yaml) .
-  [UML-style event deferral](examples/event-deferral.yaml) .
-  [Components and owned spawned instances](examples/components-spawn.yaml) .
-  [An external timer request](examples/external-timer.yaml) .
-  [CEL, native providers and custom-language source](examples/native-custom-providers.yaml) .

`examples/` contains only human-facing YAML. None asks for a content hash, UUID, capability matrix,
dependency closure or JSON Pointer. Optional native/custom slots use meaningful names such as
`review` or `eligibility` . The host deliberately resolves those names once; evaluation never
silently follows a changed alias.

## Source is not a checkpoint

The specification separates four stages:

1.  **Authored source:** YAML declares machine behavior. CEL and structured actions
   are the recommended defaults; native providers and custom languages are opt-in.
2.  **Deployment configuration:** the application selects installed providers and
   named endpoint/scope bindings. Credentials remain outside portable values.
3.  **Generated executable definition:** explicit resolution generates an immutable
   provider lock and validated resolved definition. Loading never refreshes a lock.
4.  **Generated artifacts:** the engine/host produces state, receipts, checkpoints
   and archives in exact typed JSON. Canonical JSON bytes make hashes portable.

`validated_bundle_fingerprint` is a generated SHA-256 content identity, **not a UUID**. Users never
hand-author it. A hash proves content equality, not trust, authorization or whether a transaction
committed. See [§5.4](SPEC.md#54-exact-runtime-providers-and-optional-source-compilation) for resolution and
[§16](SPEC.md#16-portable-persistence-and-definition-migration) for portable artifacts and explicit
definition migration.

## Core and optional hosting

The portable core is a foreground state transform. It requires no daemon, executable, database,
broker, clock, scheduler or heartbeat. Its serializable aggregate owns configurations, variables,
components, spawned instances and accepted ready/deferred events. Applications may persist it in
their own transactions.

The core never silently deletes an event. Transport policies cannot remove an aggregate-owned
mailbox entry or committed internal failure notification. Deliberate pre-admission/post-terminal
discard requires a selected policy and observable identity, decision and reason, even in memory.
Durable/lossless profiles retain the stronger evidence they promise. Explicit lifecycle disposal
remains observable.

Stores, transports, HTTP endpoints, external effects, timers, authority and archives are optional
host/extension contracts. Bundled and third-party providers use the same registration rules. Native
I/O can weaken an opted-in invocation's guarantees; exact identity does not make arbitrary I/O pure
or transactional. Timers are external helpers, never implicit core scheduling. Archive participants
explicitly account for helper/application state. Authority is optional; safe transfer requires proof
rather than copied bytes. Embedded, self-hosted and future SaaS operation share the same public
contracts and multiple named endpoint/scope bindings.

## Repository map

-  [SPEC.md](SPEC.md) : normative verbal contract, authored model first.
-  [schema/](schema/) : source and generated-stage JSON Schemas.
-  [examples/](examples/) : the six concise human-authored YAML machines above.
-  [vectors/](vectors/README.md) : generated wire artifacts and exhaustive cases.
-  [DECISIONS.md](DECISIONS.md) : explanatory rationale, never overriding the contract.
-  [VERSION](VERSION) : unchanged synchronized version `0.3.0` .

The authored schema is [machine.schema.json](schema/machine.schema.json) . Generated definitions use
[resolved-machine-v1.schema.json](schema/resolved-machine-v1.schema.json) . Generated locks use
[provider-lock-v1.schema.json](schema/provider-lock-v1.schema.json) . Callers select the stage
explicitly; the engine does not guess or try old grammars.

Executable specification tests belong in
[determa-state-conformance](https://github.com/fruwehq/determa-state-conformance) . This repository
contains no engine, validation scripts or CI. Specification review precedes test/implementation
updates. This redesign does not claim that the existing engines already implement the changed source
syntax.

See [CONTRIBUTING.md](CONTRIBUTING.md) for review and validation expectations.
