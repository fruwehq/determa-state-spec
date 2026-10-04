# Runtime provider contract examples

`mixed-cel-native.yaml` is schema-valid format 1. Its first branch is CEL and its
second branch binds exact native guard and action providers. The example's repeated
hex digits stand for illustrative content identities; a real loader must verify the
installed closure and fail before evaluation when the bytes do not match. Both native
slots disclose possible precommit external I/O. The example is therefore an explicit
weak embedded profile, not a deterministic-replay or semantic-inspection claim.
`guard-descriptor-v1.json` is the matching closed registration descriptor; a
registration with `kind: actions` for that Boolean binding is invalid.

`action-output.json` is a valid typed provider result: the engine checks its assign
and send against the selected statechart scope and event declarations, then applies
it before the following structured send. `invalid-action-output.json` has a dynamic
`stop` and is rejected by the closed provider-output schema.

`invalid-guard-output-type.json` and `invalid-provider-digest.json` are structural
negative bundles: a guard requires Boolean output and an exact lowercase SHA-256
provider digest. A structurally valid same-name provider with a different installed
digest instead fails executable closure resolution before creation or restoration.
An absent exact transitive dependency fails at the same boundary.

`language-source-v1.json` has a source region at the guard pointer in its template.
The exact compiler translates that region to the guard in `compiled-machine.json`.
`compilation-manifest-v1.json` binds the source digest and normalized generated
fingerprint. A complete generated definition restores against its runtime closure
without requiring this compiler. A changed manifest fingerprint or source/compiler
digest is rejected; restoration does not silently recompile.
