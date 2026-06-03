# Design 0003 — Multimodal embeddings (gemini-embedding-2)

**Status:** Proposed (targets `gemini-embedding-2` GA; proposed spec `0.13.0`)
**Author:** dirstral maintainers
**Extends:** SPEC §8.1.4–§8.1.6 (embed identity / asymmetric / dimensionality), [Design 0001 §5.6/§6](0001-multi-provider.md)
**Related:** SPEC §5.4 (spans), §7 (ingestion/representations), §8 (providers), §9/§15 (retrieval/citations), §16 (config)

## 1. Summary

Add **optional native multimodal embeddings** via Google's `gemini-embedding-2`
— Google's first natively multimodal embedding model, which maps **text,
images, audio, video, and PDFs into one shared vector space** — behind a
per-corpus `model.embed.multimodal` toggle (`off` | `augment` | `replace`).

This builds directly on the foundation already shipped:

- **dir2mcp #222** (spec 0.11.0) — the native `models/{model}:…` embed
  transport, `taskType` (asymmetric), Matryoshka `outputDimensionality` +
  re-normalization, and the dimension-bound corpus-lifetime embed identity.
- **dir2mcp #223** (spec 0.12.0) — the inline-media part encoding
  (`inlineData{mimeType, base64}`), exactly the request shape
  `gemini-embedding-2` consumes for non-text modalities.

The *adapter* delta is therefore small; the substance is the **ingestion,
store, and retrieval pipeline**, which is text-only today. Because
`gemini-embedding-2` is **Public Preview** (API may change before GA), this
design + the proposed normative spec deltas land now, and the SPEC.md
promotion + code land together at model GA.

## 2. Motivation

- dir2mcp is lossy for non-text content: images/PDFs go through OCR/docling
  and audio through STT, collapsing layout, figures, charts, handwriting,
  diagrams, and speech/acoustic cues into text before embedding. Direct
  multimodal embedding preserves that signal.
- One **shared embedding space** enables **cross-modal retrieval**: a text
  query can surface an image, a chart, an audio segment, or a PDF page —
  and vice versa — without a separate per-modality index.
- The transport already exists (§1), so the marginal cost is pipeline work,
  not a new provider adapter from scratch.

## 3. The model — `gemini-embedding-2`

Public Preview as of 2026-03-10 (Gemini API + Vertex AI). Native endpoint
`…/v1beta/models/gemini-embedding-2:embedContent` (and
`:batchEmbedContents`), `x-goog-api-key` auth — the same native surface the
existing `internal/gemini` adapter already uses.

> **GA promotion gate.** The limits, formats, endpoint, and auth below are
> from the **Public Preview** docs and MAY change. They MUST be re-verified
> against the then-current provider docs at SPEC-promotion time; if anything
> differs, update §3 and §8 in the same promotion PR.

| Modality | Per-request limit | Formats |
|---|---|---|
| Text | 8192 tokens | — |
| Images | ≤ 6 | PNG, JPEG, WebP, BMP, HEIC, HEIF, AVIF |
| Video | ≤ 120 s (1 FPS) | MP4, MOV |
| Audio | ≤ 180 s | MP3, WAV |
| PDF (documents) | 1 file, ≤ 6 pages | PDF |

All modalities share **one unified 8192-token context window** (≈ 258
tokens/image, 258 tokens/PDF page, 66 tokens/video frame, 25 tokens/audio
second), so a request mixing modalities must fit the *combined* budget —
chunking (§5) MUST account for this window, not just the per-modality caps.

Interleaved input is allowed (e.g. image + caption text in one request).
Output is **3072 dimensions** with Matryoshka truncation (recommended
3072 / 1536 / 768). `taskType` applies as for `gemini-embedding-001`.

Sources: [blog.google — Gemini Embedding 2](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-embedding-2/),
[Gemini API embeddings docs](https://ai.google.dev/gemini-api/docs/embeddings),
[Gemini Embedding 2 Preview model](https://ai.google.dev/gemini-api/docs/models/gemini-embedding-2-preview).

## 4. Configuration — `model.embed.multimodal`

A per-corpus tri-state on the embed binding:

- **`off`** (default) — current text-only behavior; **any** embed provider.
  No change for existing corpora.
- **`augment`** — keep the existing text extraction + text embeddings
  **and** additionally embed media files directly as media chunks. Both are
  indexed in the same space. Best recall (keyword/OCR text *and* visual /
  acoustic signal); higher embedding cost and index size.
- **`replace`** — for media files, embed the media directly **instead of**
  OCR/STT→text. Text files are unchanged. Simpler/cheaper index; loses
  keyword search over OCR'd content and complicates `ask` grounding (§7.3).

**Hard constraint (single shared space, SPEC §8.1.4):** when `multimodal` is
`augment` or `replace`, the embed provider for **every** modality, including
text, MUST be `gemini` with `gemini-embedding-2`. Mixing (e.g. Mistral text
+ Gemini media) would put incomparable vectors in one index and is
`CONFIG_INVALID`. `off` keeps full provider freedom.

**Reindex-bound:** the multimodal mode is part of the corpus-lifetime embed
identity (§6). Switching `off`↔`augment`↔`replace`, like changing the model
or dimension, changes what/how content is indexed and requires a reindex.

## 5. Ingestion pipeline

Today: discover → extract (OCR/docling/STT) → **text** chunks → embed text.
Multimodal introduces a second kind of embeddable unit.

- **Embeddable unit** abstraction: a *text chunk* (existing — text input) or
  a *media chunk* (a `(mimeType, bytes)` payload + a provenance span). Both
  flow through the same embed worker; the adapter sends a text part or an
  inline-data media part accordingly (#223 encoding).
- **Per-modality chunking**, respecting the §3 limits and keeping citation
  spans precise:
  - **image** → one unit (whole file).
  - **PDF** → page groups of ≤ 6 pages; **per-page preferred** for precise
    `page`/`region` citations. In `augment`, docling structured text (spec
    0.9.0) and the page-image embedding coexist for the same page.
  - **audio** → ≤ 180 s windows; **video** → ≤ 120 s windows. Window
    boundaries become `time` spans.
- Discovery filters (size caps, type allow-list, `.gitignore`, symlink
  policy) apply to media exactly as to text.
- The document/query input role (§8.1.5) carries over: index-time media
  embeds with the document task type, query-time with the query task type.

## 6. Store & index

- The vector space is shared, so the HNSW index is **dimensionally
  unchanged** (3072 or the configured truncation).
- A chunk row gains (additive migration): `modality` (`text|image|audio|
  video|pdf`), `media_ref` (path within the corpus), and the provenance
  `span`. Persisted spans reuse the existing **`span_kind ∈ {lines, page,
  time, region}`** (§5.4) — **no new persisted kind**: audio/video windows
  are `time`, PDF pages are `page`/`region`, and a standalone image maps to
  a single `page` (page 1). (`document` is *not* a persisted span_kind — it
  is a client-facing-only variant emitted by `open_file`, §15.1.1.)
- **Embed identity** (§8.1.4) extends to
  `provider | text_model | code_model | text_dim | code_dim | multimodal_mode`
  (the spec's per-axis model fields, not a single `model`). A reload whose
  identity differs from the index's errors/triggers reindex as today.
- When `multimodal` is on, the resolver MUST reject any non-`gemini` /
  non-`gemini-embedding-2` embed binding (§4 constraint) at startup.

## 7. Retrieval, citations, and the `ask` wrinkle

### 7.1 Cross-modal search
A text query is embedded by `gemini-embedding-2` (text) and retrieves any
chunk in the shared space, including media chunks. `search`, `list_files`,
and `stats` need no contract change beyond surfacing media hits.

**Dual-representation dedup (v1, deterministic).** In `augment`, a PDF page
yields *both* a docling text chunk and a page-image chunk, so the same
`(file, page)` can match twice. To keep results and citations stable across
implementations, retrieval MUST collapse candidates by `(rel_path, page)`
**before** truncating to `k` (and before any rerank), keeping the
highest-scoring representation for that page — i.e. **at most one hit per
`(file, page)`**. Non-paged media (audio/video windows, standalone images)
dedup by their own `(rel_path, span)` identity. This makes `augment` recall
strictly ≥ text-only without inflating top-k with duplicate pages.

### 7.2 Citations and inspection
Media hits cite the file + persisted span (§6): standalone image → `page`
(page 1), PDF page → `page`/`region`, audio/video → `time` (window range).

Inspection has a real constraint: `open_file` returns **text** — file lines,
OCR markdown, or a transcript — and MUST NOT emit raw binary through
`content` (§15.4; the `document` variant signals "content is the full
OCR/transcript representation", §15.1.1). So:
- In **`augment`**, a media hit still has its OCR/transcript representation,
  and `open_file` returns that text (unchanged behavior) alongside the span.
- In **`replace`**, a media-only chunk has *no* text representation;
  `open_file` therefore has nothing textual to return for it. This is a
  **permanent, non-retryable** condition, so it MUST surface as a distinct
  non-retryable outcome (proposed `MEDIA_NO_TEXT`) — **not** `OCR_NOT_READY`,
  whose retryable semantics would drive clients into useless retry loops —
  and never as raw binary. Returning the media itself for inspection would
  require a **new or extended tool** (e.g. a media-fetch surface) — called
  out as an open question (§10), not assumed here. Same root cause as the
  `ask` wrinkle (§7.3).

### 7.3 `ask` (RAG) over media — design risk
`ask` generates an answer from retrieved context. A **media-only** chunk
(`replace` mode, no OCR text) has nothing textual to feed a text chat model.
Options:
1. **`augment`** keeps OCR/STT text, so media hits still carry text for
   grounding — the safe default for `ask`-heavy corpora.
2. A **multimodal chat model** could receive the media part directly for
   grounding — out of scope for v1 (separate `chat` capability work).
3. **Cite-without-quote** — a media-only hit contributes a citation but no
   quoted context.

v1 recommendation: `search` is fully cross-modal; `ask` grounds on available
text (so `replace` mode degrades `ask` to cite-without-quote for media-only
hits, documented and surfaced, not silent). Multimodal `ask` grounding is a
follow-up.

### 7.4 Multimodal queries (deferred)
Image/audio-as-query is possible (shared space) but the MCP tools take text
queries today; deferred to a later slice.

## 8. Proposed spec changes (promote to SPEC.md at GA)

To be applied as spec `0.13.0` alongside the code:

- **New §8.1.7 Multimodal embeddings** — `gemini-embedding-2`, the §3
  modalities/limits, the `model.embed.multimodal` (`off|augment|replace`)
  config, the single-shared-space constraint (§4), reindex-bound mode, and
  `taskType` applying across modalities.
- **§8.1.4** — embed identity gains `multimodal_mode` (alongside the
  existing `embed.text_model`/`embed.code_model`/dims).
- **§16.2 config template** — `model.embed.multimodal` (default `off`).
- **§7.4.B / §5.4** — media chunks are a representation whose persisted
  provenance reuses the existing `span_kind ∈ {lines, page, time, region}`;
  no new persisted span kind (image → `page` 1; audio/video → `time`).
- **§9/§15** — results may include media-backed hits; the `ask` grounding
  rule (§7.3) documented. Inspecting a media-only chunk needs a new/extended
  tool because `open_file` returns text only (§7.2/§15.4) — flagged as the
  one potentially new surface, resolved at GA.
- `MINOR` bump (`0.13.0`) per the pre-1.0 policy — new optional config +
  provider behavior; the §8.1.2 matrix is unchanged (embed already `✅`;
  multimodality is a property of the model, not a new capability cell).

## 9. Phasing (code, at GA)

1. **Adapter** — add `gemini-embedding-2`; send media parts for non-text
   input (≈ done via #222/#223; mostly model-id + part-type wiring).
2. **Ingestion** — media-chunk units + per-modality chunking; ship `off`
   (no-op) → `augment` first.
3. **Store** — `modality`/`media_ref`/`span` columns (additive migration);
   embed-identity `multimodal_mode`.
4. **Retrieval/citations** — surface media hits with correct spans.
5. **`ask` grounding policy** (§7.3).
6. **`replace` mode** + the full config matrix + validation.

## 10. Risks / open questions

- **Public Preview churn** — the reason code is GA-gated; the §3 limits and
  field shapes may shift.
- **`ask`-over-media grounding** (§7.3) — the main unresolved UX question.
- **Inspecting media-only chunks** (§7.2) — `open_file` returns text only
  (§15.4), so a `replace`-mode media chunk can be cited but not opened.
  Decide at GA whether to add an extended media-fetch tool or restrict
  `replace` to corpora that don't need inspection.
- **Cost / storage** — `augment` adds media embeddings on top of text;
  long media multiplies windows. Per-corpus opt-in mitigates.
- **PDF double-representation** — docling structured text (0.9.0) and direct
  page-image embedding coexist in `augment`. v1 resolves double-counting
  with the deterministic `(file, page)` collapse rule in §7.1; remaining
  tuning (how `region` sub-page citations interact with a page-image hit) is
  the residual question.
- **Per-request modality caps** (≤ 6 images, ≤ 6 PDF pages, 120 s/180 s) →
  batching/windowing logic with precise span attribution.
- **Provider lock-in** — `augment`/`replace` force the whole corpus onto
  `gemini-embedding-2`; an operator wanting a different text embedder must
  use `off`.
