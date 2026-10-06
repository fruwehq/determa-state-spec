# Runtime provider contract vectors

`resolved-machine.json` is generated format 1, validated by
`schema/resolved-machine-v1.schema.json`. Its first branch is CEL and its second branch
binds exact native guard and action providers. The fixture's repeated hex digits stand for
illustrative content identities; a real loader must verify the installed closure and fail before
evaluation when the bytes do not match. Both native slots disclose possible precommit external I/O.
The fixture is therefore an explicit weak embedded profile, not a deterministic-replay or
semantic-inspection claim. `guard-descriptor-v1.json` is the matching closed registration
descriptor; a registration with `kind: actions` for that Boolean binding is invalid.

`action-output.json` is a valid typed provider result: the engine checks its assign and send against
the selected statechart scope and event declarations, then applies it before the following
structured send. Both external sends provide a non-empty portable correlation under §5.3; the CEL
send uses a string literal expression, and the provider result uses a typed string value.
`invalid-action-output.json` has a dynamic `stop` and is rejected by the closed provider-output
schema. `invalid-missing-correlation-action-output.json` is structurally valid but fails the §5.3
external-send correlation check against the same declared output event.

`invalid-guard-output-type.json` and `invalid-provider-digest.json` are structural negative bundles:
a guard requires Boolean output and an exact lowercase SHA-256 provider digest. A structurally valid
same-name provider with a different installed digest instead fails executable closure resolution
before creation or restoration. An absent exact transitive dependency fails at the same boundary.

`language-source-v1.json` has a source region at the guard pointer in its template. The exact
compiler translates that region to the guard in `compiled-machine.json` .
`compilation-manifest-v1.json` binds the source digest and normalized generated fingerprint. A
complete generated definition restores against its runtime closure without requiring this compiler.
A changed manifest fingerprint or source/compiler digest is rejected; restoration does not silently
recompile.

`multiple-send-action-output.json` repeats the same external send twice in one native slot after an
assignment. `multiple-send-identities.json` supplies the exact §9 hash operands and two distinct
expected IDs, local indexes and output sequences. The assignment consumes no emission index, and the
second send uses index `1` at the same containing action-element pointer. Restoring or replaying
that output must not collapse either intent or invent a nested proposal locator.

`invalid-multiple-send-identities.json` reuses the first ID for the second intent despite index `1`
; it is rejected by exact identity validation, rather than being treated as duplicate delivery.
`inert-provider-metadata.yaml` contains provider-like keys in metadata and a map variable's initial
value. They remain ordinary data; this definition loads without any installed native provider.
