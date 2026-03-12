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

| Impl | Supported spec versions |
|------|------------------------|
| `dir2mcp` (Go) | `0.4.x` |
| `dirstral-cli` | `0.4.x` |
| `landfall` | TBD |

## Breaking change process

1. Open a spec PR with the proposed change
2. Maintainer review required (protocol council gate)
3. Bump major version in `spec/versioning.md`
4. All implementation repos must update their compatibility matrix before releasing against the new spec version
5. `dirstral-conformance` must add a new test suite for the new behavior

## Non-breaking additions

New optional tools or optional fields in existing tool schemas may be added in a minor version without breaking existing clients. Clients MUST ignore unknown fields.
