# Contributing to Determa State

**determa-state-spec** is the normative **specification** repository. It holds the prose spec
(`SPEC.md`), JSON Schemas (`schema/`), concise YAML examples (`examples/`), and canonical
artifacts/cases (`vectors/`) — text only. There is **no test or implementation code here** and **no
CI**; the executable correctness target lives in
[`fruwehq/determa-state-conformance`](https://github.com/fruwehq/determa-state-conformance) , and
the Python reference implementation in
[`fruwehq/determa-state-python`](https://github.com/fruwehq/determa-state-python) .

## How the repos split

The specification defines the contract. The language-independent conformance repository tests that
contract. Python and Rust are reference implementations checked against those shared tests.

Work repository by repository: specification review first, shared tests next, language
implementations after that. Issue #112 covers all remaining specification 0.3.0 redesign corrections
in one draft PR. Do not create component/correction issues or change another repository during this
pass. Applicable core cases retain their §2 authority; optional profiles and harness mechanics do
not define the core API.

## Source and generated data

`examples/` contains only a small human-readable YAML set. Authors write meaningful native/language
names, never executable hashes or dependency closures at use sites. `machine.schema.json` validates
source; `resolved-machine-v1.schema.json` validates generated executable definitions. Select the
stage explicitly, without dual readers. Both entrypoints reference
`machine-grammar-v1.schema.json`, whose dynamic anchors select explicit stage-specific slots. The shared grammar is not a standalone loader. Differences are confined to
runtime and custom-source slots. Generated locks pin exact source and executable
output.

Canonical JSON wire vectors stay under `vectors/` . Moving a vector preserves its coverage, with
deliberate regeneration when semantic inputs change. Do not convert JSON wire artifacts to YAML or
silently delete a negative case. Existing pinned validators require downstream path/grammar updates
after specification approval; do not claim an old engine implements the redesigned contract.

Validate schemas/references, every curated YAML, generated identity relationships, relative links,
event discard/ownership rules, vector inventory, unchanged VERSION, line widths and
`git diff --check` . Reflow prose/YAML to 100 columns; exact normative code/hash operands and
unbreakable URLs remain documented exceptions.

## Workflow

1.  Branch from `main` , open a Pull Request, and **squash-merge** — `main` stays linear.
2.  Resolve all review threads before merging.
3.  **Never push to `main` directly.**
4.  **No AI/assistant attribution anywhere** — not in commits, PR bodies, comments, or
   docs (no `Co-Authored-By:` , no "Generated with…"). Commits and PRs read as the
   author's own work.
5.  Spec edits should reference the SPEC section(s) they touch and link the related
   `determa-state-conformance` / `determa-state-python` issues.

## Versioning

This repository carries the synchronized version in `VERSION` (currently `0.3.0` ) and a matching
line at the top of `SPEC.md` .

> determa-state-spec, determa-state-conformance, and the implementations share one synchronized
> SemVer version (currently pre-1.0 `0.3.0`). A release tags all repos `vX.Y.Z` in lockstep; an
> implementation declares "implements Determa State spec vX.Y.Z" and pins conformance at that tag.

## License

Contributions are made under the project's [MIT license](LICENSE) .
