# CLAUDE.md

## Project

dirstral-spec is the **canonical, implementation-neutral specification** for the
Dirstral MCP ecosystem. It is the source of truth for the protocol and tool
contracts, the JSON schemas and error taxonomy, the versioning/compatibility
policy, and the x402 payment extension. It contains **no application code** —
only normative prose (Markdown) and machine-readable contract artifacts (JSON
Schema). Implementations (notably [`dir2mcp`](https://github.com/dirstral/dir2mcp))
**vendor this repo as a git submodule** and conform to it.

## Repository layout

- `docs/` — human-facing normative + explanatory docs
  - `SPEC.md` — the primary normative specification (output & integration contract)
  - `VISION.md`, `ECOSYSTEM.md` — direction and ecosystem context
  - `x402-payment-adapter-spec.md` — the canonical x402 adapter spec
  - `design/` — numbered design notes / ADRs (e.g. `0001-multi-provider.md`)
  - `specs/` — **in-progress** restructure of `SPEC.md` into small, numbered,
    independently-versioned docs (`data-formats/df-*`, `behavior/bs-*`,
    `techniques/td-*`); see `docs/specs/README.md` and `docs/specs/MIGRATION.md`.
    Until a document is marked **Stable** there, `SPEC.md` remains authoritative.
- `spec/` — machine-oriented contract artifacts
  - `spec/versioning.md` — versioning & compatibility policy
  - `spec/tools/schemas.md` + `spec/tools/schemas/*.json` — per-tool JSON Schemas
    and the shared `common.json` (`Span`, `Hit`, `Citation` definitions)
  - `spec/sessions/lifecycle.md`, `spec/errors/taxonomy.md`, `spec/x402/extension.md`

## Validate

There is no build. Before opening a PR:

```bash
# every JSON schema must parse
for f in spec/tools/schemas/*.json; do python3 -m json.tool "$f" >/dev/null || echo "INVALID: $f"; done

# (when touching docs/specs/) every internal markdown link must resolve — see the
# link-check used in the restructure PRs; no dangling df-/bs-/td- links.
```

A machine-readable schema and its prose MUST agree (see Known gotchas).

## Governance (read before changing normative content)

- Spec changes require maintainer review.
- **Breaking changes require a major version bump in `spec/versioning.md`.**
- Implementations MUST NOT diverge from this repo without explicit version
  negotiation.
- **Spec-first → re-pin loop:** a behavior change lands here *first* (a
  dirstral-spec PR merged to `main`), and only then does the implementation PR
  (e.g. in `dir2mcp`) re-pin its submodule to the new commit. The published
  contract clients see is whatever commit the implementation pins — so a fix here
  reaches users only after that re-pin.

## Working conventions

- Keep changes scoped; preserve existing tool/error contracts and structured field
  names unless the change is the point (and then bump versioning).
- Prefer stable, citable identifiers: cite `df-006` / `bs-007`, not mutable
  `SPEC.md §15.1.2` section numbers (the numbered-doc restructure exists for this).
- Normative strength matters: keep MUST/SHOULD/MAY exactly as intended.
- Do not add a `Co-Authored-By` trailer to commits.

## Known gotchas

- **Single source of truth.** `spec/tools/schemas/*.json` (especially
  `common.json`'s `Hit`/`Citation`/`Span`) MUST match the implementation's served
  MCP `outputSchema`. They drifted once (`chunk_id` typed as string, `rep`/`text`
  vs `rep_type`/`snippet`) and broke strict MCP clients — keep prose, schema, and
  the reference implementation in agreement.
- **MCP target is pinned:** `protocolVersion` `2025-11-25`. Network IDs are CAIP-2.
- **Pushing from macOS hangs on the osxkeychain credential helper.** Use the gh
  token instead:
  `git -c credential.helper= -c credential.helper='!gh auth git-credential' push -u origin <branch>`
- `SPEC.md` is large; the `docs/specs/` numbered docs are the migration target —
  check `docs/specs/MIGRATION.md` for what has moved before editing `SPEC.md`.

## PR checklist

- [ ] All JSON schemas parse; prose ↔ schema ↔ reference implementation consistent
- [ ] MUST/SHOULD/MAY strength preserved; structured field names unchanged (or versioned)
- [ ] `spec/versioning.md` bumped if the change is breaking
- [ ] If this changes behavior, the dependent implementation's submodule re-pin is planned
- [ ] (docs/specs/) no dangling internal links; doc-ID references used over section numbers
- [ ] No unrelated files changed
