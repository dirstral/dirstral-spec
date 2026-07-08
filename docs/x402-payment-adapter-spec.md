# x402 Payment Adapter Specification

This document defines the **x402 payment adapter** contract referenced from `VISION.md`. The adapter sits between a dir2mcp node and a third-party facilitator to enable optional HTTP 402 gating on selected MCP routes while keeping retrieval internals payment-agnostic.

> **x402 request gating** is a lightweight HTTP standard (originating from Coinbase's x402 project) that lets a server require clients to present payment proofs before processing certain requests. dir2mcp uses this mechanism to optionally throttle or monetize access to sensitive MCP endpoints without hard‑coding any particular payment network.

## Purpose

Provide a clear, versioned contract so that:

* dir2mcp can emit and consume payment requirements without embedding blockchain logic.
* Facilitator providers can be swapped via configuration.
* Discovery and billing layers can interoperate with any compliant adapter.

## Usage in dir2mcp

When x402 gating is enabled (`--x402` flag in the CLI or `x402.mode` in config), dir2mcp invokes the adapter at the MCP request boundary (typically `POST /mcp` for `tools/call`, or selected tool names). Depending on the facilitator's response, the server either allows the request to proceed or returns `402 Payment Required` with `PAYMENT-REQUIRED` header data. Clients must then obtain and attach `PAYMENT-SIGNATURE` before retrying.

The facilitator integration currently defaults to the Coinbase x402 Go SDK client.
### Example configuration snippet

```yaml
# config.yaml
x402:
  mode: required            # off, on, required
  adapter: "coinbase"      # named adapter from internal/x402
  facilitator:
    api_key: "..."         # credentials for the external service

```

> **Security note:** never commit API keys or sensitive credentials (such as
> `x402.facilitator.api_key`) to version control.
> Store them securely using environment variables, a secret manager (e.g.
> HashiCorp Vault, AWS Secrets Manager), or encrypted configuration files.
## Normative baseline

The adapter MUST align with x402 v2 concepts and headers, at wire profile
`X402Version: 2` (see version note at the foot of this document):

* `PAYMENT-REQUIRED` for payment challenges (`HTTP 402`)
* `PAYMENT-SIGNATURE` for client payment proof
* `PAYMENT-RESPONSE` for settlement/receipt metadata

### PAYMENT-REQUIRED challenge schema

Every `PAYMENT-REQUIRED` challenge (`HTTP 402`) MUST carry, in addition to the
existing `scheme`, `network`, amount/`asset`, and `payTo` fields:

* `X402Version` — integer wire-profile version of the challenge (currently `2`).
* `resource` — the fully-qualified paid resource identifier. It MUST be a
  first-class, canonically-placed field of the challenge; it MUST NOT be buried
  only inside an opaque `extra` blob.
* `nonce` — a single-use, server-issued challenge token. The nonce is
  **stateless**: it is an HMAC token (keyed by a server secret) that binds the
  challenge to `(resource, cost, network, validAfter, validUntil)`. The adapter
  MUST be able to recompute and verify the nonce without shared mutable state,
  and additionally enforce single-use via the replay ledger (see *Payment state
  model*).
* `validAfter` / `validUntil` — the inclusive lower and exclusive upper bounds
  of the challenge's validity window (RFC 3339 timestamps or Unix seconds,
  consistently applied). A proof presented outside the window MUST be rejected
  with the `expired` failure branch.
* `maxTimeoutSeconds` — the maximum age, in seconds, the adapter will accept
  between challenge issuance and a matching `PAYMENT-SIGNATURE`.

**Per-resource / per-tool binding (normative).** A challenge — and therefore any
proof derived from it — MUST be bound to exactly one `(resource, cost)` pair via
the nonce. A `PAYMENT-SIGNATURE` proof that verifies against the challenge for
one tool/price MUST NOT verify against any other tool or price, even when both
are served by the same node. Implementations MUST NOT reuse a single global
requirement value across distinct paid routes.

Reference materials:

* x402 spec repository: <https://github.com/coinbase/x402/tree/main/specs>
* Facilitator API reference: <https://docs.cdp.coinbase.com/api-reference/v2/rest-api/x402-facilitator/x402-facilitator>
* Core concepts and migration notes:
	* <https://docs.cdp.coinbase.com/x402/core-concepts/how-it-works>
	* <https://docs.cdp.coinbase.com/x402/core-concepts/http-402>
	* <https://docs.cdp.coinbase.com/x402/migration-guide>

## Adapter contract

The contract defines, at a minimum, the following elements:  

* **Facilitator operations** – call facilitator verify and settle endpoints and map their responses into dir2mcp transport behavior. Adapters must implement verify (to validate payment proofs) and settle (to finalize payments) operations according to their facilitator's API specification.
* **Authentication** – adapter-to-facilitator auth must be explicit (for example API key auth for hosted facilitator, mTLS or signed requests for self-managed deployments).  
* **Transport security** – the adapter→facilitator transport MUST be `https` whenever the connection is **credentialed** (any bearer token, API key, or signed credential is attached) **or** the facilitator host is **non-loopback**. A bearer token or payment payload MUST NEVER traverse plaintext `http` to a non-loopback host. Plaintext `http` is permitted ONLY for a loopback (`127.0.0.0/8`, `::1`) host with no credential attached (local development). This requirement holds in **all** modes, including `on` (fail-open) — a non-https credentialed/non-loopback facilitator URL is a configuration error, not a degradable condition.  
* **Payment state model** – canonical states `required -> verified -> settled` with failure branches (`invalid`, `rejected`, `expired`, `failed`). dir2mcp does not persist **custodial** payment state; the facilitator is source of truth for verify/settle outcomes. dir2mcp MAY, however, persist **non-custodial replay-protection state** — a bounded ledger of consumed challenge nonces / payment idempotency keys — for single-use enforcement. This ledger holds no funds and no custodial balance; it records only which nonces have been spent, with a TTL at least as long as `maxTimeoutSeconds` and SHOULD survive process restart.
  * **Single-use / replay semantics on `verified -> settled`.** A challenge nonce MUST be consumed **exactly once**. On the `verified -> settled` transition the adapter MUST atomically mark the nonce consumed in the replay ledger before finalizing settlement. A subsequent request presenting an already-consumed nonce MUST be rejected via the `rejected` failure branch and MUST NOT drive a second tool execution or a second settlement — even against an idempotent-success or sandbox facilitator that would otherwise re-approve it. Replay detection MUST key off the payment nonce (not off the raw request bytes): a replay carrying the same nonce but a **different** request payload MUST be rejected rather than treated as a fresh payment. Idempotent retry of the *same* `(nonce, request)` pair MUST resolve to the original outcome, not a re-charge.  
* **Error codes and retries** – standard HTTP handling (`402`, `4xx`, `5xx`), idempotent settle calls, bounded retry/backoff for transient failures, and explicit non-retryable classes for invalid signatures/requirements mismatch.  
* **Network normalization** – adapters must use CAIP-2 network identifiers at all payment-related boundaries.  Examples include:
  * `eip155:8453`
  * `eip155:84532`
  * Solana mainnet: `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d`
  * Solana devnet: `solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1wcaWoxPkrZBG`
  * Solana testnet: `solana:4uhcVJyU9pJkvQyS88uRDiswHXSCkY3zQawwpjk2NsNY`.  
* **Discovery passthrough (optional)** – if Bazaar is enabled, expose extension metadata to facilitator discovery ingestion rather than implementing a custom discovery protocol in dir2mcp.  

## Out of scope

This adapter does not define:

* wallet key management,
* custodial fund handling,
* marketplace ranking/reputation logic,
* non-x402 billing schemes.

---

> **x402 wire profile:** `X402Version: 2`. This revision extends the
> `PAYMENT-REQUIRED` challenge shape additively (adds `nonce`,
> `validAfter`/`validUntil`, `maxTimeoutSeconds`, and promotes `resource` to a
> first-class field) — a `X402Version: 2` client still parses the `402`; the new
> fields are additive and the hardening is server-side enforcement, so the
> profile integer is unchanged and matches the shipped implementation
> (`x402Version: 2`) and the `X402Version` recorded in SPEC.md §18.

> **Note:** this document is paired with the global MCP [SPEC.md](../SPEC.md). whenever the
> protocol version, message formats, field definitions, or error codes evolve you must keep
> both documents in sync. reviewers should use the following checklist when making changes:
>
> 1. bump the protocol version in **both** specs
> 2. reconcile any example request/response messages and payload schemas
> 3. update compatibility/upgrade notes or migration guidance in both places
> 4. verify field definitions (names, types, required/optional semantics) match
> 5. refresh error code tables and retry logic descriptions
>
> Failing to update either doc can lead to incompatible implementations, so include a
> comment in your PR pointing to the related edits.
