# Versioning Policy

## Spec versioning

The spec uses [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

| Change type | Version bump |
|-------------|-------------|
| Breaking wire/schema behavior | Major |
| New optional fields, new optional tools | Minor |
| Clarifications, doc fixes | Patch |

**Pre-1.0 (beta) policy.** While the spec is `0.x` the project is pre-institutional and treated as **beta**: the `MAJOR` component stays `0`, *breaking* wire/schema changes bump the `MINOR` (e.g. `0.4.0 → 0.5.0`), and non-breaking additions/clarifications bump the `PATCH`. The table above describes post-`1.0` semantics and takes effect when the spec reaches `1.0.0`.

**Current spec version:** `0.5.0`
**MCP protocol target:** `2025-11-25`

## Implementation compatibility

Each implementation declares the spec version(s) it supports. `dirstral-cli` validates the supported spec version at runtime during `initialize`.

## Compatibility matrix

| Impl | Supported spec versions | Notes |
|------|------------------------|-------|
| `dir2mcp` (Go) | `0.5.x` | Reference implementation used for spec validation; reviewed against `internal/mcp/` as of 2026-04-05. The spec is authoritative — when discrepancies arise, maintainers file a spec-gap issue and decide whether to correct the spec or the implementation. |
| `dirstral-cli` | `0.4.x` | MUST update to `0.5.x` before releasing against spec `0.5.0` (no client code change required — result contract is unchanged; tool names are the only wire-visible delta). |
| `landfall` | TBD | |

## Contract freeze (issue #104)

As of spec version `0.4.0`, the following machine-readable artifacts have been added:

- `spec/tools/schemas/` — JSON Schema Draft-07 files for all 9 tools
- `spec/errors/taxonomy.md` — complete error code table including tool-execution errors
- `spec/sessions/lifecycle.md` — session expiry and `X-MCP-Session-Expired` header documented
- `spec/x402/extension.md` — `upto` scheme and `maxAmountRequired` field documented

Spec gaps identified during the review (see `<!-- spec-gap: ... -->` comments in each file):

- `SESSION_NOT_FOUND` JSON-RPC code was documented as `-32002`; implementation uses `-32001`
- `UNAUTHORIZED` JSON-RPC code was documented as `-32001`; implementation uses `-32000`
- Error `data` envelope (`{"code": ..., "retryable": ...}`) was not documented
- Tool execution errors return HTTP 200 with `isError: true`; this was not explicitly stated
- Several error codes (`MISSING_FIELD`, `INVALID_FIELD`, `INVALID_RANGE`, `STORE_CORRUPT`, `INTERNAL_ERROR`, `FORBIDDEN_ORIGIN`, `METHOD_NOT_FOUND`) were absent from the taxonomy

## 0.5.0 — reconcile shipped dir2mcp (spec-gap resolution)

Protocol-council decision: the dir2mcp reference implementation had shipped behavior that diverged from canonical `0.4.0`. Per the pre-1.0 beta policy and the "spec is authoritative; maintainers decide spec-vs-impl direction" rule, all of the following were resolved **impl → spec** (the spec now ratifies shipped behavior); breaking deltas bump `MINOR` (`0.4.0 → 0.5.0`):

- **Tool naming** `dir2mcp.<tool>` → `dir2mcp_<tool>` (breaking; ratifies dir2mcp #172). The former dotted-namespace rule is **superseded** — underscore form is canonical across `docs/SPEC.md`, `spec/tools/schemas.md`, and every `spec/tools/schemas/*.json` title.
- **`rep_type` enum** `ocr_markdown` → `extracted_markdown` (breaking; ratifies dir2mcp #152 docling extractor abstraction).
- **`k` default** `10` → `15` for `search`/`ask`/`ask_audio`/`transcribe_and_ask` (ratifies dir2mcp #163).
- **`OCR_NOT_READY`** tool-execution error added + `open_file` binary-doc semantics + `span.kind="document"` variant (new optional; ratifies dir2mcp #180).
- **`serverInfo.name`** per-instance auto-derivation + `dir2mcp-dev-` prefix for dev builds (new optional; ratifies dir2mcp #184/#185).
- **x402 adapter**: facilitator defaults to the Coinbase x402 Go SDK client (clarification).

`dirstral-conformance` SHOULD extend suites for the renamed tool surface before any impl releases against `0.5.0`.

## Breaking change process

1. Open a spec PR with the proposed change
2. Maintainer review required (protocol council gate)
3. Bump the version in `spec/versioning.md` (while `0.x`: breaking → `MINOR` per the pre-1.0 policy; post-`1.0`: `MAJOR`)
4. All implementation repos must update their compatibility matrix before releasing against the new spec version
5. `dirstral-conformance` must add a new test suite for the new behavior

## Non-breaking additions

New optional tools or optional fields in existing tool schemas may be added in a minor version without breaking existing clients. Clients MUST ignore unknown fields.
