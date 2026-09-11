# Portable persistence vectors

These files are compact normative vectors for
[SPEC §16](../../SPEC.md#16-portable-persistence-and-definition-migration) and
[SPEC §17](../../SPEC.md#17-portable-execution-checkpoints-and-hosting-adapters).

- `source.yaml` and `target.yaml` are machine documents using numeric `format: 1`.
  They have the same aggregate shape; the target changes only `meta`.
- `aggregate-state-v2.json` is the sole portable aggregate example. It retains one
  ready and one deferred envelope in the addressed runtime and separates immutable
  acceptance identity from mutable queue placement.
- `compatible-migration-v2.json` is the sole direct migration-descriptor example. It
  advances the source definition to the target without a state transform and carries
  the required queue-preservation policy.
- `execution-checkpoint-v2.json` is the sole portable checkpoint example. Its embedded
  aggregate owns its runtime mailboxes; there is no duplicate host pending-work
  collection.
- `execution-checkpoint-v2-maintenance-cases.json` proves that an empty migration
  receipt remains verifiable after root tombstoning. It rejects a missing or altered
  target definition fingerprint and rejects an unsupported receipt wrapper.
- `target-identity-cases.json` fixes the three exact machine-format-1 target shapes
  using artifact decimal-string projections, including values beyond JavaScript's safe
  integer boundary and rejected malformed projections.

The source and target validated-bundle fingerprints are respectively
`sha256:cf1429c9cc0ecfb62e406bff29c9b537d668fad6601e30f0da0210986b7f6413`
and
`sha256:eeec154ca64b619fe802af811b942073a2bc988b669429d7d08748dd72ab6cfc`.
Their shared aggregate-shape fingerprint is
`sha256:2e66cdbbdcfe44dad5a2edd52e5c1693e28c6c60ae590682f4862ee56fe058cc`.
