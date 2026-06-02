# Error Taxonomy

**Spec version:** `0.12.0`

Canonical error codes for all dir2mcp MCP tools and session operations.

## MCP tool errors

These are returned as JSON-RPC error objects (`{"error": {"code": ..., "message": ..., "data": {"code": ..., "retryable": ...}}}`).

<!-- spec-gap: The JSON-RPC numeric codes in the original spec (-32001 through -32006) do not match the implementation. The Go server uses -32000 for UNAUTHORIZED and -32001 for SESSION_NOT_FOUND. The implementation does not define unique numeric codes for INDEX_NOT_READY, FILE_NOT_FOUND, PERMISSION_DENIED, or RATE_LIMIT_EXCEEDED — these all use -32000 as the JSON-RPC code. The canonical code is carried in error.data.code, not in error.code. -->

### Session/auth/request errors

| Code string | JSON-RPC code | HTTP status | Retryable | Description |
|-------------|--------------|-------------|-----------|-------------|
| `UNAUTHORIZED` | -32000 | 401 | No | Missing or invalid bearer token |
| `SESSION_NOT_FOUND` | -32001 | 404 | Yes (after re-initialize) | Session ID not recognised; client should re-initialize, then retry the request |
| `RATE_LIMIT_EXCEEDED` | -32000 | 429 | Yes | Request rate exceeded; back off and retry |
| `FORBIDDEN_ORIGIN` | -32000 | 403 | No | Origin header not in the server's allowed-origins list |
| `MISSING_FIELD` | -32600 | 400 | No | Required field absent in request body |
| `INVALID_FIELD` | -32600 | 400 | No | Field present but invalid type or value |
| `METHOD_NOT_FOUND` | -32601 | 200 | No | Unknown JSON-RPC method in this transport profile |

`Retryable` indicates whether the operation may succeed after the documented client recovery step, if any. For example, `SESSION_NOT_FOUND` is retryable only after the client re-initializes the session.
### Tool execution errors (carried in toolCallResult.structuredContent.error)

These are returned with HTTP 200. The `isError: true` flag is set on the result. The error shape is `{"code": "<CODE>", "message": "...", "retryable": <bool>}` in `structuredContent.error`.

| Code string | Retryable | Description |
|-------------|-----------|-------------|
| `INDEX_NOT_READY` | Yes | Index is still building or retriever not configured |
| `FILE_NOT_FOUND` | No | Requested file path does not exist in the corpus |
| `OCR_NOT_READY` | Yes | `dir2mcp_open_file` on a binary doc type (PDF, audio) before its OCR/transcript representation is cached; retry once ingestion completes |
| `PERMISSION_DENIED` | No | Path is outside the root or excluded by policy |
| `MISSING_FIELD` | No | Required tool argument absent |
| `INVALID_FIELD` | No | Tool argument present but invalid type or value |
| `INVALID_RANGE` | No | Numeric argument outside allowed bounds |
| `STORE_CORRUPT` | No | Store read failed with unexpected error |
| `INTERNAL_ERROR` | Yes | Unexpected server-side error |

## x402 payment errors

Returned when x402 mode is `on` or `required`. See `spec/x402/extension.md`.

| Code string | Description |
|-------------|-------------|
| `PAYMENT_REQUIRED` | No payment proof provided; 402 response with `PAYMENT-REQUIRED` header |
| `PAYMENT_INVALID` | Payment proof provided but verification failed |
| `PAYMENT_FACILITATOR_UNAVAILABLE` | Facilitator unreachable; fail-open (`on`) or fail-closed (`required`) |
| `PAYMENT_SETTLEMENT_FAILED` | Settlement call to facilitator failed |
| `PAYMENT_SETTLEMENT_UNAVAILABLE` | Settlement endpoint unreachable |
| `PAYMENT_CONFIG_INVALID` | x402 config is incomplete or invalid |

## Error shape contract

### Session/auth/request errors (transport semantics)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "missing or invalid bearer token",
    "data": {
      "code": "UNAUTHORIZED",
      "retryable": false
    }
  }
}
```

The `error.data.code` field carries the canonical string error code. The `error.code` is a standard JSON-RPC numeric code. The `error.message` field is human-readable. Status is usually 4xx for session/auth/request validation failures, except `METHOD_NOT_FOUND`, which is returned as HTTP 200 in this implementation profile.

### Tool execution errors (HTTP 200 with isError: true)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "isError": true,
    "content": [{"type": "text", "text": "ERROR: INDEX_NOT_READY: retriever not configured"}],
    "structuredContent": {
      "error": {
        "code": "INDEX_NOT_READY",
        "message": "retriever not configured",
        "retryable": true
      }
    }
  }
}
```

Tool execution errors are always returned with HTTP 200. The `content[0].text` field prefixes the error with `ERROR: <code>: `.
