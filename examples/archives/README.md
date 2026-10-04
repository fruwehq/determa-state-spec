# Portable archive vectors

`archive-v1.json` is one complete version-1 archive with four selected roots. `server-1`
has ready and deferred accepted inputs with matching admission receipts. `fault-1`
retains a terminal deferred-capacity fault and its terminal receipt. `job-42` has a
compatible migration, exact descriptor and definition attachments, audit record, and
maintenance receipt. `effect-1` has an ambiguous pending intent, a full terminal
intent, and an effect tombstone, all anchored by its creation receipt. Each checkpoint
has its own verified revision and digest. The five normalized definitions and one
migration descriptor reproduce their declared fingerprints and digests.

The archive records nonsecret `source` provenance and a complete required/optional
`participant_contract`. The source profile and contract digests are checked against
independent trusted host policy on import. A resealed artifact cannot omit the
required `helper-state` participant or weaken its advertised source profile. The
standalone cases use an explicit null scope, binding, and generation with no host
claims. The `standalone_core_without_participants` vectors export and stage all four
checkpoints with an empty required set and no provider. Scope provenance is not a
credential or authority grant.

The required `helper-state` participant has an embedded typed payload validated by
`helper-payload-v1.schema.json`. Fixture `provider_closure_evidence` is outside the
archive; its `determa-archive-fixture-provider-1` digest reproduces the pinned provider
reference and declared empty dependency closure. Production providers verify their
actual executable closure under §11.5.

`host-journal-payload-v1.schema.json` closes the typed envelope for the conditional
§19 participant. The durable-native positive vector has one required participant
with four decoded, closed journals in root order, each digest-paired to its selected
checkpoint. All four are empty because this source snapshot has no §19 native-effect
record; the `effect-1` outbox work is §17 external output. Provider closure, payload,
profile, contract, and journal digests are pinned independently. Resealed missing and
torn-journal vectors refuse staging even though their outer hashes are valid.

`stage-cases-v1.json` contains 31 complete import requests, complete input archives,
configured trusted policy and provider evidence, exact result objects, and the exact
inert staged archive or null. Cases include external payload reconstruction; missing
required and absent optional participants; a resealed omission or weakened contract;
changed source provenance; and resealed omissions of deferred work, admission
receipts, pending/terminal/compact outbox evidence, migration audit, and faults.
`export-cases-v1.json` contains nine complete closed export requests, complete
closed source captures, and exact archives/results or refusals. It exercises embedded and external payload capture, a scoped source,
an explicit standalone source with and without participants, durable-native journal
capture, absent optional data, missing required capture, invalid selection, and an
inconsistent capture point.

`hash-checks-v1.json` pins the JCS SHA-256 of all six archive/request/source
schemas, the closed fixture staging configuration, both participant payload schemas,
five definitions, four checkpoints, descriptor,
provider closures, source profiles, participant contracts, journals, and outer archives. Each stage
case's `changed_paths_from_positive` lists every JSON pointer whose value differs
from the positive archive. Resealed semantic-negative cases prove that a fresh outer
digest does not make omitted queue, receipt, outbox, or audit evidence valid. The
positive archive, requests, source captures, and result vectors validate against the
closed schemas in `schema/`. JSON object source whitespace is immaterial; digest
input is RFC 8785 JCS with the domain arrays in SPEC §22.
