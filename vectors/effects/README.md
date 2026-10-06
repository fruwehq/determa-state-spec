# Native effect host vectors

`empty-host-effect-journal-v1.json` is a closed host-owned journal paired with
`../persistence/execution-checkpoint-v1.json` at revision `0`. Its SHA-256 digest
uses the §19.2 domain and RFC 8785 JSON canonicalization. The empty record set
proves that merely having an execution checkpoint does not invent a provider call.

`result-identity-cases-v1.json` fixes the independent effect-result event domain and
names negative admission cases. The hash is over the three-element array in §19.4.
These cases require a matching committed intent, exact pinned mapping, current scope
authority and a live worker claim before a new worker outcome can commit. Recovery
admission of an already committed outcome uses the current host scope guard and
recorded evidence. A hash or token by itself is never authorization.

`host-native-effect-cases-v1.json` enumerates required conformance scenarios and
their observable admission, provider-call and mutation boundaries.

`committed-effect-checkpoint-v1.json` and `leased-host-effect-journal-v1.json`
form a checkpoint/journal shape and digest pair with one pinned intent. The fixture
starts from the persistence checkpoint and adds a synthetic committed external
emission for the host-boundary contract; it is not a replay of `source.yaml`.
`active-effect-claim-v1.json` shows the separate live authority record and
validates under both the §18 authority claim and §19 native effect claim schemas,
including its Unix nanosecond expiry. The result
request, envelope, and committed, replay, conflict, and ambiguous-report responses
pin response shapes and hashes. `result-admission-cases-v1.json` says exactly when a
new admission or mutation occurs. The two cancellation journal vectors show a
pre-claim prevented start with a durable declared `cancelled` outcome and a
post-call reconciliation requirement; they are alternative branches from the producing transaction, not sequential revisions of
the leased vector.

The committed response's acceptance receipt binds the exact normalized
`effect-result-envelope-v1.json`; it illustrates the admission boundary after the
leased snapshot. The synthetic result event declaration and provider binding are
stand-ins for a host conformance fixture; an implementation test MUST supply an exact
validated machine definition and a real selected core emission before claiming
end-to-end conformance.

`outcome-recorded-journal-v1.json`, `expired-effect-claim-v1.json`, and
`recovery-after-claim-expiry-v1.json` fix the crash window after a terminal
outcome commits but before admission. The host admits the recorded result under
current scope authority with zero new claims or provider calls, even at the old
claim expiry boundary. `effect-cancellation-cases-v1.json` fixes first commit, equal
replay, and missing-mapping refusal; the preclaim cancellation uses the exact
`effect-cancellation-request-v1.json` and committed response with a retained
operation-response digest. Test context supplies trusted Unix nanosecond time
explicitly; these vectors do not depend on the machine running the test.
