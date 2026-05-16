# x402 Payment Extension

**Spec version:** `0.7.0`
**Status:** Optional extension — not required for core MCP conformance.

## Overview

x402 is an HTTP-native payment challenge protocol. `dir2mcp` supports optional x402 request gating on `tools/call` endpoints. This document specifies the wire behavior.

## Modes

| Mode | Value | Behavior |
|------|-------|----------|
| Disabled | `off` | No payment gating. All requests pass through. |
| Enabled | `on` | Payment gating active. Fail-open if facilitator is unreachable. |
| Required | `required` | Payment gating active. Fail-closed — reject requests if facilitator is unreachable. Config validation enforced at startup. |

## Headers

| Header | Direction | Description |
|--------|-----------|-------------|
| `PAYMENT-REQUIRED` | Server → Client | JSON challenge payload on 402 response |
| `PAYMENT-SIGNATURE` | Client → Server | Client payment proof on retry |
| `PAYMENT-RESPONSE` | Server → Client | Settlement receipt on successful paid call |

## Payment flow

```
Client                              Server            Facilitator
  |                                   |                    |
  |  POST /mcp (tools/call)           |                    |
  |  ─────────────────────────────>   |                    |
  |                                   |                    |
  |  402 Payment Required             |                    |
  |  PAYMENT-REQUIRED: <challenge>    |                    |
  |  <─────────────────────────────   |                    |
  |                                   |                    |
  |  POST /mcp (tools/call)           |                    |
  |  PAYMENT-SIGNATURE: <proof>       |                    |
  |  ─────────────────────────────>   |                    |
  |                                   |  POST /v2/verify   |
  |                                   |  ─────────────────>|
  |                                   |  200 OK            |
  |                                   |  <─────────────────|
  |                                   |  POST /v2/settle   |
  |                                   |  ─────────────────>|
  |                                   |  200 OK            |
  |                                   |  <─────────────────|
  |  200 OK                           |                    |
  |  PAYMENT-RESPONSE: <receipt>      |                    |
  |  {"result": {...}}                |                    |
  |  <─────────────────────────────   |                    |
```

## Challenge payload (`PAYMENT-REQUIRED` header value)

The JSON value is serialized to a string and placed in the `PAYMENT-REQUIRED` response header.

Supported schemes: `exact` (fixed amount) and `upto` (amount up to a maximum).

```json
{
  "x402Version": 2,
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "1000000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xYourWalletAddress",
      "resource": "https://your-endpoint/mcp"
    }
  ]
}
```

For the `upto` scheme, `maxAmountRequired` is also present in the accept entry. Both `amount` and `maxAmountRequired` are represented as decimal integer strings (not JSON numbers) to avoid floating-point precision issues. `maxAmountRequired` MUST parse as a positive integer >= `amount`:

```json
{
  "x402Version": 2,
  "accepts": [
    {
      "scheme": "upto",
      "network": "eip155:8453",
      "amount": "500000",
      "maxAmountRequired": "1000000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xYourWalletAddress",
      "resource": "https://your-endpoint/mcp"
    }
  ]
}
```

## Fail-open vs fail-closed

| Mode | Facilitator reachable | Behavior |
|------|----------------------|---------|
| `on` | Yes | Normal payment flow |
| `on` | No | Fail-open: serve request without payment verification |
| `required` | Yes | Normal payment flow |
| `required` | No | Fail-closed: return `PAYMENT_FACILITATOR_UNAVAILABLE` error |

## Config validation at startup

- `off`: no validation required
- `on`: warn if config is incomplete, but start
- `required`: reject startup if facilitator URL, wallet address, or asset config is missing
