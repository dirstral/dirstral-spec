# Session Lifecycle

**Spec version:** `0.4.0`
**MCP protocol target:** `2025-11-25`

## Transport

`dir2mcp` supports two transports:

| Transport | Value | Description |
|-----------|-------|-------------|
| Streamable HTTP | `streamable-http` | Default. HTTP POST with session ID tracking via `MCP-Session-Id` header |
| stdio | `stdio` | For subprocess/embedding use. No session headers. |

## Streamable HTTP session lifecycle

```
Client                            Server
  |                                 |
  |  POST /mcp                      |
  |  MCP-Protocol-Version: 2025-11-25
  |  {"method": "initialize", ...}  |
  |  ─────────────────────────────> |
  |                                 |
  |  200 OK                         |
  |  MCP-Session-Id: <uuid>         |
  |  {"result": {"capabilities": ...}}
  |  <───────────────────────────── |
  |                                 |
  |  POST /mcp                      |
  |  MCP-Session-Id: <uuid>         |
  |  {"method": "notifications/initialized"}
  |  ─────────────────────────────> |
  |                                 |
  |  202 Accepted                   |
  |  <───────────────────────────── |
  |                                 |
  |  POST /mcp                      |
  |  MCP-Session-Id: <uuid>         |
  |  {"method": "tools/call", ...}  |
  |  ─────────────────────────────> |
  |                                 |
  |  200 OK                         |
  |  {"result": {...}}              |
  |  <───────────────────────────── |
```

## Required headers

| Header | Direction | Value |
|--------|-----------|-------|
| `MCP-Protocol-Version` | Client → Server | `2025-11-25` |
| `MCP-Session-Id` | Server → Client (initialize response) | UUID |
| `MCP-Session-Id` | Client → Server (subsequent requests) | Same UUID |

## Session recovery

If a client receives `SESSION_NOT_FOUND` (`-32002`), it MUST:
1. Re-send `initialize` without a session ID
2. Re-send `notifications/initialized`
3. Retry the failed request with the new session ID

Servers MUST NOT silently drop `SESSION_NOT_FOUND` — it must propagate as a JSON-RPC error.

## stdio session lifecycle

stdio transport uses newline-delimited JSON-RPC with `Content-Length` framing:

```
Content-Length: <N>\r\n
\r\n
<N bytes of JSON>
```

No session ID is issued or required. Session state is implicit in the subprocess lifetime.

## Protocol version negotiation

Clients MUST send `MCP-Protocol-Version: 2025-11-25` on every request.
Servers MUST reject requests with an unsupported protocol version.
The `initialize` request body MUST also include `"protocolVersion": "2025-11-25"`.
