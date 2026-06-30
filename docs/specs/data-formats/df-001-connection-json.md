# df-001: `connection.json`

- **ID:** df-001
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §4.3 (migrated; adds `format_version` per df-000 — see Changelog)

## Scope

`connection.json` is written into the state directory on `dir2mcp up`. It tells
an MCP client how to reach the server: the transport, the endpoint URL, the
headers to send (including the bearer token), and how the session id is assigned.
It carries no session id — that is assigned at `initialize` (see bs-005).

The state directory is always **local** ([df-000](df-000-base.md) invariant), so
`connection.json` lives next to the corpus's `.dir2mcp/`, not on a remote source.

## Specification (normative)

```json
{
  "format_version": "0.1.0",
  "transport": "mcp_streamable_http",
  "url": "http://127.0.0.1:52143/mcp",
  "headers": {
    "MCP-Protocol-Version": "2025-11-25",
    "Authorization": "Bearer <token-from-secret.token>"
  },
  "session": {
    "uses_mcp_session_id": true,
    "header_name": "MCP-Session-Id",
    "assigned_on_initialize": true
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `format_version` | string | The df-000 cross-version signal. A client that does not understand a **major**-incompatible value MUST refuse to connect rather than guess (df-000). |
| `transport` | string | `"mcp_streamable_http"`. |
| `url` | string | Absolute endpoint URL ending in `/mcp`. The host/port is the deterministic per-corpus listen address; a client MUST use this value verbatim and MUST NOT assume a fixed port. |
| `headers` | object | Headers the client MUST send on every request. |
| `headers."MCP-Protocol-Version"` | string | The pinned MCP protocol version (`2025-11-25`); see df-000 / bs-005. |
| `headers.Authorization` | string | `Bearer <token>`, the token from `secret.token` (df-002). |
| `session.uses_mcp_session_id` | boolean | Whether the server uses an `MCP-Session-Id`. |
| `session.header_name` | string | The session header name (`MCP-Session-Id`). |
| `session.assigned_on_initialize` | boolean | The session id is assigned by the server on `initialize`, not present in this file. |

### Security

`connection.json` embeds the bearer token, so it is **sensitive**: it MUST be
written with owner-only permissions (`0600`) into the local state directory and
MUST NOT be copied to a remote corpus source or logged. Rotating `secret.token`
(df-002) invalidates the `Authorization` value here; the file is rewritten on the
next `up`.

## Changelog

- **0.1.0** — Migrated from SPEC.md §4.3. Added the `format_version` field
  (df-000 / dir2mcp #468), a field table, and the security/permissions note.
  Cross-referenced `secret.token` as df-002 and the session lifecycle as bs-005.
