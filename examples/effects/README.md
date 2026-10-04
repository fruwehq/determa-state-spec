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
