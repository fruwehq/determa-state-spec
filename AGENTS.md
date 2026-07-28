# AGENTS.md — determa-state-spec

Guidance for AI/coding agents working in this repository. (Tool-agnostic; not specific to any one assistant.)

## What this repo is
The **normative specification** for Determa State. Text only: `SPEC.md` (the prose spec),
strict schemas under `schema/`, normative vectors under `examples/`, and `VERSION`.
Portable persistence vectors live under `examples/persistence/`. **No implementation
code, no tests, no CI here.** The executable correctness
target lives in `determa-state-conformance`. Applicable normative core cases are
authoritative when they conflict with prose; profiles bind only implementations that
declare them, and harness mechanics bind no public implementation interface, as recorded
in `DECISIONS.md`.

## Determa in one paragraph
**Determa** is a family of tools for defining and running well-specified, verifiable
behavior. Its first product, **Determa State**, is a **language-agnostic statechart
engine** in the Harel/UML lineage. A bundle is declared once in YAML/JSON and run by an
implementation in any language; all implementations agree because they are validated
against one shared conformance suite. Guards and computed action values are written in
**CEL** (Common Expression Language). An umbrella `determa` launcher dispatches
`determa <product> …` to the `determa-<product>` binary on `PATH`.

## Repositories (GitHub org `fruwehq`, local folders under `~/src/personal/`)
| Repo / folder | Role |
|---|---|
| **determa-state-spec** (this) | normative spec — `SPEC.md`, `schema/`, `examples/`, `VERSION`. No CI. |
| determa-state-conformance | language-agnostic conformance suite + source/schema consistency CI. |
| determa-state-python | Python reference impl — dist `determa-state`, import `determa.state`. |
| determa-state-rust | Rust impl — crate `determa-state`, module `determa_state`. |
| determa | umbrella launcher monorepo — `python/` (PyPI), `rust/` (crates.io), `node/` (npm). |

## Working rules (apply in every Determa repo)
- **One issue → one PR.** Branch → PR → **squash-merge**. Linear history. Resolve all review threads. Never push to `main` directly (`main` is protected).
- **No AI/assistant attribution anywhere** — no `Co-Authored-By`, no "Generated with…", in commits, PR bodies, comments, or docs. Everything reads as the author's own work.
- **Conformance-first** for behavior changes: land the spec text here, then the matching case in `determa-state-conformance`, then the implementations.
- **Synchronized SemVer** across spec + conformance + both engines — currently **0.0.7**. (The `determa` launcher versions independently — currently 0.2.0.)
- **No new abbreviations** in JSON fields / public identifiers. The current grammar
  deliberately retains `init`, `lang`, `meta`, `on_events`, and
  `transition_to`.

## Making a change here
- Reference the SPEC section(s) you touch; link the related `determa-state-conformance` / impl issues.
- A version bump: edit `VERSION` **and** the `Spec version:` line at the top of `SPEC.md`, then tag `vX.Y.Z` after merge. Implementations pin the conformance suite at that tag.

## Pointers
- Prose spec: `SPEC.md` (§2 format identity, §4 grammar, §5 CEL, §6 transitions,
  §10 faults, §11 plugin boundary, §16 portable persistence/migration).
- Non-normative rationale and review guidance: `DECISIONS.md`. It never overrides
  `SPEC.md`, the schema, or conformance.
- Schemas: `schema/machine.schema.json`, `schema/aggregate-state.schema.json`,
  `schema/migration-descriptor.schema.json`, and
  `schema/aggregate-state-package.schema.json`. Examples: `examples/`.
