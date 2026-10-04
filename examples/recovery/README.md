# Recovery vectors

`recovery-cases-v1.json` binds to the complete verified `../archives/archive-v1.json`
producer and its successful `complete_embedded_snapshot` staging case. The archive
contains four root checkpoints, including retained ready/deferred queues, receipts,
terminal effects, an ambiguous pending effect, a fault, migration audit and a required
helper participant. The `source_checkpoint_digests` and `source_archive_digest` are
checked before any recovery operation. The inherited-work list derives from the
archive outbox records; it is not separately trusted input.

Every case has a closed request, complete first result and, when mutation succeeds,
the complete resulting `recovery-record-v1` record. The `prior_record` links sequential
cases without rewriting source archive bytes. Equal retries replay the complete
first result and record. The unsupported transfer cases use the stock profile with
`safe_relocation: false`. The guard probes are read-only denials; their corresponding
native actions must enforce the same result at commit time.

The quarantine probes cover admission, processing, dispatch, helper fire/cancel,
ingress acknowledgement, promotion and clock timeout decisions. The replay case
repeats the first takeover request and its entire result; the conflict case changes
the namespace under the same operation ID. The partial batch shows one committed
quarantine beside an unsupported relocation refusal, with no cross-scope rollback.

The three early refusal vectors cover an unsupported interface, unsupported version,
and malformed recognized request. Their closed results contain null operation and
scope fields because parsing failed before a durable operation identity existed.

`archive-local-transfer-v1.json` is a complete resealed derivative of the same
four-root producer under a separately negotiated guarded local test profile. Its
source profile, participant contract and outer archive digests are recomputed and
pinned by the trusted local stage request and result in the cases file. The sorted
member manifest and required Determa capability set match the final §22 producer.
The archive carries only a retained `freeze_scope` response pointer; its null
transfer reference grants no retirement or activation authority. This test topology has one
authority domain; it asserts no distributed grant service. Its prepared proof binds
a frozen source and reserved destination. Only the committed proof binds §18 source
retirement and consumed grant. The positive sequence covers prepare, stage, commit
and activate; negative cases cover changed destination, stale generation, consumed
grant, source transaction in doubt, an old worker and an ambiguous provider retry.
The stock profile continues to advertise `safe_relocation: false` and refuses
unsupported transfer requests.
