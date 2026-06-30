# bs-005: MCP lifecycle (wire-level)

- **ID:** bs-005
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §11

## Scope

The wire-level MCP session lifecycle for the HTTP transport: the `initialize`
handshake, the advertised capabilities, server-side session-id assignment, and
the `notifications/initialized` confirmation. This covers ordering and the
JSON-RPC envelopes exchanged during connection setup, before any
`tools/list` or `tools/call` traffic (bs-006). Authentication and origin checks
that gate these requests are specified in [bs-004](bs-004-mcp-transport.md);
the pinned `protocolVersion` value is defined in [df-000](../data-formats/df-000-base.md).

## Specification (normative)

All JSON-RPC messages are POSTed to the MCP endpoint.

The lifecycle proceeds in order: the client sends `initialize`, the server
responds (assigning a session id), then the client sends the
`notifications/initialized` notification. A client MUST complete this handshake
before issuing other requests.

### `initialize` request

The client MUST send an `initialize` request whose `params` carry the pinned
`protocolVersion` (`2025-11-25`, see [df-000](../data-formats/df-000-base.md)), its declared
`capabilities`, and `clientInfo`. Example:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": { "tools": { } },
    "clientInfo": { "name": "example-client", "version": "0.0.1" }
  }
}
```

### `initialize` response

The HTTP response headers MUST include the assigned session id:

* `MCP-Session-Id: sess_...`

`serverInfo.name` is per-instance: by default it is auto-derived as
`dir2mcp-<slug>-<6-hex>` from the absolute path of the indexed directory so that
operators running many `dir2mcp` instances can distinguish them in their MCP
client list. Builds whose embedded version is recognized as a dev version
(specifically `0.0.0-dev` or `dev-<sha>[+dirty]`) use a
`dir2mcp-dev-<slug>-<6-hex>` prefix so local dev binaries can coexist with
brew-installed releases without identity collision. Other non-release builds,
including `go install` snapshots or pseudo-versions, still use the normal
`dir2mcp-<slug>-<6-hex>` prefix. It can be overridden via the `server.name` YAML
key or the `DIR2MCP_SERVER_NAME` env variable; overrides apply verbatim
regardless of build type.

The response body echoes the `protocolVersion`, advertises tool capabilities
(`tools.listChanged: false`), and carries `serverInfo` plus `instructions`:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "tools": { "listChanged": false }
    },
    "serverInfo": {
      "name": "dir2mcp-stas-legal-a1b2c3",
      "title": "dir2mcp: Directory RAG MCP Server",
      "version": "0.7.0"
    },
    "instructions": "Use tools/list then tools/call. Results include citations."
  }
}
```

> **Drift note (dir2mcp #404):** the `protocolVersion` is currently pinned to
> `2025-11-25` and echoed verbatim rather than negotiated against the client's
> requested value. Version negotiation is unimplemented; this note records the
> gap and does not change the specified behavior.

### `notifications/initialized`

After receiving the `initialize` response, the client MUST send the
`notifications/initialized` notification (a JSON-RPC notification, i.e. no `id`):

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

The server returns: HTTP 202.

## Changelog

- **0.1.0** — Migrated from SPEC.md §11. Made the handshake ordering and the
  per-message MUSTs (session-id header, `protocolVersion`, the `initialized`
  notification) explicit while preserving every wire-level detail and example.
  Cross-references rewired to doc IDs: auth/transport and origin checks (§10) →
  [bs-004](bs-004-mcp-transport.md); subsequent tool traffic (§12/§13) →
  [bs-006](bs-006-mcp-tools-list-call.md); the pinned `protocolVersion` / `format_version`
  concept (§1) → [df-000](../data-formats/df-000-base.md). Added a drift note for the
  unimplemented `protocolVersion` negotiation (dir2mcp #404).
