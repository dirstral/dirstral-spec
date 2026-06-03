# Versioning Policy

## Spec versioning

The spec uses [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

| Change type | Version bump |
|-------------|-------------|
| Breaking wire/schema behavior | Major |
| New optional fields, new optional tools | Minor |
| Clarifications, doc fixes | Patch |

**Pre-1.0 (beta) policy.** While the spec is `0.x` the project is pre-institutional and treated as **beta**: the `MAJOR` component stays `0`; **both** breaking wire/schema changes **and** new optional fields/tools bump the `MINOR` (e.g. `0.4.0 → 0.5.0`); only clarifications/doc-fixes bump the `PATCH`. (The SemVer table above describes post-`1.0` semantics — breaking → `MAJOR`, new optional → `MINOR` — and takes effect at `1.0.0`. The "Non-breaking additions" section below remains accurate: new optional surface is a `MINOR` bump in either regime.)

**Current spec version:** `0.14.0`
**MCP protocol target:** `2025-11-25`

## Implementation compatibility

Each implementation declares the spec version(s) it supports. `dirstral-cli` validates the supported spec version at runtime during `initialize`.

## Compatibility matrix

| Impl | Supported spec versions | Notes |
|------|------------------------|-------|
| `dir2mcp` (Go) | `0.14.x` (pending) | Reference implementation used for spec validation; reviewed against `internal/` as of 2026-06-04. The spec is authoritative — when discrepancies arise, maintainers file a spec-gap issue and decide whether to correct the spec or the implementation. Native Gemini embedding parity (`taskType`, MRL `outputDimensionality`, #222) and native Gemini STT/TTS (#223) shipped. The multimodal-embedding arc (`gemini-embedding-2`, §8.1.7) shipped phased + default-off: adapter slice (#224), image ingestion (#225), per-page PDF (#226), audio/video time-window embedding (#227, `0.14.0`), retrieval dedup + result modality (#228), and `open_file` `MEDIA_NO_TEXT` + ask-over-media grounding (#229); docling adapter contract CI (#230). The model is Public Preview, so this row stays pending until the implementation releases against the GA-verified model. |
| `dirstral-cli` | `0.4.x` | MUST update to `0.7.x` before releasing against spec `0.7.0`. No client code change for `0.6.0`/`0.7.0` (reranking and multi-provider selection are server-side; the wire/result contract is unchanged); the `0.5.0` tool-name rename remains the only wire-visible delta in this range. |
| `landfall` | TBD | |

## Contract freeze (issue #104)

As of spec version `0.4.0`, the following machine-readable artifacts have been added:

- `spec/tools/schemas/` — JSON Schema Draft-07 files for all 9 tools
- `spec/errors/taxonomy.md` — complete error code table including tool-execution errors
- `spec/sessions/lifecycle.md` — session expiry and `X-MCP-Session-Expired` header documented
- `spec/x402/extension.md` — `upto` scheme and `maxAmountRequired` field documented

Spec gaps identified during the review (see `<!-- spec-gap: ... -->` comments in each file):

- `SESSION_NOT_FOUND` JSON-RPC code was documented as `-32002`; implementation uses `-32001`
- `UNAUTHORIZED` JSON-RPC code was documented as `-32001`; implementation uses `-32000`
- Error `data` envelope (`{"code": ..., "retryable": ...}`) was not documented
- Tool execution errors return HTTP 200 with `isError: true`; this was not explicitly stated
- Several error codes (`MISSING_FIELD`, `INVALID_FIELD`, `INVALID_RANGE`, `STORE_CORRUPT`, `INTERNAL_ERROR`, `FORBIDDEN_ORIGIN`, `METHOD_NOT_FOUND`) were absent from the taxonomy

## 0.14.0 — audio/video media windowing

Extends the multimodal-embedding surface (0.13.0, §8.1.7) with normative
**time-window chunking** for audio and video direct embedding under
`model.embed.multimodal` `augment`/`replace`, completing the modality set
after images and PDFs. `MINOR` bump per the pre-1.0 policy (refines optional
multimodal behavior; no new tool, error code, config field, span kind, or
wire-contract change — `time` spans already exist, §5.4). The §8.1.2
capability matrix is unchanged.

- §8.1.7 **(new) Media chunking (windowing)** — audio/video are split into
  non-overlapping, contiguous **time windows** (one media chunk each, `time`
  span) that MUST respect both the per-modality duration cap (audio ≤ 180 s,
  video ≤ 120 s) and the unified 8192-token budget; window boundaries are
  deterministic for stable citations. Image = one chunk; PDF = one chunk/page.
- §8.1.7 **fallback** — a file whose duration cannot be determined is a
  non-fatal per-document condition (§7.7): not directly embedded, warned;
  modalities with a text path (image/PDF OCR, audio transcript) keep that text
  representation even under `replace`, while video (no text path) is left
  unindexed.
- §8.1.7 **video** — has no default text representation (no video→text
  analogue to audio STT, §7.4.C): searchable only via media windows.
- §5.1 / §7.3 — `video` added to the `documents.doc_type` enumeration and the
  type-classification list (`.mp4`, `.mov`).
- No new tool, error code, config field, span kind, or wire-contract change.
- Implementation note: lands in a follow-up dir2mcp code PR (Phase 2c); the
  reference impl reads media duration via an `ffprobe` subprocess with a
  graceful skip when absent.

## 0.13.0 — multimodal embeddings (gemini-embedding-2)

Promotes [Design 0003](../docs/design/0003-multimodal-embeddings.md) to normative
spec: optional native **multimodal embeddings** that map text + images + audio +
video + PDFs into one shared vector space, via Google `gemini-embedding-2`,
behind a per-corpus `model.embed.multimodal` toggle. `MINOR` bump per the pre-1.0
policy (new optional config + one new tool-execution error code; no wire/tool
shape change). The §8.1.2 capability matrix is unchanged — multimodality is a
property of the chosen embed model, not a new capability cell. The model is
**Public Preview**: §8.1.7 carries a re-verify caveat and the implementation is
phased + GA-gated (compatibility row stays pending).

- §8.1.7 **(new) Multimodal embeddings** — `gemini-embedding-2`, modalities +
  per-request limits + the unified 8192-token window, the
  `model.embed.multimodal` (`off|augment|replace`) toggle, the single-shared-
  space constraint (`augment`/`replace` ⇒ `gemini-embedding-2` for all
  modalities, else `CONFIG_INVALID`), reindex-bound mode, provenance reusing
  existing span kinds, the page-image retrieval dedup rule, the `ask` grounding
  rule, and inspection via `MEDIA_NO_TEXT`.
- §8.1.4 **embed identity** — gains the multimodal mode.
- §14.2 **error taxonomy** — new non-retryable `MEDIA_NO_TEXT` (`open_file` on a
  `replace`-mode media chunk with no text representation); mirrored in
  `spec/errors/taxonomy.md`.
- §15.4 **`open_file`** — documents the `MEDIA_NO_TEXT` outcome.
- §16.2 **config template** — `model.embed.multimodal` (default `off`).
- No new tool, span kind, or wire-contract change.
- Implementation note: lands in phased follow-up dir2mcp code PRs (adapter →
  ingestion → store → retrieval → `ask` → `replace`), starting with the
  default-off adapter slice; preview limits/endpoints re-verified at GA.

## 0.12.0 — native Gemini STT/TTS

Pins the native wire mechanism for the already-`✅` Gemini STT and TTS
capability cells (8.1.2). Gemini's OpenAI-compatible layer does not expose
`/v1/audio/*`, so audio must use the native `models/{model}:generateContent`
surface — this release makes that normative and defines the TTS audio
container. `MINOR` bump per the pre-1.0 policy (provider-behavior change on
already-`✅` matrix cells; no new tool, error code, config field, or
wire-contract change — STT/TTS knobs `stt_model`/`stt_language`/`tts_model`/
`tts_voice` already exist since 0.7.0).

- §8.1.1 **provider profiles**: `gemini` STT/TTS clarified as native
  (`generateContent`); the `kind: openai` Gemini path serves chat only.
- §8.2 **STT**: Gemini transcribes via `generateContent` with the audio as
  an inline-data part; output normalized to the `transcript` representation
  like every other provider. `stt_model` default `gemini-2.5-flash`.
- §8.3 **TTS**: Gemini synthesizes via `generateContent` with
  `generationConfig.responseModalities: ["AUDIO"]` + a `speechConfig` voice (`tts_voice`,
  default `Kore`; `tts_model` default `gemini-2.5-flash-preview-tts`). The
  returned raw PCM (s16le, 24 kHz, mono) MUST be wrapped in a WAV container
  so the bytes are directly playable, matching ElevenLabs/OpenAI. TTS stays
  fail-open (8.3).
- No new tool, error code, config field, span kind, or wire-contract
  change; the §8.1.2 matrix is unchanged (`gemini` STT/TTS were already `✅`).
- Implementation note: the native Gemini STT/TTS backend lands in a
  follow-up dir2mcp code PR once this spec change is merged (replacing the
  current OpenAI-compat `/audio/*` shim, which Gemini does not serve).

## 0.11.0 — native Gemini embedding parity (taskType + Matryoshka)

Promotes the `gemini` embed adapter from the OpenAI-compatible shim to Gemini's
**native** embed surface so it reaches feature parity with `gemini-embedding-001`:
asymmetric `taskType` (document/query, with a code-aware refinement) and
configurable Matryoshka output dimensionality. `MINOR` bump per the pre-1.0
policy (new optional config fields + provider-behavior change on an already-`✅`
matrix cell; no new tool, error code, or wire-contract change). The §8.1.2
matrix is unchanged (`gemini` embed was already `✅`).

- §8.1.1 **provider profiles**: `gemini` embed clarified as native, **asymmetric**
  via `taskType`, with Matryoshka output dimensionality; the OpenAI-compatible
  alternative forgoes `taskType` and stays symmetric.
- §8.1.4 **embed identity**: the requested output dimension joins provider+model
  in the corpus-lifetime embed identity (recorded as `embed_text_dim`/
  `embed_code_dim`, §5.5); changing it forces a reindex / `CONFIG_INVALID`.
- §8.1.5 **asymmetric embeddings**: `gemini` added alongside Cohere/Voyage. Role
  mapping: `document`→`RETRIEVAL_DOCUMENT`, `query`→`RETRIEVAL_QUERY`; for the
  configured **code** model a `query` maps to `CODE_RETRIEVAL_QUERY`.
- §8.1.6 **configurable embedding dimensionality (Matryoshka/MRL)** (new):
  optional `model.embed.text_dim`/`code_dim`; adapters request `outputDimensionality`
  where supported and **re-normalize** truncated vectors; unsupported dimensions
  are `CONFIG_INVALID`, never silently ignored.
- §16.2 **config template**: `model.embed` gains optional `text_dim`/`code_dim`
  (commented; native dimension when omitted). No provider/matrix change.
- No new tool, error code, span kind, or wire-contract change. STT/TTS remain
  `✅` in the matrix; native Gemini STT/TTS implementation is a separate slice.
- Implementation note: the native Gemini embedding backend (and, separately,
  native Gemini STT/TTS) land in follow-up dir2mcp code PRs once this spec
  change is merged.

## 0.10.0 — docling-serve HTTP extraction transport (docling-serve)

Adds `docling-serve` as an `ingest.extractor` value: an alternative *transport*
for docling extraction that talks to a local docling-serve HTTP container
instead of the docling CLI subprocess. Extraction remains its own selection axis
(`ingest.extractor`), independent of the §8 model/provider bindings — it is
deliberately **not** modeled as a provider capability (the §8.1.2 matrix is
unchanged). `MINOR` bump per the pre-1.0 policy (new optional `ingest.extractor`
value + one new optional config field, non-breaking). Output is byte-identical
to the docling CLI path (same `extracted_markdown` + `region` spans from 0.9.0).

- §7.4.B **representation**: `ingest.extractor` gains `docling-serve`. The `ingest.extractor` value selects the transport explicitly (`docling` = CLI subprocess, `docling-serve` = HTTP service); both transports MUST produce identical output, and extraction is independent of the §8 provider bindings. Selecting `docling-serve` requires a non-empty, reachable `serve_url`; an empty or unreachable endpoint disables that extractor for diagnostics (§7.7), like a missing docling binary, and MUST NOT silently fall back to the CLI. (Under `extractor: auto` the transport is implementation-determined.)
- §16.2 **config template**: add `ingest.docling.serve_url` (empty by default; required when `extractor=docling-serve`, otherwise the HTTP transport is simply not used). One new optional config field; no provider/matrix change.
- No new tool, error code, span kind, provider kind, or wire-contract change. The MCP tool surface and persisted store shape are unchanged.
- Implementation note: the dir2mcp docling-serve backend lands in a follow-up code PR once this spec change is merged (HTTP extractor reusing the existing DoclingDocument parser; doctor probes the endpoint's `/ready`). Container lifecycle is user-managed (dir2mcp probes and fails fast; it does not start/stop the container).

## 0.9.0 — structured docling extraction (region provenance)

Formalizes structured `DoclingDocument` ingestion as the docling extraction contract (previously flat Markdown) and adds region-level provenance for precise citations. `MINOR` bump per the pre-1.0 policy (new optional span kind + new optional citation fields, non-breaking — clients that ignore the new fields continue to work). Design: [docs/design/0002-structured-extraction.md](../docs/design/0002-structured-extraction.md).

- §5.4 **spans**: new `region` span kind — page range in `start`/`end`, with `bbox` (primary-page bounding box), `section` breadcrumb, and element `label` carried in the existing `extra_json` column. **No schema migration** (reuses `extra_json`).
- §7.4.B **representation**: structured extraction preserves reading order, section hierarchy, per-element page/bbox provenance, atomic tables, and figure captions/classifications; title from the model's title element. The persisted `extracted_markdown` representation remains rendered Markdown (structure lands in spans); raw `DoclingDocument` JSON is an implementation-private cache, not a representation. Page-separated OCR fallback (Mistral) is unchanged.
- §7.5 **chunking**: section/element-aware chunking for structured documents (group by section breadcrumb, keep tables atomic).
- §15.1.1 **`Span`** (client-facing) + `spec/tools/schemas/common.json`: additive `region` variant (`start_page`/`end_page`, required `bbox`, optional `section`). A `region` span always carries a `bbox` — an element without provenance is recorded as a `page` span instead. The machine-readable `common.json` `Span` (previously the drifted minimal `{start,end}` shape) is brought in line with the kind-tagged `§15.1.1` union (`lines|page|time|region|document`) so the authoritative JSON schema matches the prose. Backward compatible — existing kinds unchanged; clients MUST ignore unrecognized kinds/fields.
- §9.2 **result objects** / §9.3 **citation rendering**: `region` added to the hit span-kind list; region citations render the primary page (`bbox.page`), or a page range when `start_page != end_page`, optionally suffixed with the section breadcrumb.
- No new tool, no new error code, no config-shape change. `spec/errors/taxonomy.md` and `spec/sessions/lifecycle.md` are unaffected (header version bump only). A new `dirstral-conformance` test for the `region` span variant is recommended-not-required.
- Implementation note: the structured pipeline lands in a follow-up dir2mcp PR (extractor → `--to json`, store `extra_json` read/write, section-aware chunking, retrieval + MCP citation surface) once this spec change is merged.

## 0.8.0 — stats.recent_failures (per-document failure visibility)

Extends `dir2mcp_stats` output with an **optional** `recent_failures` array surfacing the most-recent documents with `status='error'` along with a short, sanitized `error_message`. `MINOR` bump per the pre-1.0 policy (new optional field on an existing tool, non-breaking — clients that ignore the field continue to work).

Motivation: a maintainer triaging a failed corpus today can see *that* documents failed (existing `indexing.errors` counter) but not *which* documents or *why*. The information already exists in the implementation's metadata store (per-document `error_message`) and ships in `dir2mcp support-bundle`'s `list-files.json`; this spec change makes it available programmatically through the spec-blessed diagnostic surface (`stats`) so doctor-style dashboards and remote diagnostics can render it without scraping the bundle.

- §15.6 `dir2mcp_stats` output: optional `recent_failures` array (`additionalProperties: false`), each item `{rel_path, doc_type, mtime_unix, error_message}` (all required when an item is present), newest-first by `mtime_unix`. Implementations SHOULD cap at 20 entries by default and SHOULD cap `error_message` at 512 bytes on a UTF-8 rune boundary with control characters stripped (one-line render). Implementations MAY omit the field entirely when no failures are recorded; clients MUST treat omission as "no recent failures" (not "unsupported") per the existing "Clients MUST ignore unknown fields" rule. `error_message` is normative as a **diagnostic** signal — it MUST NOT contain secrets, raw file content, or unsanitized provider response bodies.
- `spec/tools/schemas/stats.json`: mirrored.
- No new tool, no new error code, no config-shape change. `spec/errors/taxonomy.md`, the rest of `spec/tools/schemas/*`, and the `dirstral-conformance` suites are unaffected (a new conformance test for the optional field is recommended-not-required).
- Implementation note: dir2mcp reference impl 0.5.8+ persists per-document `documents.error_message` (additive SQLite migration; introduced in dir2mcp #212). The stats wiring lands in a follow-up dir2mcp PR once this spec change is merged.

## 0.7.0 — multi-provider model abstraction

Generalizes the model pipeline from Mistral-centric to **provider-agnostic**: every capability (embed/chat/ocr/stt/rerank) binds to a configurable provider profile. A `MINOR` bump per the pre-1.0 policy — it is both a config-shape break (the monolithic `mistral:` block is removed) and new optional surface; a clean break is acceptable (no compatibility users). Design: [docs/design/0001-multi-provider.md](../docs/design/0001-multi-provider.md).

- §1 **Implementation goal** rewritten provider-agnostic; Mistral is the default profile, not privileged.
- §8.1 **Provider model**: profiles (`kind` = `openai`/`mistral`/`anthropic`/`gemini`/`cohere`/`elevenlabs`), the OpenAI-compatible backbone covering OpenAI/OpenRouter/Groq/Azure/local **and Mistral chat+embed**, bespoke adapters only for non-OpenAI surfaces (Mistral `/v1/ocr`, Anthropic, Cohere rerank, ElevenLabs).
- §8.1.2 **Capability matrix** (normative): binding a capability to an incapable `kind` is `CONFIG_INVALID`.
- §8.1.3 **Provider selection**: explicit `<cap>.provider`, else capability-driven auto-pick by precedence among credentialed+capable profiles (generalizes the rerank/STT rule).
- §8.1.4 **Embeddings corpus-lifetime invariant**: embed identity is bound to the index; mismatched reload MUST error or reindex (no silent vector-space mixing).
- §8.1.5 **Asymmetric embeddings (input role)**: every embedding call carries a document/query input role; asymmetric providers (Cohere `input_type`, Voyage) MUST honor it, symmetric providers ignore it. The reference `Embedder` interface gains the role parameter (clean internal pre-1.0 break).
- **Full Cohere**: `kind: cohere` serves embed + chat (`/v2/chat`) + rerank in 0.7.0 (not rerank-only).
- **Provider-agnostic STT/TTS**: §8.2/§8.3 generalized — STT/TTS are selected per §8.1.3 among capable profiles (Mistral/ElevenLabs/OpenAI/Gemini for STT; ElevenLabs/OpenAI/Gemini for TTS). `kind: openai` audio is endpoint-dependent (validated at first use, never `CONFIG_INVALID`); every other matrix `✅` is statically valid. No provider is left half-wired.
- §16.2 config template: monolithic `mistral:` replaced by `providers:` map + `model:` capability bindings; `stt:`/`rerank:` shapes retained.
- §2.5 startup preflight generalized from "requires Mistral API key" to per-capability provider credentials.
- **No new tool, tool-schema field, or error code** (one new config-validation case reuses `CONFIG_INVALID`). `spec/tools/schemas/*` and `spec/errors/taxonomy.md` unchanged; `dirstral-conformance` unaffected.

## 0.6.0 — optional reranking (Cohere)

New **optional** retrieval-quality stage; capability-driven (auto-activates only when a rerank provider credential is present, off otherwise), non-breaking — `MINOR` bump per the pre-1.0 policy (new optional surface → `MINOR`).

- §8.4 **Rerank providers (optional)**: Cohere (`POST /v2/rerank`, default `rerank-v3.5`); capability-driven activation (auto-on when a credential is present, mirroring embedding/OCR provider gating); `rerank.enabled` is a tri-state override (unset → auto, `false` → force off, `true` → require + warn/fail-open if absent); fail-open; key not persisted.
- §9.1.1 **Optional reranking**: post-fusion re-scoring of the top `rerank.candidate_pool` (default 50) candidates before truncation to `k`; reorder-only (result structure §9.2 unchanged); `index=both` reranks once on the merged pool; deterministic tie-break by `chunk_id`.
- §16.2 config template: `rerank:` block (mirrors the `stt:` provider-selector shape).
- No new tool, tool-schema field, or error code (fail-open surfaces no new tool error). `spec/tools/schemas/*` and `spec/errors/taxonomy.md` unchanged.

## 0.5.0 — reconcile shipped dir2mcp (spec-gap resolution)

Protocol-council decision: the dir2mcp reference implementation had shipped behavior that diverged from canonical `0.4.0`. Per the pre-1.0 beta policy and the "spec is authoritative; maintainers decide spec-vs-impl direction" rule, all of the following were resolved **impl → spec** (the spec now ratifies shipped behavior); breaking deltas bump `MINOR` (`0.4.0 → 0.5.0`):

- **Tool naming** `dir2mcp.<tool>` → `dir2mcp_<tool>` (breaking; ratifies dir2mcp #172). The former dotted-namespace rule is **superseded** — underscore form is canonical across `docs/SPEC.md`, `spec/tools/schemas.md`, and every `spec/tools/schemas/*.json` title.
- **`rep_type` enum** `ocr_markdown` → `extracted_markdown` (breaking; ratifies dir2mcp #152 docling extractor abstraction).
- **`k` default** `10` → `15` for `search`/`ask`/`ask_audio`/`transcribe_and_ask` (ratifies dir2mcp #163).
- **`OCR_NOT_READY`** tool-execution error added + `open_file` binary-doc semantics + `span.kind="document"` variant (new optional; ratifies dir2mcp #180).
- **`serverInfo.name`** per-instance auto-derivation + `dir2mcp-dev-` prefix for dev builds (new optional; ratifies dir2mcp #184/#185).
- **x402 adapter**: facilitator defaults to the Coinbase x402 Go SDK client (clarification).

`dirstral-conformance` SHOULD extend suites for the renamed tool surface before any impl releases against `0.5.0`.

## Breaking change process

1. Open a spec PR with the proposed change
2. Maintainer review required (protocol council gate)
3. Bump the version in `spec/versioning.md` (while `0.x`: breaking → `MINOR` per the pre-1.0 policy; post-`1.0`: `MAJOR`)
4. All implementation repos must update their compatibility matrix before releasing against the new spec version
5. `dirstral-conformance` must add a new test suite for the new behavior

## Non-breaking additions

New optional tools or optional fields in existing tool schemas may be added in a minor version without breaking existing clients. Clients MUST ignore unknown fields.
