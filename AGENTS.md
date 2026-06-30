# AGENTS.md

## Purpose

Operational guide for coding agents working in this repository.

## Before you start

- This repo is a **specification**, not an application — there is no code to
  build or run. Your changes are normative prose (Markdown) and JSON Schema.
- Read the context first: `README.md`, `docs/SPEC.md` (the primary normative
  spec), `spec/versioning.md` (compatibility policy), and — if you touch the
  restructured docs — `docs/specs/README.md` + `docs/specs/MIGRATION.md`.
- Confirm whether the area you are editing has already moved into the numbered
  `docs/specs/` docs (`df-*`/`bs-*`/`td-*`); avoid editing the same content in
  both `SPEC.md` and a migrated doc.
- Preserve existing tool/error contracts, structured field names, and MUST/SHOULD/
  MAY strength unless the change is the explicit point (and then bump versioning).

## Project summary

dirstral-spec is the canonical, implementation-neutral contract for the Dirstral
MCP ecosystem: protocol and tool contracts, JSON schemas, the error taxonomy, the
versioning/compatibility policy, and the x402 payment extension. Implementations
(e.g. `dir2mcp`) vendor it as a git submodule and must conform to it. MCP protocol
target: `2025-11-25`.

## Repo map

- `docs/SPEC.md` — primary normative specification
- `docs/VISION.md`, `docs/ECOSYSTEM.md` — direction & ecosystem context
- `docs/x402-payment-adapter-spec.md` — canonical x402 adapter spec
- `docs/design/` — numbered design notes / ADRs
- `docs/specs/` — in-progress numbered-doc restructure of `SPEC.md`
  (`data-formats/`, `behavior/`, `techniques/`) with a `README.md` index and a
  `MIGRATION.md` map
- `spec/versioning.md` — versioning & compatibility policy
- `spec/tools/schemas/*.json` — per-tool JSON Schemas + shared `common.json`
  (`Span`/`Hit`/`Citation`)
- `spec/sessions/lifecycle.md`, `spec/errors/taxonomy.md`, `spec/x402/extension.md`

## Validate before a PR

```bash
# JSON schemas must parse
for f in spec/tools/schemas/*.json; do python3 -m json.tool "$f" >/dev/null || echo "INVALID: $f"; done
# (docs/specs/) no dangling internal markdown links
```

The machine-readable schemas (especially `common.json`'s `Hit`/`Citation`) MUST
match the reference implementation's served MCP `outputSchema` — they have drifted
before and broken strict clients. Keep prose ↔ schema ↔ implementation in sync.

## Conventions

- **Spec-first governance:** behavior changes merge here first; the implementation
  re-pins its submodule afterward. The contract clients see is whatever commit the
  implementation pins.
- Breaking changes ⇒ major bump in `spec/versioning.md`.
- Cite stable doc IDs (`df-006`, `bs-007`) rather than mutable `SPEC.md` section
  numbers.
- Keep changes scoped; no unrelated files.
- Do **not** add a `Co-Authored-By` trailer to commits.
- Pushing from macOS may hang on the osxkeychain credential helper — use
  `git -c credential.helper= -c credential.helper='!gh auth git-credential' push`.
