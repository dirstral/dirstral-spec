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

### Challenge conformance and hardening (x402 v2 enforcement)

The `PAYMENT-REQUIRED` challenge (`HTTP 402`) is the standard x402 v2
`PaymentRequired` object: `x402Version: 2`, a first-class `resource`
(`ResourceInfo`: `url`, optional `description`/`mimeType`), an `accepts` array of
`PaymentRequirements` (`scheme`, `network`, `amount`, `asset`, `payTo`,
`maxTimeoutSeconds`, optional `extra`), and optional `extensions`. The adapter
MUST emit `resource` and `maxTimeoutSeconds` as first-class fields (never only
inside `extra`), and MUST advertise a distinct `PaymentRequirements` per paid
route rather than reuse one global requirement across routes (per-resource
binding, below).

x402 v2 already defines the primitives needed to prevent replay and cross-route
reuse — a client-signed `authorization.nonce` (a 32-byte random value, *"to
prevent replay attacks"*), a `validAfter`/`validBefore` validity window,
`maxTimeoutSeconds`, and a Parameter-Matching verification step. This section
adds **no new wire fields**; it makes those guarantees enforced by the adapter
rather than nominal.

**Single-use nonce / replay (normative).** The `nonce` carried in the client's
`PaymentPayload.payload.authorization` MUST be treated as single-use. On the
`verified -> settled` transition the adapter MUST atomically record the nonce in
a bounded, non-custodial replay ledger (see *Payment state model*) before
finalizing settlement. Replay detection MUST key off the authorization `nonce`,
**not** the raw request bytes. A later request that presents an already-recorded
nonce is classified by whether it carries the **same logical request** — same
tool, arguments, and matched `PaymentRequirements`/`resource` — that the nonce
was recorded for (payload framing is irrelevant to this comparison):

* **Same `(nonce, request)`** — an idempotent retry, **not** a replay: the
  adapter MUST re-surface the recorded outcome for that pair (see the
  reserve → commit/rollback rules below) and MUST NOT drive a second tool
  execution or settlement, and MUST NOT re-charge.
* **Same nonce, different request** — a replay/misuse attempt: the adapter MUST
  reject it via the `rejected` failure branch — even against an
  idempotent-success or sandbox facilitator that would otherwise re-approve it —
  and MUST NOT drive a second tool execution or settlement.

Consumption is a two-phase **reserve → commit/rollback** so a transient failure
never permanently burns an otherwise-valid nonce. The ledger entry recorded
before settlement is a *reservation* that blocks concurrent replays while the
settle call is in flight; it becomes **durably consumed only when settlement
succeeds**. On success the adapter MUST retain the settled outcome and
re-surface it on any idempotent retry of the same `(nonce, request)` pair
(never a re-charge). If settlement fails with a **retryable/transient** error
(network timeout, facilitator `5xx`, or any non-terminal `failed`), the adapter
MUST roll the reservation back — release the nonce — so the same
`(nonce, request)` pair MAY be retried; a transient settlement failure MUST NOT
leave the nonce consumed. A **terminal** failure (`invalid`, `rejected`,
`expired`) is a durable outcome: the payment is not valid, the nonce is not
retryable, and an idempotent retry MUST resolve to that same terminal outcome
rather than a fresh charge.

**Validity window (normative).** The adapter MUST reject a proof whose
`authorization.validAfter`/`validBefore` window does not cover the current time
(the `expired`/`invalid` failure branch), and MUST enforce the matched
`PaymentRequirements.maxTimeoutSeconds` as the maximum age between challenge and
`PAYMENT-SIGNATURE`. It MUST NOT rely on the facilitator alone for the time
check.

**Per-resource / per-tool binding (normative).** Following x402 v2's
Parameter-Matching verification step, the adapter MUST verify the client's proof
against the **entire** selected `PaymentRequirements` object (not `scheme` +
`network` alone), and MUST match the challenge `resource` to the route actually
being invoked. A `PAYMENT-SIGNATURE` proof that verifies for one resource/price
MUST NOT verify against any other tool, price, or route served by the same node.

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
* **Payment state model** – canonical states `required -> verified -> settled` with failure branches (`invalid`, `rejected`, `expired`, `failed`). dir2mcp does not persist **custodial** payment state; the facilitator is source of truth for verify/settle outcomes. It does, however, persist **non-custodial replay-protection state** — a bounded ledger of consumed authorization nonces / payment idempotency keys — because single-use enforcement is normative (above). This ledger holds no funds and no custodial balance; it records only which nonces have been spent. Because a lost entry re-opens the replay window, the ledger MUST be **durable**: a consumed nonce MUST survive process restart (crash or otherwise) for at least its validity window — the greater of the matched `PaymentRequirements.maxTimeoutSeconds` and the time to `authorization.validBefore`. An adapter that cannot persist durably (in-memory only) MUST scope its single-use guarantee explicitly to process lifetime and document that a restart within a nonce's validity window may admit one replay. After the validity window elapses an entry MAY be evicted, since the payment is then independently time-expired (`expired`) and cannot be replayed regardless.
  * **Single-use / replay semantics on `verified -> settled`.** The client's `authorization.nonce` (the 32-byte value in the `PaymentPayload`) MUST be consumed **exactly once**. On the `verified -> settled` transition the adapter MUST atomically mark that nonce consumed in the replay ledger before finalizing settlement. A subsequent request presenting an already-consumed nonce **with a different logical request** MUST be rejected via the `rejected` failure branch and MUST NOT drive a second tool execution or a second settlement — even against an idempotent-success or sandbox facilitator that would otherwise re-approve it — whereas an exact idempotent retry of the same `(nonce, request)` pair re-surfaces the recorded outcome (never a re-charge). Replay detection MUST key off the authorization `nonce` (not off the raw request bytes): a replay carrying the same nonce but a **different** request MUST be rejected rather than treated as a fresh payment. Consumption is **reserve → commit/rollback**: the pre-settlement ledger entry is a reservation that becomes durably consumed only on settlement success (its outcome retained and re-surfaced on idempotent retry of the same `(nonce, request)` pair, never a re-charge); a **retryable/transient** settlement failure MUST roll the reservation back so the pair MAY be retried, whereas a **terminal** failure (`invalid`, `rejected`, `expired`) is durable and its outcome is re-surfaced rather than re-charged.  
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

> **x402 wire profile:** `X402Version: 2`. This revision adds **no new wire
> fields**: it requires server-side *enforcement* of primitives x402 v2 already
> defines — single-use of the client's `authorization.nonce`, the
> `validAfter`/`validBefore` window, `maxTimeoutSeconds`, and full
> Parameter-Matching binding — plus `https` transport for credentialed/non-loopback
> facilitators. The profile integer is therefore unchanged and matches both the
> shipped implementation (`x402Version: 2`, the current latest x402 version) and
> the `X402Version` recorded in SPEC.md §18.

> **Note:** this document is paired with the global MCP [SPEC.md](SPEC.md). whenever the
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
