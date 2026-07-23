# td-001: Provider model & capability activation

- **ID:** td-001
- **Version:** 0.5.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §8.1–§8.5 (excl. §8.1.7 → td-002), §8.8

## Scope

dir2mcp's **provider-agnostic** model: how each model capability
(`embed`, `chat`, `ocr`, `stt`, `tts`, `rerank`) binds to a named **provider
profile**, how a profile maps to an adapter / wire protocol, how providers are
selected (set vs. auto), and how optional providers **activate automatically on
credential presence**. It also covers the OpenAI-compatible **backbone** vs. the
handful of **bespoke native** adapters, the corpus-lifetime embed identity,
asymmetric (input-role) and Matryoshka (configurable-dimension) embeddings,
STT / TTS / rerank provider selection (each an **open** set, not a closed enum),
first-class self-hosted OpenAI-compatible endpoints, and the **detected-language
resolution** rules for representation language.

Two principles govern the whole document: dir2mcp is **best-config-by-default
AND fully provider-swappable**. Mistral is the **default** profile but is **not**
privileged — no provider is — and every runtime capability may be reconfigured
to any provider that can serve it. Multimodal embeddings and media chunking are
split out to **td-002**; audio/video transcription, translation, and subtitles
to **td-003**; structured document extraction to **td-004**.

## Specification (normative)

### 8.1 Provider model (provider-agnostic)

dir2mcp is **provider-agnostic**. Each model capability — `embed`, `chat`,
`ocr`, `stt`, `tts`, `rerank` — binds to a named **provider profile**. Mistral
is the default profile but is **not** privileged. Rationale and full design:
[Design 0001](../../design/0001-multi-provider.md).

#### 8.1.1 Provider profiles

A profile declares a `kind` (the adapter / wire protocol), a `base_url`
(defaulted per kind; overridable), an **optional** `api_key` secret reference
(resolved per bs-011 §16.1.1, never persisted), and per-capability default model
names. A profile with no `api_key` is **credential-less** (e.g. a local
Ollama/vLLM/LM Studio endpoint that requires no key); credential-less profiles
are first-class and count as **eligible** for selection and preflight (8.1.3).
Defined `kind`s:

* `openai` — the OpenAI-compatible **backbone**: OpenAI, OpenRouter, Groq,
  Together, Azure-style, and local Ollama/vLLM/LM Studio — **and Mistral
  chat/embeddings** (`api.mistral.ai` already serves `/v1/chat/completions` and
  `/v1/embeddings`). Endpoints that expose audio also serve STT
  (`/v1/audio/transcriptions`, Whisper / `gpt-4o-transcribe`) and TTS
  (`/v1/audio/speech`) — endpoint-dependent, see 8.1.2.
* `mistral` — native `/v1/ocr` (and Voxtral STT); the only genuinely non-OpenAI
  Mistral surface.
* `anthropic` — Messages API (chat only).
* `gemini` — native embed (**asymmetric** via `taskType`, with Matryoshka output
  dimensionality — see 8.1.5/8.1.6), chat, STT (audio transcription), and TTS.
  The native embed surface (`models/{model}:batchEmbedContents`) is required for
  `taskType`/`outputDimensionality`; STT and TTS likewise use the native
  `models/{model}:generateContent` surface (see 8.2/8.3) — Gemini's
  OpenAI-compatible layer does **not** expose `/v1/audio/*`, so only chat may
  ride the `kind: openai` path. A `gemini` profile MAY alternatively be
  configured as a `kind: openai` profile via Gemini's OpenAI-compatible
  endpoint, which serves chat only and forgoes `taskType` (and thus the
  asymmetric/role behavior).
* `cohere` — embed, chat, and rerank (8.4). Cohere embeddings are **asymmetric**
  (see 8.1.5).
* `elevenlabs` — STT/TTS.

Built-in profiles ship for common providers so operators typically only supply a
credential.

#### 8.1.2 Capability matrix (normative)

| `kind` | embed | chat | ocr | stt | tts | rerank |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| `openai` | ✅ | ✅ | ❌ | ✅³ | ✅³ | ❌ |
| `mistral` | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| `anthropic` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `gemini` | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| `cohere` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `elevenlabs` | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |

Binding a capability to a `kind` whose cell is `❌` MUST be rejected as
`CONFIG_INVALID` (static validation). ³ = `kind: openai` audio (STT/TTS) is
**endpoint-dependent** and cannot be statically validated (an arbitrary
OpenAI-compatible `base_url` may omit `/v1/audio/*`). The adapter implements it;
if the configured endpoint lacks it, the failure surfaces **at first use** as a
provider error — a required STT path fails that ingest item, optional TTS fails
open (8.3) — never as `CONFIG_INVALID`. All other `✅` cells are statically
valid.

**Extraction is not a cell in this matrix.** Document/image *extraction-engine*
selection (docling / docling-serve / mistral-ocr / pandoc) is a per-format,
fidelity-ordered routing decision owned by
[td-004](td-004-representation-extraction.md) §B.1, not a `kind`-level capability
here: extraction fidelity is per-format and two engines (`docling`, `pandoc`)
have no §8.1.1 provider profile. Where an extraction engine *is* an §8 surface,
it binds through the corresponding capability — the `mistral` extraction engine
is the active `ocr` provider (selected per 8.1.3), and the audio extraction path
binds `stt` ([td-004](td-004-representation-extraction.md) §C). No
`extract`/`CapExtract` capability cell is added.

#### 8.1.3 Provider selection

For each capability, with `<cap>.provider`:

1. **Set** → use that profile, validated against 8.1.2. If it is required and the
   profile is not eligible (no credential present **and** not credential-less) →
   `CONFIG_INVALID` with remediation.
2. **Unset (auto)** → select the first profile, by a fixed deterministic
   precedence, that both (a) is **eligible** — a credential is present, or the
   profile is credential-less (e.g. a local endpoint) — and (b) can serve the
   capability. This generalizes the capability-driven activation rule already
   used by rerank (8.4) and STT (8.2).
3. **None qualify** → a *required* capability (`embed`) fails the startup
   preflight; an *optional* one (`rerank`) stays off silently.

#### 8.1.4 Embeddings are a corpus-lifetime invariant

Vectors from different embed providers/models — **or from the same provider/model
served at a different endpoint** — are not comparable. The embed
**identity** — provider, **the normalized embed endpoint `base_url` (8.1.1)**,
per-axis model, **and the requested output dimension**
(8.1.6, recorded as `embed_text_dim`/`embed_code_dim`, df-003 §5.5) — is bound to
the index at first build and recorded in the config snapshot. On load, if the
configured embed identity differs from the index's, the server MUST refuse to mix
vector spaces — either erroring (`CONFIG_INVALID`) or triggering a full reindex.
`embed.provider`/**the normalized `base_url`**/`embed.text_model`/`embed.code_model`/`embed.text_dim`/`embed.code_dim`
— **and the multimodal mode (td-002)**, **the late-chunking mode
(`ingest.late_chunking`)** **and the contextual-retrieval mode
(SPEC §8.1.8)** — are therefore deploy-time, reindex-bound choices;
`chat`/`ocr`/`stt`/`rerank` providers are runtime-swappable. The input role
(8.1.5) is **not** part of this identity.

**The identity tuple (ordered).** The full pipe-delimited identity is
`provider | base_url | text_model | code_model | text_dim | code_dim | multimodal | late_chunking | contextual`.
New fields are **appended**, never inserted, so every extension is a
backward-compatible migration. `contextual` is the terminal field; `late_chunking`
sits between `multimodal` and `contextual`.

`late_chunking` is the literal `on`/`off` rendering of `ingest.late_chunking`
(SPEC §8.1.4, §16; dir2mcp #332/#446). Late chunking embeds the whole document
through a long-context model and pools each chunk's token vectors, so its chunk
vectors are **not** comparable to chunk-then-embed vectors from the same
provider/model — toggling the mode MUST re-derive rather than mix vector spaces.
It is the one component derived from a **config** key rather than a provider
attribute, and it is deliberately conservative: it records the configured flag,
not the runtime token-embedding capability, so it re-derives even where the
graceful chunk-then-embed fallback means no vector changed. A named token
(`on`/`off`) rather than a bare boolean leaves room for future pooling modes
without a further field migration.

`contextual` is `off` when contextual retrieval (SPEC §8.1.8) is disabled or
falls open to `off`, and
otherwise `ctx:<hash>` — a single opaque token hashing a canonical serialization
of **every** context-generation input (generator provider **+ normalized
endpoint**, model, `max_tokens`, `prompt_version`, and the effective prompt text;
an operator prompt override is folded in via its content). The hash makes the
nested identity collision-free against the outer `|` delimiter and ensures a
change to **any** generator input re-embeds rather than reusing vectors built
from a different generator. The per-chunk `embedding_mode` (df-003 §5.3) is
**not** part of this identity.

**Migration ladder.** A recorded identity MUST be canonicalized to the current
9-field form, by field count, before comparison; every pre-`base_url` form also
gains an empty `base_url` inserted at position 2 (SPEC §8.1.4):

| Recorded fields | Canonicalization |
|---|---|
| 3 (pre-8.1.6) | insert empty `base_url`, append `0`, `0`, `off`, `off`, `off` |
| 5 (pre-td-002) | insert empty `base_url`, append `off`, `off`, `off` |
| 6 (pre-late-chunking) | insert empty `base_url`, append `off`, `off` |
| 7 (pre-`base_url`) | insert empty `base_url`, append `off` |
| 8 (pre-contextual) | append `off` |
| 9 | unchanged |

Each canonicalized value is exactly what a fresh build with the newer features
disabled computes, so the identities **compare equal** and **no existing corpus
spuriously reindexes** (the migrated string gains components, but the vectors and
the comparison outcome are unchanged). An empty recorded identity (fresh index)
always passes; an unrecognized field count is left unchanged and fails loudly.

**Why `base_url` is part of the identity.** Two profiles with the same `kind` and
model name pointed at **different** endpoints (e.g. two `kind: openai` self-hosted
vLLM/Ollama deployments, or a proxy vs. the hosted API) serve **different** vector
spaces. Without `base_url` in the identity they collapse to one identity and their
vectors can silently mix in a single index — a violation of the "MUST refuse to
mix vector spaces" rule above. Including the endpoint closes that gap.

**`base_url` normalization (normative).** `base_url` enters the identity in
**canonical, normalized** form so that trivially-different-but-equivalent URLs do
not fragment the identity and force needless re-embeds. The recorded value is
computed as follows:
1. **Not-meaningful → empty.** For a `kind` whose embed endpoint is a single
   canonical provider surface that does not select an alternate model space
   (native `gemini`, `cohere`), the normalized `base_url` is the **empty string**
   `""` — `base_url` does not participate in the identity for that provider.
2. **Canonical/default → empty.** If the effective `base_url` is unset, or equals
   the built-in profile's shipped canonical `base_url` for that provider (e.g.
   `kind: openai` at `https://api.openai.com/v1`, the default `mistral` profile at
   `https://api.mistral.ai/v1`), it normalizes to `""`. Only an
   operator-**overridden**, non-canonical endpoint (the exact mis-bind case)
   yields a non-empty component.
3. **URL canonicalization** (applied before comparison, for the non-empty case):
   lowercase the scheme and host; remove the default port (`80` for `http`, `443`
   for `https`); strip trailing slash(es) and collapse duplicate slashes in the
   path; **preserve** the remaining path (e.g. `/v1`, which can select a different
   API mount); drop any userinfo, query, and fragment; apply canonical
   percent-/IDN-encoding. The result is compared exactly (path remains
   case-sensitive after host lowercasing).

**`""` is a valid identity component.** The empty string is a first-class,
legitimate value of the `base_url` component, not a sentinel for "unknown".
Consequently an index built **before** this rule — which recorded no `base_url` —
is treated as having `base_url == ""` and remains **valid** on reload against any
provider whose normalized `base_url` is also `""` (all built-in/hosted-default
deployments, per rules 1–2). Only a corpus whose embed endpoint is a
**non-canonical / custom** `base_url` sees a one-time `CONFIG_INVALID`/reindex on
first reload after this change — the correct, bounded safety action, since those
are exactly the corpora previously at risk of silent cross-endpoint mixing.

#### 8.1.5 Asymmetric embeddings (input role)

Some embedding providers (notably **Cohere** via `input_type`, **Gemini** via
`taskType`, and Voyage) are **asymmetric**: documents and queries MUST be
embedded with a distinct input role to achieve their stated retrieval quality.
Therefore:

* Every embedding call carries an **input role** ∈ {`document`, `query`}:
  corpus/index-time embeddings use `document`; search-time query embeddings use
  `query`. The role is determined by the call site, not by configuration.
* Adapters for asymmetric providers MUST map the role to the provider's
  mechanism. Adapters for symmetric providers (OpenAI, Mistral) MUST accept the
  role and MAY ignore it; behavior MUST NOT differ for symmetric providers.
  * **Cohere**: `input_type=search_document` (role `document`) / `search_query`
    (role `query`).
  * **Gemini** (native embed surface): `taskType` MUST be sent on every call.
    Role `document` → `RETRIEVAL_DOCUMENT`; role `query` → `RETRIEVAL_QUERY`.
    **Code-aware refinement:** when the call uses the configured **code** model
    (`embed.code_model`), role `query` maps to `CODE_RETRIEVAL_QUERY` (code
    documents still embed as `RETRIEVAL_DOCUMENT`, since Gemini has no
    code-specific document task). A `gemini` profile configured as `kind: openai`
    (OpenAI-compatible endpoint) cannot send `taskType` and is therefore treated
    as symmetric.
* The input role is **not** a configuration knob and does not affect the
  corpus-lifetime invariant (8.1.4): the recorded embed identity is provider +
  model + requested dimension (8.1.6), independent of role.
* The reference `Embedder` interface gains the role parameter (a clean, internal,
  pre-1.0 break — no compatibility users); see
  [Design 0001 §5.6](../../design/0001-multi-provider.md).

#### 8.1.6 Configurable embedding dimensionality (Matryoshka / MRL)

Some embedding models (notably **Gemini** `gemini-embedding-001`) are trained
with Matryoshka Representation Learning: a single model emits a high-dimensional
vector (Gemini native **3072**) whose leading prefix MAY be truncated to a
smaller dimension (e.g. **1536**, **768**) with graceful quality degradation.
Therefore:

* `model.embed.text_dim` / `model.embed.code_dim` are **optional** config knobs
  requesting a specific output dimensionality per axis. Omitted ⇒ the model's
  native dimension. The default for `gemini-embedding-001` is its native
  **3072**.
* When a non-native dimension is requested, the adapter MUST (a) request it from
  the provider where supported (e.g. Gemini `outputDimensionality`) and (b)
  **re-normalize** the returned vector to unit L2 length — MRL-truncated vectors
  below the native dimension are not pre-normalized, and the index's cosine/IP
  scoring assumes unit vectors.
* The requested dimension is part of the corpus-lifetime embed identity (8.1.4):
  it is recorded as `embed_text_dim`/`embed_code_dim` (df-003 §5.5), and changing
  it forces a reindex / `CONFIG_INVALID` on mismatched reload, exactly like a
  model change.
* A provider/model that does not support a requested dimension (no MRL, or a
  value its model cannot serve) MUST fail with `CONFIG_INVALID` rather than
  silently ignoring the knob, so an operator never believes a dimension is in
  effect when it is not.

#### 8.1.7 Multimodal embeddings (optional)

Multimodal embeddings — the optional **natively-multimodal** embed mode (text and
media mapped into one shared vector space), the `model.embed.multimodal`
tri-state knob, and media chunking / windowing (standalone image, PDF page,
audio/video time-window) — are specified in **td-002** (migrated from SPEC.md
§8.1.7). The 8.1.2 capability matrix is unchanged: multimodality is a property of
the chosen embed model, not a new capability cell.

### 8.2 STT providers

* STT provider is selected per 8.1.3 among STT-capable profiles (8.1.2):
  **Mistral** (Voxtral), **ElevenLabs** (Scribe), **OpenAI** (Whisper /
  `gpt-4o-transcribe`), **Gemini**. Default profile: **Mistral**. The set of
  STT-capable providers is **open** (not a closed enum) — any profile whose
  `kind` carries the `stt` capability (8.1.2) qualifies.
* Outputs MUST be normalized to the same `transcript` representation format
  regardless of provider.
* **Gemini** transcribes via the native `models/{model}:generateContent` surface:
  the audio is sent as an inline-data part (base64, with its MIME type) alongside
  a transcription instruction, and the model's text output is the transcript.
  Gemini's OpenAI-compatible layer exposes no `/v1/audio/transcriptions`, so the
  native surface is required (a `kind: openai` Gemini profile is therefore not
  STT-capable). The `stt_model` (default `gemini-2.5-flash`) and optional
  `stt_language` apply as for other providers.

### 8.3 Note on TTS

* TTS is optional and not required for core retrieval/inspection functionality.
* When used, the TTS provider is selected per 8.1.3 among TTS-capable profiles
  (8.1.2): **ElevenLabs**, **OpenAI** (`/v1/audio/speech`), **Gemini**. As with
  STT, the set of TTS-capable providers is **open**, not a closed enum.
* It must remain additive and must not break non-TTS workflows; a TTS provider
  error fails open (the workflow proceeds without audio).
* **Gemini** synthesizes via the native `models/{model}:generateContent` surface
  with `generationConfig.responseModalities: ["AUDIO"]` and a `speechConfig`
  voice (`tts_voice`, default `Kore`); the TTS model is `tts_model` (default
  `gemini-2.5-flash-preview-tts`). Gemini returns raw single-channel PCM (signed
  16-bit little-endian, 24 kHz) as inline data; the adapter MUST wrap it in a
  self-describing container (WAV) so the bytes are directly playable, matching
  the ready-to-play audio the ElevenLabs/OpenAI adapters return. Gemini's
  OpenAI-compatible layer exposes no `/v1/audio/speech`, so the native surface is
  required.

### 8.4 Rerank providers (optional)

* Reranking is **optional** and **capability-driven**: it activates automatically
  when a rerank provider credential is present and is disabled otherwise. No
  explicit enable flag is required (this mirrors how embedding/OCR providers
  activate on credential presence under the server-first preflight model).
* `rerank.enabled` is an **optional override**, not the activation switch:
  * unset → auto (reranking on **iff** a credential is present);
  * `false` → force reranking **off** even when a credential is present;
  * `true` → require reranking — if no credential is present the server MUST fall
    back (fail-open) and SHOULD emit a warning.
* Optional rerank provider: **Cohere** (`POST /v2/rerank`, default model
  `rerank-v3.5`). The set of rerank-capable providers is **open** (not a closed
  enum) — any profile whose `kind` carries the `rerank` capability (8.1.2)
  qualifies.
* When active, the reranker re-scores the fused candidate pool before truncation
  to `k` (see bs-003 §9.1.1).
* Reranking MUST be **fail-open**: any provider error (missing key, network
  failure, non-2xx) falls back to the pre-rerank fused order and MUST NOT fail
  the query.
* The rerank API key follows the same secret-source rules as other provider
  credentials (bs-011 §16.1.1) and MUST NOT be persisted to the config snapshot.

### 8.5 Self-hosted / OpenAI-compatible provider endpoints

A **self-hosted model server** is a **first-class provider** when it conforms to
the OpenAI-compatible contract: it is declared as a `kind: openai` profile
(8.1.1) whose `base_url` points at the self-hosted endpoint. **No new `kind` is
introduced** — a self-hosted server is just an `openai`-kind profile on a
non-OpenAI `base_url`, exactly like Ollama/vLLM/LM Studio already are (8.1.1).

* **Credential-less by default.** A self-hosted endpoint on a trusted network MAY
  have **no `api_key`** and is therefore credential-less (8.1.1).
  Credential-less self-hosted profiles are still **eligible** for selection and
  auto-selection (8.1.3) and pass preflight — they are not second-class.
* **Capability mapping** (which OpenAI-compatible route serves each capability):
  * **embed** → `POST /v1/embeddings` (e.g. Hugging Face TEI, vLLM, Infinity).
  * **chat** → `POST /v1/chat/completions`.
  * **stt** → `POST /v1/audio/transcriptions` (e.g. a faster-whisper or
    whisper.cpp server). As with any `kind: openai` audio route, STT here is
    **endpoint-dependent** and **validated at first use** (8.1.2 footnote ³), not
    statically rejected — an arbitrary self-hosted `base_url` may or may not
    expose `/v1/audio/transcriptions`, and that can only be known when it is
    called.
  * **ocr** has **no OpenAI analog** — OCR is a bespoke surface (8.1.2 shows
    `ocr` as `❌` for `kind: openai`); a self-hosted OCR server is not reachable
    through this contract.
* **Embed identity.** A self-hosted **embed** endpoint is bound by the
  corpus-lifetime embed identity (8.1.4) like any other embed provider: changing
  the self-hosted embed model (or its endpoint such that the model changes)
  forces a reindex / `CONFIG_INVALID` on mismatch.
* **STT normalization.** A self-hosted STT response is normalized to the
  `transcript` representation as defined in **td-003** (transcript
  representation, timestamps, language) — this section does not re-define it; see
  td-003 §8.6.1.
* **No shipped defaults.** dir2mcp ships **no per-deployment default and no
  built-in self-hosted profile** — there is no canonical self-hosted `base_url`
  to guess. The operator MUST declare the self-hosted profile explicitly in
  config (bs-011 §16.2).

### 8.8 Detected-language resolution (representation language)

A representation's recorded language (df-003 §5.2 `language`, `language_source`,
`language_confidence`) enables multilingual-corpus filtering and per-language
retrieval (bs-003 §9.5). Recording it is **optional, additive, and best-effort**;
it MUST NOT make ingestion fail.

* **Auto-detect by default; pin optional.** Language detection is **on by default
  and best-effort**: an implementation SHOULD record a representation's language
  when it can determine one (a `transcript` already does, td-003 §8.6.2; OCR and
  plain text MAY add it). An operator MAY pin the language (`media.language` /
  per-provider `stt_language`, bs-011 §16.2; an analogous pin for non-media text
  is implementation-defined and optional). No fixed or default language is
  assumed — the surface is general-purpose and language-agnostic.
* **Resolution precedence.** When more than one signal is available, the recorded
  effective `language` MUST be resolved deterministically with this precedence,
  and `language_source` MUST record which signal won:
  1. **`configured`** — an explicit operator pin always wins (bs-011 §16.2).
  2. **`declared`** — a language asserted by the source itself (sidecar suffix
     td-003 §8.6.4, document/track language tag, OCR-provider-reported language).
  3. **`detected`** — an auto-detector's best-effort result.
  A translated transcript's effective `language` is its **target** language
  (td-003 §8.6.2), recorded independently of the above.
* **Graceful degradation (absent ⇒ unknown, never an error).** When no signal is
  available — no pin, no declaration, and detection is unavailable, fails, or
  returns below a configured confidence floor — the representation records **no**
  `language` and is treated as **unknown language**. Unknown is a first-class,
  non-error state: ingestion, indexing, retrieval, and citation all proceed
  exactly as today; only per-language filtering (bs-003 §9.5) is affected.
* **Confidence floor (optional).** An implementation MAY apply a configured
  minimum confidence at detection time and decline to record a low-confidence
  `detected` language (leaving it unknown). Once a `language` value is written it
  is authoritative for retrieval matching (bs-003 §9.5); `language_confidence` is
  informational and MUST NOT be re-applied as a filter at query time.
* **Stability & re-derivation.** Detection MUST be deterministic for identical
  input + detector so the recorded language is stable across re-indexing. The
  detector/pin is **not** part of a representation's derivation identity (td-003
  §8.6.7) unless an implementation chooses to make a *pin change* trigger
  re-derivation; a pure detector change MAY refresh `language` opportunistically
  without forcing re-embedding (language metadata does not change chunk `text`).

## Changelog

- **0.5.0** — §8.1.4: recorded `late_chunking` as the 8th embed-identity field,
  between `multimodal` and `contextual` (SPEC §8.1.4; dir2mcp #332/#446). The
  reference implementation has recorded it since #446 but it was never specified,
  so the documented tuple and the shipped one had diverged on the 8th slot.
  Documents why the mode is corpus-lifetime (context-pooled vectors are not
  comparable to chunk-then-embed vectors), that it uniquely derives from a config
  key, the conservative config-not-capability gate, the `on`/`off` named token,
  and the full field-count migration ladder (3/5/6/7/8 ⇒ 9). Mirrors the §6.4
  tuple in bs-008.
- **0.4.0** — §8.1.4: appended `contextual` as the terminal embed-identity field
  for contextual retrieval (SPEC §8.1.8; dir2mcp #330). Documented the ordered
  identity tuple, the deterministic no-reindex append (`…|multimodal` ⇒
  `…|multimodal|off`, which compares equal to a fresh contextual-off build), and
  that the enabled value is a `ctx:<hash>` token over the full generator identity
  (provider+endpoint, model, max_tokens, prompt_version, effective prompt).
  Mirrors the §6.4 tuple in bs-008 and the SPEC §8.1.4 amendment.
- **0.3.0** — §8.1.2: clarified that document/image extraction-engine selection
  is a td-004 §B.1 routing decision, not a capability cell here; the `mistral`
  extraction engine binds the `ocr` capability (§8.1.3). No matrix cell added.
- **0.2.0** — Added the normalized embed endpoint `base_url` to the
  corpus-lifetime embed identity in §8.1.4, with normalization rules
  (not-meaningful/default → empty; scheme/host/port/trailing-slash
  canonicalization) and the back-compat rule that pre-existing indexes with no
  recorded `base_url` are treated as `""` and stay valid. Mirrors the §6.4 tuple
  in bs-008. Unblocks dir2mcp #560.
- **0.1.0** — Migrated from SPEC.md §8.1–§8.5 and §8.8. §8.1.7 (multimodal
  embeddings & media chunking) is **not** included here — it moves to td-002, and
  8.1.7 is replaced by a one-line pointer. §8.6 (→td-003) and §8.7 (→td-005) are
  out of scope. Cross-references rewired to stable doc IDs: §1→df-000; §5→df-003
  (incl. §5.2/§5.5); §6→bs-008; §7→bs-002; §7.4→td-004; §8.1.7→td-002; §8.6→td-003
  (incl. §8.6.1/§8.6.2/§8.6.4/§8.6.7); §8.7→td-005; §9→bs-003 (incl. §9.1.1/§9.5);
  §14→df-008; §16→bs-011 (incl. §16.1.1/§16.2). Internal §8.x cross-references are
  preserved as in-document section numbers; the `Design 0001` link is repointed to
  `../../design/0001-multi-provider.md`. Added explicit "open set, not a closed
  enum" notes to the STT/TTS/rerank provider lists (faithful to §8.1.3's
  auto-selection semantics).
