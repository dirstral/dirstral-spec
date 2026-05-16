# Design 0001 — Multi-provider model abstraction

**Status:** Proposed (targets spec `0.7.0`)
**Author:** dirstral maintainers
**Supersedes:** the implicit "Mistral is the model provider" assumption in SPEC §1, §8.1
**Related:** SPEC §8 (model/provider utilization), §16 (configuration); [versioning.md](../../spec/versioning.md)

## 1. Summary

Generalize dir2mcp from a Mistral-centric pipeline to a **provider-agnostic** one. Every model capability — embeddings, chat/RAG generation, OCR, STT, rerank — becomes a *binding* to a named **provider profile**. A single **OpenAI-compatible adapter** is the backbone for chat + embeddings across OpenAI, OpenRouter, Groq, Together, Fireworks, DeepInfra, Azure-style and local (Ollama/vLLM/LM Studio) endpoints — **and for Mistral itself**. Bespoke adapters remain only where the wire protocol is genuinely *not* OpenAI-shaped.

This is a **clean break**: dir2mcp is pre-institutional beta with no compatibility users, so the monolithic `mistral:` config block is removed rather than shimmed.

## 2. Motivation

- Today Mistral is hard-wired as the provider for embed/chat/OCR/STT. Users want OpenAI, Anthropic, Gemini, OpenRouter, and local/self-hosted models — for cost, quality, data-residency, and offline use.
- The abstraction cost is **low**: `internal/model` already defines narrow, provider-agnostic interfaces (`Embedder`, `Generator`, `OCR`, `Transcriber`, `Reranker`), and a per-capability provider-selector pattern already exists for `stt.provider` and `rerank.provider`. This design *generalizes existing patterns*, it is not a rearchitecture.

## 3. Key insight — Mistral is already (mostly) OpenAI-compatible

`internal/mistral/client.go` already calls OpenAI-shaped endpoints on `api.mistral.ai`:

| Capability | Mistral endpoint | OpenAI-compatible? |
|---|---|---|
| Chat | `/v1/chat/completions` | **Yes** |
| Embeddings | `/v1/embeddings` | **Yes** |
| OCR | `/v1/ocr` | **No** — proprietary, no OpenAI analog |
| STT | `/v1/audio/transcriptions` | OpenAI-audio-shaped, Voxtral specifics |

Therefore the OpenAI-compatible adapter is not just "OpenAI + OpenRouter + locals" — it is the **embed/chat backbone for every provider, Mistral included**. Mistral's only genuinely bespoke surface is `/v1/ocr`.

> Caveat: OpenAI chat compatibility is ~90%, not 100% (tool-call JSON nuances, assistant prefill, FIM, embedding `dimensions`/`encoding_format`). The adapter targets the common path; provider-specific knobs live in the profile.

## 4. Capability matrix (normative source for §8)

| Provider `kind` | embed | chat | ocr | stt | tts | rerank |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| `openai` (OpenAI-compatible: OpenAI, OpenRouter, Groq, Together, Azure, Ollama, vLLM, …) | ✅ | ✅ | ❌ | ✅¹ | ✅¹ | ❌ |
| `mistral` (native OCR + Voxtral STT) | — | — | ✅ | ✅ | – | – |
| `anthropic` (Messages API) | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `gemini` (native; also has an OpenAI-compat endpoint) | ✅ | ✅ | ❌ | ✅ | – | ❌ |
| `cohere` | ✅ | ✅ | – | – | – | ✅ |
| `elevenlabs` | – | – | – | ✅ | ✅ | – |

¹ Depends on the concrete endpoint (e.g. OpenAI Whisper/TTS; a bare OpenRouter gateway exposes chat only).

Selection and preflight **MUST** validate against this matrix: binding a capability to a `kind` that cannot serve it is `CONFIG_INVALID`.

## 5. Architecture

### 5.1 Provider profiles

A `providers:` map of named profiles. Each profile:

- `kind`: the adapter/wire protocol — `openai` | `mistral` | `anthropic` | `gemini` | `cohere` | `elevenlabs`
- `base_url`: endpoint (defaulted per kind; overridable for OpenRouter/Azure/local)
- `api_key`: secret reference (env/keychain/file per §16.1.1) — **never persisted**
- per-capability default model names

Built-in profiles ship for the common providers so users typically only supply a credential. `mistral` is a built-in profile with `kind: openai` (chat+embed via `api.mistral.ai`) **plus** a `kind: mistral` capability for `/v1/ocr`.

### 5.2 Per-capability bindings

Each capability binds to a profile + model:

```
model.embed.provider  / model.embed.text_model / model.embed.code_model
model.chat.provider   / model.chat.model
model.ocr.provider    / model.ocr.model
stt.provider          / stt.<profile>.model      (existing shape, retained)
rerank.provider       / rerank.<profile>.model   (existing shape, retained)
```

### 5.3 Selection semantics (reconciles capability-driven activation with multiple providers)

For each capability:

1. If its `*.provider` is set → use it (validated against §4). If that profile lacks a credential and the capability is required → `CONFIG_INVALID` with remediation.
2. If unset (**auto**) → pick the **first profile, by a fixed precedence, that (a) has a credential present and (b) can serve the capability** per §4.
3. If none qualify → the capability is unavailable: a *required* capability (embed) fails preflight; an *optional* one (rerank) stays off silently (the existing capability-driven rule, generalized).

This preserves "credentials are the opt-in" while making selection deterministic when several providers are credentialed.

### 5.4 Adapter taxonomy

| Adapter | Scope | Status |
|---|---|---|
| `openai` (new) | embed + chat for OpenAI/OpenRouter/Groq/Together/Azure/local **and Mistral** | new, the backbone |
| `mistral` (existing, shrinks) | **only** `/v1/ocr` (+ Voxtral STT) | retained, reduced |
| `anthropic` (new) | chat only (Messages API) | new |
| `gemini` (new) | embed + chat (native, or via its OpenAI-compat endpoint as a `kind: openai` profile) | new |
| `cohere` (existing) | rerank (+ optionally embed later) | unchanged |
| `elevenlabs` (existing) | STT/TTS | unchanged |

### 5.5 The "full modification" — re-express Mistral now

`internal/mistral` is reduced to the OCR (and Voxtral STT) client. Mistral chat + embeddings flow through the generic `openai` adapter via the built-in `mistral` profile (`base_url: https://api.mistral.ai/v1`, default models `mistral-small-*`, `mistral-embed`, `codestral-embed`). One code path for chat/embed across *all* providers; bespoke code only where the protocol truly differs.

## 6. Embeddings are a corpus-lifetime invariant (normative)

Vectors from different embed models/providers are **not comparable**. Therefore:

- The embed provider + model identity is bound to the index at first build.
- The config snapshot records embed identity.
- On load, if the configured embed identity differs from the index's, the server **MUST** either refuse to serve stale results (`CONFIG_INVALID` / `STORE_CORRUPT`-class) or trigger a full reindex — it MUST NOT silently mix vector spaces.
- `embed.provider`/`embed.text_model`/`embed.code_model` are effectively **deploy-time, reindex-bound** choices, not runtime toggles. Chat/OCR/STT/rerank providers *are* runtime-swappable.

## 7. Config schema (clean break)

**Before (0.6.0):**

```yaml
mistral:
  api_key: ${MISTRAL_API_KEY}
  chat_model: mistral-small-2506
  embed_text_model: mistral-embed
  embed_code_model: codestral-embed
  ocr_model: mistral-ocr-latest
```

**After (0.7.0):**

```yaml
providers:
  mistral:                       # built-in; shown for clarity
    kind: openai                 # chat+embed are OpenAI-shaped on api.mistral.ai
    base_url: https://api.mistral.ai/v1
    api_key: ${MISTRAL_API_KEY}
  mistral-ocr:                   # native /v1/ocr
    kind: mistral
    api_key: ${MISTRAL_API_KEY}
  openai:
    kind: openai
    api_key: ${OPENAI_API_KEY}
  openrouter:
    kind: openai
    base_url: https://openrouter.ai/api/v1
    api_key: ${OPENROUTER_API_KEY}
  anthropic:
    kind: anthropic
    api_key: ${ANTHROPIC_API_KEY}
  local:
    kind: openai
    base_url: http://localhost:11434/v1   # Ollama/vLLM/LM Studio

model:
  embed:
    provider: mistral            # unset => auto by precedence among credentialed+capable
    text_model: mistral-embed
    code_model: codestral-embed
  chat:
    provider: anthropic          # e.g. answer with Claude
    model: claude-sonnet-4-6
  ocr:
    provider: mistral-ocr
    model: mistral-ocr-latest

stt:    { provider: mistral, ... }      # existing shape
rerank: { provider: cohere, ... }       # existing shape
```

## 8. Wire / contract impact: none

No new or changed MCP tool, tool schema, error envelope, or session behavior. Provider selection is **server-internal configuration**. Consequences:

- `spec/tools/schemas/*` and `spec/errors/taxonomy.md`: unchanged (one new *config-validation* condition reuses existing `CONFIG_INVALID`).
- `dirstral-conformance`: no new tests required (wire contract unchanged).
- `dirstral-cli`: no client code change; version-matrix bump only.

Same blast radius as the 0.6.0 rerank change.

## 9. Versioning

Spec `0.6.0 → 0.7.0`. Pre-1.0 beta policy: a config-shape break **and** new optional surface both bump `MINOR`. Clean break is acceptable (no compatibility users).

## 10. Phasing

1. **Backbone**: `openai` adapter (embed+chat) + `providers:`/`model.*` config + generalized selection + Mistral re-expressed (OCR retained). Covers OpenAI, OpenRouter, Groq, local, Mistral.
2. **Anthropic** (chat) + **Gemini** (embed+chat).
3. **Voyage** (embed + rerank) as an additional `kind` if desired.

## 11. Non-goals

- Backward-compatible dual config (no users).
- Per-request provider override via MCP tool args (server config only).
- Embeddings hot-swap without reindex.
- AWS Bedrock / Vertex IAM integration (future, if requested).
