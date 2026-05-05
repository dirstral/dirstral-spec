# Versioning Policy

## Spec versioning

The spec uses [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

| Change type | Version bump |
|-------------|-------------|
| Breaking wire/schema behavior | Major |
| New optional fields, new optional tools | Minor |
| Clarifications, doc fixes | Patch |

**Current spec version:** `0.4.0`
**MCP protocol target:** `2025-11-25`

## Implementation compatibility

Each implementation declares the spec version(s) it supports. `dirstral-cli` validates the supported spec version at runtime during `initialize`.

## Compatibility matrix

| Impl | Supported spec versions | Notes |
|------|------------------------|-------|
| `dir2mcp` (Go) | `0.4.x` | Reference implementation used for spec validation; reviewed against `internal/mcp/` as of 2026-04-05. The spec is authoritative — when discrepancies arise, maintainers file a spec-gap issue and decide whether to correct the spec or the implementation. |
| `dirstral-cli` | `0.4.x` | |
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

## Breaking change process

1. Open a spec PR with the proposed change
2. Maintainer review required (protocol council gate)
3. Bump major version in `spec/versioning.md`
4. All implementation repos must update their compatibility matrix before releasing against the new spec version
5. `dirstral-conformance` must add a new test suite for the new behavior

## Non-breaking additions

New optional tools or optional fields in existing tool schemas may be added in a minor version without breaking existing clients. Clients MUST ignore unknown fields.
