# Error Taxonomy

**Spec version:** `0.4.0`

Canonical error codes for all dir2mcp MCP tools and session operations.

## MCP tool errors

These are returned as JSON-RPC error objects (`{"error": {"code": ..., "message": ...}}`).

| Code string | JSON-RPC code | HTTP status | Retryable | Description |
|-------------|--------------|-------------|-----------|-------------|
| `UNAUTHORIZED` | -32001 | 401 | No | Missing or invalid bearer token |
| `SESSION_NOT_FOUND` | -32002 | 404 | Yes (re-init) | Session ID not recognised; client should re-initialize |
| `INDEX_NOT_READY` | -32003 | 503 | Yes | Index is still building; retry after a short delay |
| `FILE_NOT_FOUND` | -32004 | 404 | No | Requested file path does not exist in the corpus |
| `PERMISSION_DENIED` | -32005 | 403 | No | Path is outside the root or excluded by policy |
| `RATE_LIMIT_EXCEEDED` | -32006 | 429 | Yes | Request rate exceeded; back off and retry |

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

All errors MUST follow this JSON-RPC error envelope:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32001,
    "message": "UNAUTHORIZED: missing or invalid bearer token"
  }
}
```

The `message` field SHOULD be human-readable and MAY include the code string as a prefix.
