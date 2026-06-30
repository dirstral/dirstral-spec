# bs-004: MCP Streamable-HTTP transport

- **ID:** bs-004
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §10

## Scope

The HTTP transport for the dir2mcp MCP server, using the MCP Streamable HTTP
binding pinned to protocol version `2025-11-25` (the same pin and `format_version`
concept governed by [df-000](../data-formats/df-000-base.md)). Covers the
endpoint, required headers, session lifecycle, notification handling, Origin
checks (DNS-rebinding mitigation), and bearer-token auth. The JSON-RPC tool
surface carried over this transport is defined in [bs-005](bs-005-mcp-lifecycle.md)
(`search`/`ask`/`open_file`) and the error model in
[df-008](../data-formats/df-008-error-taxonomy.md).

## Specification (normative)

### Endpoint

- Default MCP path: `/mcp`.
- `POST` accepts JSON-RPC messages: a single object. Batch arrays MAY be accepted
  optionally.

### Required headers

Clients MUST send:

- `MCP-Protocol-Version: 2025-11-25` (after initialization).
- `Authorization: Bearer <token>` (unless auth is disabled).
- `Accept: application/json, text/event-stream` (recommended).

The server returns:

- `MCP-Session-Id: <id>` on the `initialize` response.

### Sessions

- On `initialize` success, the server assigns a session id and returns it in
  `MCP-Session-Id`.
- The client MUST include `MCP-Session-Id` on subsequent requests.
- Sessions are stateful resources with a defined lifecycle:
  - **Inactivity timeout:** a session SHOULD expire if the server has not seen
    any requests using that `MCP-Session-Id` for a configurable period. The
    reference implementation defaults to 24 hours of inactivity (matching the
    previous hardcoded `sessionTTL`), though some deployments may prefer shorter
    windows such as 30 minutes. Servers SHOULD expose a configuration parameter
    (e.g. `session_inactivity_timeout` as a YAML duration) so operators can
    adjust the value.
  - **Absolute lifetime (optional):** servers MAY enforce a maximum absolute
    duration (e.g. 24 hours) after which the session expires regardless of
    activity. In the reference implementation this is governed by
    `session_max_lifetime` (YAML duration); a zero value disables the limit.
  - **Cleanup/eviction:** expired sessions MUST be evicted or garbage-collected
    from the server's in-memory or persisted session store. Cleanup can run
    lazily on access or via a periodic background task; the key requirement is
    that an expired `MCP-Session-Id` is treated as unknown.
  - **Logging & visibility:** servers SHOULD log session expiration events,
    including the reason (inactivity vs. lifetime) and the session id. Responses
    MAY include a diagnostic header such as
    `X-MCP-Session-Expired: inactivity|max-lifetime`.
- Unknown or expired session id:
  - The server returns HTTP `404`. This is the same status used for any
    non-existent session; clients SHOULD treat both cases identically even if a
    diagnostic header is present.
  - The client MUST re-initialize by issuing a fresh `initialize` request. The
    previous id is discarded and a new `MCP-Session-Id` will be returned. Clients
    SHOULD treat a `404` as indicating that they should restart the flow rather
    than retrying.
- **Production guidance:**
  1. Choose default timeout values appropriate for your workload and security
     requirements. Public-facing servers often use shorter inactivity timeouts to
     conserve resources.
  2. Expose configuration knobs for both inactivity and absolute lifetime.
     Document defaults in your service README.
  3. Surface expiration reasons in logs and, optionally, response headers to
     assist operators and clients.
  4. Implement robust cleanup to avoid unbounded session growth; periodic
     eviction or TTL caches are recommended.

### Notifications

If a `POST` is a JSON-RPC notification (no `id`), and accepted:

- The server returns HTTP `202 Accepted` and no body.

### Origin checks (DNS-rebinding mitigation)

If the `Origin` header is present:

- It MUST match the allowlist.
- Otherwise the server returns HTTP `403`.

### Auth

- A bearer token is required by default.
- Token storage: `.dir2mcp/secret.token`.
- If `--auth file:<path>` is set, the token is loaded from that path,
  `connection.data.token_source` MUST be `file`, and `connection.data` SHOULD
  include `token_file` (or `token_source_details.path`).
- Tokens MUST NOT be embedded in URLs by default (avoid `?token=` in docs or
  outputs).

## Notes / drift

- **protocolVersion echo vs pin (dir2mcp #404):** clients MUST send
  `MCP-Protocol-Version: 2025-11-25`. The reference daemon has been observed to
  echo the client-supplied protocol version rather than asserting the pinned
  `2025-11-25` value. This spec pins the version (per
  [df-000](../data-formats/df-000-base.md)); the implementation drift is
  tracked in dir2mcp #404 and is not fixed here.

## Changelog

- **0.1.0** — Initial migration from SPEC.md §10 ("MCP server: Streamable HTTP
  (2025-11-25)"). All normative requirements preserved verbatim in substance
  (endpoint, required headers, session lifecycle, notifications, Origin checks,
  auth). The pinned `protocolVersion 2025-11-25` and `format_version` concept are
  cross-referenced to df-000; the tool surface is cross-referenced to bs-005 and
  the error model to df-008. Added a non-normative drift note for the
  protocolVersion echo-vs-pin behaviour (dir2mcp #404).
