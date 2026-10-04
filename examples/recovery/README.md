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
