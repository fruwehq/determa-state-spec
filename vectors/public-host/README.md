# Public execution-host protocol vectors

`positive-v1.json` contains complete version-1 requests and responses, with the canonical JCS
request-hash operand and digest for every case. `negative-v1.json` contains valid protocol requests
that must be refused by operation semantics, invalid request shapes, and invalid response shapes. A
null `expected_response` means that a malformed request, transport denial, or client binding refusal
produces no valid public response. `transport_context` and `client_context` describe the surrounding
invocation and are not protocol fields.

The committed creation/read vectors contain the exact portable checkpoint from
[`../persistence/execution-checkpoint-v1.json`](../persistence/execution-checkpoint-v1.json).
Committed admission and processing contain the exact before/after checkpoint snapshots, acceptance
receipt, and terminal receipt from
[admission and processing snapshots](../delivery/execution-checkpoint-transfer-v1.json).
Deferred and recalled processing use the exact checkpoint snapshots in
[`../delivery/queue-placement-checkpoints-v1.json`](../delivery/queue-placement-checkpoints-v1.json)
and retain public commit receipts even where deferral has no terminal receipt. The authority read
and capability report copy the closed normative values in
[`../authority/host-authority-cases-v1.json`](../authority/host-authority-cases-v1.json) and
[authority profile cases](../authority/host-authority-profile-cases-v1.json).
Accepted mutating authority cases also carry full public receipts; the read-only authority case has
none. Invalid response cases exercise rejected authority values, missing mutation receipts, and a
resealed acceptance-only receipt on a committed response. The successful structural inspection of a
resolved candidate copies the normative synthetic snapshot question and answer from
[`../inspection/shape-and-precedence-v1.json`](../inspection/shape-and-precedence-v1.json); it
checks public binding of that inspection result, while the separate absent-target case binds a real
checkpoint. The effect result and preclaim cancellation cases copy [`../effects/`](../effects/)
request and result artifacts. The archive export and inert import cases carry complete archive bytes
and exact closed results from [`../archives/export-cases-v1.json`](../archives/export-cases-v1.json)
and [`../archives/stage-cases-v1.json`](../archives/stage-cases-v1.json). The destination capability
report uses its independently configured supported set. The refusal case carries the complete §22
result in the public error, with no stage receipt. The recovery cases copy complete §24 requests,
results, and retained destination records from
[`../recovery/recovery-cases-v1.json`](../recovery/recovery-cases-v1.json). They cover quarantine,
fresh-scope takeover, clone, verified local activation, and exact refusals for active strict
restore, weak clone isolation, and unsupported transfer topology. The timer cases copy the complete
§23 helper requests and results from
[`../timers/timer-helper-cases-v1.json`](../timers/timer-helper-cases-v1.json). They cover schedule,
cancel, due claim, committed fire, read, and exact collision, early claim, stale fence, and
clock-unavailable refusals.

These are specification vectors. Engine/client/reference-host execution checks live in the
conformance and implementation repositories. `compatibility-change-v1.json` records this first
public boundary; its fingerprints are recomputed when a recorded source changes.

`source-order-compatibility-change-v1.json` records issue #104: the transport descriptor permits the
already specified `source_ordered` capability. Request and response schema fingerprints are
unchanged; the manifest records the corrected descriptor bytes. The ordering proof requirements in
§21.3 remain mandatory.

`timer-evidence-compatibility-change-v1.json` records issue #106: equal timer completion replay and
the archived schedule correlation now match their immutable requests. The request and response
schema fingerprints are unchanged; the manifest tracks the corrected timer vectors and archive
bytes.

`provider-correlation-compatibility-change-v1.json` records issue #108: the positive native-provider
vectors now carry the required external-send correlation. The public request and response schemas
are unchanged; the manifest fingerprints the corrected fixture bytes.

`native-slot-identities-compatibility-change-v1.json` records issue #110: multiple native send
proposals use distinct continuing slot-local ordinals and exact containing-slot locators;
provider-like keys in inert data cannot install or select code. Public message schemas remain
version 1 and unchanged. The manifest binds the clarified prose and complete positive/negative
identity vectors.
