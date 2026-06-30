# bs-010: Native x402 integration

- **ID:** bs-010
- **Version:** 0.1.0
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
- Paid **retry** requests **MUST** be validated from `PAYMENT-SIGNATURE`
  (x402 v2 semantics).
- x402 **network identifiers** **MUST** use CAIP-2 format (for example:
  `eip155:8453`, `eip155:84532`,
  `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d`).

### Verification and settlement (non-custodial)

- For paid requests, **verification and settlement** **MUST** be delegated to a
  **facilitator** (hosted or self-managed); dir2mcp remains **non-custodial**.
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
