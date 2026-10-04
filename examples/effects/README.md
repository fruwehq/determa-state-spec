# Native effect host vectors

`empty-host-effect-journal-v1.json` is a closed host-owned journal paired with
`../persistence/execution-checkpoint-v1.json` at revision `0`. Its SHA-256 digest
uses the §19.2 domain and RFC 8785 JSON canonicalization. The empty record set
proves that merely having an execution checkpoint does not invent a provider call.

`result-identity-cases-v1.json` fixes the independent effect-result event domain and
names negative admission cases. The hash is over the three-element array in §19.4.
These cases require a matching committed intent, exact pinned mapping, current scope
authority and worker claim before a positive result can be admitted. A hash or token
by itself is never authorization.

`host-native-effect-cases-v1.json` enumerates required conformance scenarios and
their observable admission, provider-call and mutation boundaries.

`committed-effect-checkpoint-v1.json` and `leased-host-effect-journal-v1.json`
form a checkpoint/journal shape and digest pair with one pinned intent. The fixture
starts from the persistence checkpoint and adds a synthetic committed external
emission for the host-boundary contract; it is not a replay of `source.yaml`.
`active-effect-claim-v1.json` shows the separate live authority record. The result
request, envelope, and committed, replay, conflict, and ambiguous-report responses
pin response shapes and hashes. `result-admission-cases-v1.json` says exactly when a
new admission or mutation occurs. The two cancellation journal vectors show a
pre-claim prevented start and a post-call reconciliation requirement; they are
alternative branches from the producing transaction, not sequential revisions of
the leased vector.

The committed response's acceptance receipt binds the exact normalized
`effect-result-envelope-v1.json`; it illustrates the admission boundary after the
leased snapshot. The synthetic result event declaration and provider binding are
stand-ins for a host conformance fixture; an implementation test MUST supply an exact
validated machine definition and a real selected core emission before claiming
end-to-end conformance.
