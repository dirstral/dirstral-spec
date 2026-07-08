# bs-010: Native x402 integration

- **ID:** bs-010
- **Version:** 0.2.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §18 (+ canonical adapter spec: ../x402-payment-adapter-spec.md)

## Scope

The **minimum** normative requirements for native x402 request gating — an
HTTP-402-based payment challenge system layered on the MCP request boundary.
This document captures the integration contract: the operator-facing mode
switch, where enforcement happens, the challenge/retry handshake, the
non-custodial verify/settle delegation, error mapping, telemetry, and optional
Bazaar discovery. It does **not** restate the wire-level detail (header
encodings, payload shapes, facilitator API contracts, fail-open vs. strict
edge cases) — that canonical detail lives in the
[x402 payment adapter spec](../../x402-payment-adapter-spec.md), which this document
points to.

## Specification (normative)

### Mode switch

- x402 mode is **optional** and **MUST** be switchable via config/flags, with
  three values: `off | on | required`. The mode semantics are:
  - `off` — x402 is **disabled**; no payment gating.
  - `on` — enabled, **fail-open** on incomplete configuration (gating degrades
    gracefully rather than blocking when config is partial).
  - `required` — enabled with **strict** config validation/gating (incomplete
    configuration is an error, and paid routes are strictly gated).

### Enforcement boundary

- Payment enforcement **MUST** happen at the HTTP/MCP **request boundary**, not
  in retrieval/indexing internals.
- Recommended paid scope: gate `tools/call` (or selected tool names); the
  lifecycle methods (`initialize`, `tools/list`) **SHOULD** remain ungated.

### Challenge / retry handshake

- When a paid route is called **without** valid payment, the server returns HTTP
  `402 Payment Required` with machine-readable payment requirements carried in
  the `PAYMENT-REQUIRED` field.
- The adapter **MUST** enforce x402 v2's replay and binding primitives (no new wire fields): the client's single-use `authorization.nonce` **MUST** be consumed exactly once via a bounded replay ledger; the `authorization.validAfter`/`validBefore` window and the matched `PaymentRequirements.maxTimeoutSeconds` **MUST** be checked adapter-side; and the proof **MUST** match the **entire** selected `PaymentRequirements` and the challenge `resource` — so a proof valid for one resource/price/tool **MUST NOT** be valid for another. Enforcement detail lives in the [x402 payment adapter spec](../../x402-payment-adapter-spec.md).
- Paid **retry** requests **MUST** be validated from `PAYMENT-SIGNATURE`
  (x402 v2 semantics, wire profile `X402Version: 2`).
- x402 **network identifiers** **MUST** use CAIP-2 format (for example:
  `eip155:8453`, `eip155:84532`,
  `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d`).

### Verification and settlement (non-custodial)

- For paid requests, **verification and settlement** **MUST** be delegated to a
  **facilitator** (hosted or self-managed); dir2mcp remains **non-custodial**.
- The adapter→facilitator transport **MUST** be `https` whenever it is credentialed or the facilitator host is non-loopback; a bearer token **MUST NOT** traverse plaintext `http` to a non-loopback host (all modes, including `on`). See the [adapter spec](../../x402-payment-adapter-spec.md).
- Remaining **non-custodial** does **not** preclude single-use enforcement: dir2mcp **MAY** persist a bounded, non-custodial replay ledger of consumed nonces / idempotency keys. On the `verified -> settled` transition a nonce **MUST** be consumed exactly once; a replay of a consumed nonce — or the same nonce with a different request — **MUST** be rejected and **MUST NOT** drive a second execution or settlement. Replay detection keys off the payment nonce, not raw request bytes.
- Successful paid responses **SHOULD** include facilitator settlement metadata
  via `PAYMENT-RESPONSE` when available.

> Drift note (dir2mcp #400): the facilitator verify/settle path is specified to
> carry the verdict **in the response value**, not as a transport error. An
> implementation that treats a "not verified" verdict as a thrown error (rather
> than a value to branch on) can bypass gating. The canonical handling is in the
> [adapter spec](../../x402-payment-adapter-spec.md); this is a one-line pointer,
> not a fix.

### Error mapping

- Payment failures **MUST** map to the canonical tool/transport errors
  ([df-008](../data-formats/df-008-error-taxonomy.md)) — `UNAUTHORIZED`,
  `MISTRAL_FAILED` — plus x402-specific payment-failure metadata.

### Telemetry

- If enabled, the server **SHOULD** emit payment telemetry in NDJSON
  ([bs-009](bs-009-security-safety.md)) with event types
  `payment_required | payment_verified | payment_settled | payment_failed`.

### Bazaar / discovery (optional, additive)

- Bazaar/discovery metadata is **optional and additive**; lack of Bazaar
  metadata **MUST NOT** affect core MCP behavior ([bs-004](bs-004-mcp-transport.md)).
- If Bazaar support is enabled, discovery metadata **SHOULD** be emitted via
  x402 extension metadata and resolved through facilitator discovery APIs (for
  example, `GET {facilitator_url}/discovery/resources`).

## Changelog

- **0.2.0** — x402 flow hardening (syncs canonical adapter spec; unblocks dir2mcp #421, follows #400). Enforces x402 v2's existing primitives (no new wire fields): single-use of the client `authorization.nonce`, the `validAfter`/`validBefore` window + `PaymentRequirements.maxTimeoutSeconds`, and full-`PaymentRequirements` + `resource` binding so a proof for one route isn't valid for another. Verification/settlement: require `https` transport for credentialed/non-loopback facilitators; permit a bounded non-custodial replay ledger with exactly-once nonce consumption on `verified -> settled`. Wire profile stays `X402Version: 2` (current latest). Enforcement detail in the x402 payment adapter spec.
- **0.1.0** — Migrated from SPEC.md §18 (native x402 integration requirements,
  minimum). Every normative requirement preserved; prose tightened and grouped
  by concern, with the `off|on|required` mode semantics made explicit. The
  canonical wire-level detail remains in the
  [x402 payment adapter spec](../../x402-payment-adapter-spec.md), which this
  document points to rather than duplicating. Cross-references rewired to stable
  doc IDs: error taxonomy → [df-008](../data-formats/df-008-error-taxonomy.md);
  MCP tools/lifecycle → [bs-004](bs-004-mcp-transport.md); NDJSON telemetry →
  [bs-009](bs-009-security-safety.md). Added a one-line drift note for the
  verify/settle verdict-in-value handling (dir2mcp #400).
