# Portable archive vectors

`archive-v1.json` is one complete version-1 archive with four selected roots. `server-1`
has ready and deferred accepted inputs with matching admission receipts. `fault-1`
retains a terminal deferred-capacity fault and its terminal receipt. `job-42` has a
compatible migration, exact descriptor and definition attachments, audit record, and
maintenance receipt. `effect-1` has an ambiguous pending intent, a full terminal
intent, and an effect tombstone, all anchored by its creation receipt. Each checkpoint
has its own verified revision and digest. The five normalized definitions and one
migration descriptor reproduce their declared fingerprints and digests.
The root object is the `determa.scope_archive` manifest. Its `archive_digest` is the
content identity; `members` lists every checkpoint, definition, descriptor, and
participant by sorted identity with exact canonical UTF-8 byte length and SHA-256.
`required_determa_capabilities` is the exact minimum for staging those members,
checked independently of source profile claims. Optional participant references
remain declared even when their payload is absent. Fence and transfer references
are null in these vectors and confer no authority.

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
checkpoint. `effect-1` has an ambiguous native attempt for its pending output and a
separate recorded business outcome for a terminal adapter-accepted output. That
result has not yet been admitted; adapter acceptance and business completion are
distinct. Its pinned route, attempt reports, outcome, result event, and retained
public operation response are all carried in the typed participant. The other
three journals are empty. The independently trusted
`archive-host-journal-inventory-v1.schema.json` commitment pins the complete IDs
and digests at the source consistency point, bound to source provenance and the
selected checkpoint pairs. Provider closure, payload, profile, contract, journal,
and inventory digests are pinned independently. Resealed missing and torn-journal
vectors refuse staging, as do resealed deletions of a native record, attempt,
outcome, or response reference.

`stage-cases-v1.json` contains 43 complete import requests, complete input archives,
configured trusted policy and provider evidence, exact result objects, and the exact
inert staged archive or null. Cases include external payload reconstruction; missing
required and absent optional participants; a resealed omission or weakened contract;
changed source provenance; and resealed omissions of deferred work, admission
receipts, pending/terminal/compact outbox evidence, migration audit, and faults.
It also has correctly resealed refusals for a wrong member byte length, identity, or
digest, a missing or extra member, an omitted required capability or optional
participant reference, and a destination that does not support a genuine required
capability.
`export-cases-v1.json` contains nine complete closed export requests, complete
closed source captures, and exact archives/results or refusals. It exercises embedded and external payload capture, a scoped source,
an explicit standalone source with and without participants, durable-native journal
capture, absent optional data, missing required capture, invalid selection, and an
inconsistent capture point.

`hash-checks-v1.json` pins the JCS SHA-256 of all seven archive/request/source/inventory
schemas, the closed fixture staging configuration, both participant payload schemas,
five definitions, four checkpoints, descriptor,
provider closures, source profiles, participant contracts, journals, authoritative
inventory, and outer archives. Each stage
case's `changed_paths_from_positive` lists every JSON pointer whose value differs
from the positive archive. Resealed semantic-negative cases prove that a fresh outer
digest does not make omitted queue, receipt, outbox, or audit evidence valid. The
positive archive, requests, source captures, and result vectors validate against the
closed schemas in `schema/`. JSON object source whitespace is immaterial; digest
input is RFC 8785 JCS with the domain arrays in SPEC §22.
