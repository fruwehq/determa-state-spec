# Portable archive vectors

`archive-v1.json` is the complete version-1 archive. Its checkpoint contains one
ready input and one deferred input. Both have matching admission receipts, and its
creation receipt binds the same normalized definition attachment. The attachment is
an exact normalized variant of `../portable-event-deferral.yaml` whose initial state
is `busy.receiving`; its fingerprint, root runtime identity, envelope digests,
aggregate digest, checkpoint digest, participant payload/schema digests, and archive
digest are all independently reproducible from SPEC §§8, 9, 16, 17, and 22.

The fixture also carries `provider_closure_evidence` outside the archive. Its exact
`determa-archive-fixture-provider-1` digest reproduces the pinned participant provider
reference, including the declared empty dependency closure. Production providers use
their own verified executable-closure contract under §11.5.

The required `helper-state` participant has a complete embedded typed payload checked
against `helper-payload-v1.schema.json`. `stage-cases-v1.json` carries complete input
archives and complete expected results for both embedded and external payloads, exact
provider identity, optional absence, optional unsupported participation, missing
required bytes/provider/definition/checkpoint, changed queue or receipts, changed
schema, dependency cycle, and digest mismatch. Its `expected_staged_archive` is the
exact inert archive retained on success and null on refusal. `export-cases-v1.json`
has complete expected export artifacts and results. Configuration in these fixtures is
host evidence outside portable archive bytes; it grants no scope authority.

`hash-checks-v1.json` pins the JCS SHA-256 of each closed archive schema and the payload
schema, plus the positive definition, checkpoint, and archive digests. Each stage case's
`changed_paths_from_positive` lists every JSON pointer whose value differs from the
positive archive; resealed semantic-negative cases therefore prove that a fresh outer
digest does not make omitted queue or receipt evidence valid. The positive archive and
result vectors validate against `schema/archive-v1.schema.json` and
`schema/archive-result-v1.schema.json`. JSON object source whitespace is immaterial;
digest input is RFC 8785 JCS with the domain arrays given in §22.
