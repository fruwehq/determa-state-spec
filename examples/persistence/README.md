# Portable persistence vectors

These files are compact normative vectors for [SPEC §16](../../SPEC.md#16-portable-persistence-and-definition-migration).

- `source.yaml` and `target.yaml` have the same aggregate shape; the target changes
  only `meta`.
- `aggregate-state.json` is the human-readable running aggregate for `source.yaml`;
  `aggregate-state.canonical.json` contains its exact RFC 8785 bytes with no trailing
  newline.
- `compatible-migration.json` advances that aggregate to `target.yaml` without a
  state transform.
- `migrated-aggregate-state.json` is the human-readable result, and
  `migrated-aggregate-state.canonical.json` is its exact RFC 8785 byte result; identity
  and logical counters are preserved while current definition bindings and
  `migration_sequence` advance.
- `compatible-migration-audit.json` is the exact successful per-descriptor audit
  record.
- `faulted-aggregate-state.json` proves that the complete format-1 fault record,
  including `step_sequence`, survives the wire.
- `target-identity-cases.json` fixes the three exact format-1 target shapes and five
  rejected extra/missing-field regressions.
- `aggregate-state-package.json` carries the aggregate, both normalized definitions,
  the descriptor, and the exact one-hop route.

The source and target validated-bundle fingerprints are respectively
`sha256:cf1429c9cc0ecfb62e406bff29c9b537d668fad6601e30f0da0210986b7f6413`
and
`sha256:eeec154ca64b619fe802af811b942073a2bc988b669429d7d08748dd72ab6cfc`.
Their shared aggregate-shape fingerprint is
`sha256:2e66cdbbdcfe44dad5a2edd52e5c1693e28c6c60ae590682f4862ee56fe058cc`.
