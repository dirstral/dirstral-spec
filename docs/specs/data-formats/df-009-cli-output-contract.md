# df-009: CLI output contract

- **ID:** df-009
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §3 (migrated verbatim)

## Scope

What `dir2mcp` writes to stdout/stderr: the default human-readable output, the
`--json` NDJSON event stream, and the non-interactive missing-config error
contract. The on-disk artifacts these reference are specified in
[df-002](df-002-state-outputs.md) (state dir) and
[df-001](df-001-connection-json.md) (`connection.json`).

## Specification (normative)

### 3.1 Human output (default)

On `dir2mcp up`, stdout **MUST** print:

1. Index/state location + mode
2. The MCP connection block (URL, headers, where the token is stored)
3. Progress lines during indexing

Illustrative:

```txt
Index: /repo/.dir2mcp  (meta.sqlite + vectors_text.hnsw + vectors_code.hnsw)
Mode: incremental  (server-first; indexing in background)

MCP endpoint:
  URL:    http://127.0.0.1:52143/mcp
  Auth:   Bearer (source=file, from --auth file:/run/secrets/dir2mcp.token)
  Headers:
    MCP-Protocol-Version: 2025-11-25
    Authorization: Bearer <token>
    MCP-Session-Id: (assigned after initialize response)

Progress: scanned=412 indexed=55 skipped=340 deleted=2 reps=88 chunks=1480 embedded=920 errors=1
```

Progress-line fields (minimum): `scanned`, `indexed`, `skipped`, `deleted`,
`reps` (representations created/updated), `chunks` (chunks total known/created),
`embedded` (chunks embedded successfully), `errors` (non-fatal per-document
failures).

### 3.2 NDJSON output (`--json`)

Emit NDJSON — one JSON object per line:

```json
{
  "ts": "2026-02-25T12:34:56.789Z",
  "level": "info|warn|error",
  "event": "index_loaded|server_started|connection|scan_progress|embed_progress|file_error|payment_required|payment_verified|payment_settled|payment_failed|fatal",
  "data": {}
}
```

Required events for `up`: `index_loaded`, `server_started`, `connection`
(endpoint + headers + token reference), periodic `scan_progress` and
`embed_progress`, `file_error` for per-document (non-fatal) failures, and — if
x402 is enabled — `payment_required`, `payment_verified`, `payment_settled`,
`payment_failed`.

`connection.data` **MUST** include `transport: "mcp_streamable_http"`, `url`,
`headers` (with `MCP-Protocol-Version` and an `Authorization` placeholder), and
`token_source` (`secret.token` | `env` | `file`).

If `--auth file:<path>` is used, `token_source` **MUST** be `file`, and the
payload **SHOULD** include either `token_file` (preferred) or
`token_source_details.path` to distinguish a user-provided token file from the
auto-generated `.dir2mcp/secret.token`.

```json
{
  "transport": "mcp_streamable_http",
  "url": "http://127.0.0.1:52143/mcp",
  "headers": {
    "MCP-Protocol-Version": "2025-11-25",
    "Authorization": "Bearer <token>"
  },
  "token_source": "file",
  "token_file": "/run/secrets/dir2mcp.token"
}
```

### 3.3 Non-interactive missing-config error contract

When required config is missing and prompts are disabled (`--non-interactive` or
non-TTY), output **MUST** be actionable and **MUST NOT** print secret values:

```txt
ERROR: CONFIG_INVALID: Missing MISTRAL_API_KEY
Set env: MISTRAL_API_KEY=...
Or run: dir2mcp config init
```

The error code is a canonical [df-008](df-008-error-taxonomy.md) code.

### 3.4 Hosted demo smoke probe (operational runbook, non-normative)

A hosted-endpoint readiness check is `./scripts/smoke_hosted_demo.sh`. Set
`DIR2MCP_DEMO_TOKEN` whenever the hosted MCP endpoint enforces auth; it is
optional only for no-auth deployments (omitting it against an auth-enabled
endpoint can fail early, e.g. HTTP `401`).

```bash
DIR2MCP_DEMO_URL="https://your-host.example/mcp" \
DIR2MCP_DEMO_TOKEN="<optional-bearer-token>" \
./scripts/smoke_hosted_demo.sh
```

Expected pass conditions:

- `initialize` returns HTTP `200` and includes `MCP-Session-Id`.
- `tools/list` returns HTTP `200` and includes tool metadata.
- `tools/call` against `dir2mcp_list_files` returns either HTTP `200` with a
  JSON-RPC body, or HTTP `402` with `PAYMENT-REQUIRED` when x402 route gating is
  enabled.

## Changelog

- **0.1.0** — Migrated from SPEC.md §3. Cross-referenced the on-disk artifacts to
  df-001/df-002 and `CONFIG_INVALID` to df-008.
