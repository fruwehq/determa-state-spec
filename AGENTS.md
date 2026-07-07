# AGENTS.md — determa-state-spec

Guidance for AI/coding agents working in this repository. (Tool-agnostic; not specific to any one assistant.)

## What this repo is
The **normative specification** for Determa State. Text only: `SPEC.md` (the prose spec),
`schema/machine.schema.json` (structural JSON Schema for a machine), `examples/`, and
`VERSION`. **No implementation code, no tests, no CI here.** The executable correctness
target lives in `determa-state-conformance`; where prose and the suite disagree, **the
suite wins** (SPEC §2) and a bug is filed against this document.

## Determa in one paragraph
**Determa** is a family of tools for defining and running well-specified, verifiable
behavior. Its first product, **Determa State**, is a **language-agnostic statechart
engine** (Harel/UML lineage, PSiCC run-to-completion semantics): a machine is declared
once in YAML/JSON and run by an implementation in any language; all implementations agree
because they are validated against one shared conformance suite. Guards and computed
action values are written in **CEL** (Common Expression Language). An umbrella `determa`
launcher dispatches `determa <product> …` to the `determa-<product>` binary on `PATH`.

## Repositories (GitHub org `fruwehq`, local folders under `~/src/personal/`)
| Repo / folder | Role |
|---|---|
| **determa-state-spec** (this) | normative spec — `SPEC.md`, `schema/`, `examples/`, `VERSION`. No CI. |
| determa-state-conformance | the language-agnostic conformance suite (the arbiter of correctness). No CI. |
| determa-state-python | Python reference impl — dist `determa-state`, import `determa.state`. |
| determa-state-rust | Rust impl — crate `determa-state`, module `determa_state`. |
| determa | umbrella launcher monorepo — `python/` (PyPI), `rust/` (crates.io), `node/` (npm). |

## Working rules (apply in every Determa repo)
- **One issue → one PR.** Branch → PR → **squash-merge**. Linear history. Resolve all review threads. Never push to `main` directly (`main` is protected).
- **No AI/assistant attribution anywhere** — no `Co-Authored-By`, no "Generated with…", in commits, PR bodies, comments, or docs. Everything reads as the author's own work.
- **Conformance-first** for behavior changes: land the spec text here, then the matching case in `determa-state-conformance`, then the implementations.
- **Synchronized SemVer** across spec + conformance + both engines — currently **0.0.6**. (The `determa` launcher versions independently — currently 0.2.0.)
- **No abbreviations** in JSON fields / public identifiers (full words: `definition`, not `def`). Deliberately **kept** for now: `config`; the machine-language keywords `esvs`/`on_events`/`top`/`transition_to`; and the snapshot keys `def_id`/`def_version` and `spawn.def`. Renaming any of those is a separate, explicitly-approved migration (it touches every machine file).

## Making a change here
- Reference the SPEC section(s) you touch; link the related `determa-state-conformance` / impl issues.
- A version bump: edit `VERSION` **and** the `Spec version:` line at the top of `SPEC.md`, then tag `vX.Y.Z` after merge. Implementations pin the conformance suite at that tag.

## Pointers
- Prose spec: `SPEC.md` (§2 conformance, §4 grammar, §5 semantics, §6 CEL, §13 CLI/JSON, §14 introspection).
- Schema: `schema/machine.schema.json`. Examples: `examples/`.
