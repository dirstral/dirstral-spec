# td-002: Multimodal media chunks

- **ID:** td-002
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §8.1.7

## Scope

This document specifies **optional multimodal embedding** of media files: how a
PDF, image, audio, or video file is windowed into **media chunks** and embedded
directly into the corpus vector space, and how those chunks are persisted,
retrieved, grounded, and inspected. It applies only when a **natively
multimodal** embedding model is in use; the text-only default is unaffected.

Some embedding models are natively multimodal — they map text and media
(images, audio, video, PDFs) into one **shared** vector space, so a text query
can retrieve a media chunk and vice versa. The reference multimodal model is
Google **`gemini-embedding-2`** (native surface `models/{model}:embedContent` /
`:batchEmbedContents`). Design rationale and phasing:
[Design 0003](../../design/0003-multimodal-embeddings.md).

The provider/profile machinery, capability matrix, embed identity, asymmetric
input role, and Matryoshka dimensionality referenced below live in
[td-001](td-001-provider-model.md). Persisted spans and the `chunks`/`spans` tables
are [df-003](../data-formats/df-003-sqlite-schema.md); the client-facing `Span`
shape is [df-005](../data-formats/df-005-span.md); the retrieved `Hit` shape
(with its `modality`/`media_ref` fields) is
[df-006](../data-formats/df-006-hit-citation.md).

> **Preview caveat.** `gemini-embedding-2` is Public Preview; the limits and
> formats below are from preview docs and MUST be re-verified against the
> current provider docs before any implementation releases against them.

## Specification (normative)

### Per-request limits

Per-request limits (preview): text ≤ 8192 tokens; images ≤ 6 (PNG, JPEG, WebP,
BMP, HEIC, HEIF, AVIF); video ≤ 120 s (MP4, MOV); audio ≤ 180 s (MP3, WAV); PDF
1 file ≤ 6 pages. All modalities share one **unified 8192-token window**, so
chunking MUST budget the *combined* request, not just the per-modality caps.
Output is 3072-dim with Matryoshka truncation ([td-001](td-001-provider-model.md));
the input role / `taskType` ([td-001](td-001-provider-model.md)) applies across all
modalities.

### Media chunking (windowing)

A media file is chunked into one or more media chunks before embedding, each
chunk sized to fit one embed request:

- A standalone **image** is one chunk (`page` 1). A **PDF** is one chunk per page
  (`page` span); one page per request stays within the per-modality page cap
  (≤ 6 pages). Per-page token cost still counts against the unified 8192-token
  budget like any other modality.
- **Audio** and **video** are chunked into **non-overlapping, contiguous time
  windows** covering the whole file; each window is one media chunk with a
  `time` span (`start_ms`/`end_ms`;
  [df-003 §5.4](../data-formats/df-003-sqlite-schema.md) at rest,
  [df-005](../data-formats/df-005-span.md) on the wire). Each window MUST respect
  **both** the per-modality duration cap (audio ≤ 180 s, video ≤ 120 s) **and**
  the unified 8192-token budget; implementations SHOULD use conservative default
  window lengths at or below the caps and MAY make them configurable. A file of
  duration *D* with window length *W* yields ⌈*D*/*W*⌉ windows, the last being
  the remainder.
- **Video has no default text representation** (there is no video→text path
  analogous to audio STT, [td-004](td-004-representation-extraction.md) /
  [td-003](td-003-transcription-translation-subtitles.md)). It is therefore searchable **only** via
  its media windows: under `off` a video produces no chunks; under `augment` and
  `replace` it is represented solely by its `time`-windowed media chunks. Audio
  retains its transcript path ([td-003](td-003-transcription-translation-subtitles.md)) in
  `off`/`augment` as before.
- Windowing MUST be **deterministic** — the same file produces the same window
  boundaries on every (re)index — so `time`-span citations are stable.
- The ingester determines media duration. A file whose duration cannot be
  determined is **not** directly embedded; the condition is treated as a
  non-fatal per-document error (per-document error handling,
  [bs-002](../behavior/bs-002-ingestion-pipeline.md)) and a warning SHOULD be
  emitted. For modalities that have a text path (image/PDF OCR, audio
  transcript), that text representation is retained **even under `replace`**, so
  the file stays searchable; a video, which has no text path, is left unindexed.
  (This same text-path-retained fallback applies when a PDF's page count cannot
  be read.)

### `model.embed.multimodal` mode

- **`model.embed.multimodal`** is a tri-state per-corpus knob
  (config; [bs-011](../behavior/bs-011-configuration.md)):
  - `off` (default) — text-only; current behavior; **any** embed provider.
  - `augment` — keep text extraction + text embeddings **and** additionally
    embed media files directly; both are indexed.
  - `replace` — embed media files directly **instead of** OCR/STT→text; text
    files are unchanged.
- **Single shared space (per the embed-binding invariant,
  [td-001](td-001-provider-model.md)).** When `multimodal` is `augment` or `replace`,
  the **entire** embed binding MUST resolve to the multimodal model on `gemini`:
  `embed.provider: gemini` **and both** `embed.text_model` **and**
  `embed.code_model` set to `gemini-embedding-2` (the code axis is not exempt —
  leaving it on a different model would mix incomparable vectors in one index).
  Any other binding is `CONFIG_INVALID`
  ([df-008](../data-formats/df-008-error-taxonomy.md)). `off` keeps full provider
  freedom.
- **Reindex-bound.** The multimodal mode is part of the embed identity
  ([td-001](td-001-provider-model.md)); switching `off`↔`augment`↔`replace` requires a
  reindex.

### Provenance

A media chunk is a representation
([td-004](td-004-representation-extraction.md)) whose persisted span reuses the
existing `span_kind ∈ {lines, page, time, region}`
([df-003 §5.4](../data-formats/df-003-sqlite-schema.md)) — **no new persisted
kind**: a standalone image → `page` 1, audio/video windows → `time`, PDF pages →
`page`/`region`. (`document`, [df-005](../data-formats/df-005-span.md), remains a
client-facing `open_file`-only variant, not persisted.)

### Retrieval

A text query embeds via the model's text path and retrieves any chunk in the
shared space, including media; a retrieved media chunk surfaces as a
[df-006](../data-formats/df-006-hit-citation.md) `Hit` carrying the optional
`modality` (e.g. `image`/`audio`/`video`) and `media_ref` fields. In `augment`,
a PDF page may carry several docling text/region chunks
([td-004](td-004-representation-extraction.md)) **and** one coarse page-image chunk;
to avoid double-counting, retrieval MUST drop a page-image candidate for
`(rel_path, page)` only when a text/region candidate for that same page survives,
**before** truncation/rerank — distinct text/region chunks are never collapsed
into each other. (Retrieval routing and result structure:
[bs-003](../behavior/bs-003-retrieval-and-answer.md).)

### `ask` over media

Generation grounds on available text: in `augment` the media hit's OCR/transcript
text grounds the answer; a `replace`-mode media-only hit (no text) is cited
without quoted context. (Multimodal answer grounding is a later concern. Answer
generation: [bs-003](../behavior/bs-003-retrieval-and-answer.md).)

### Inspection

`open_file` returns text only
([bs-007](../behavior/bs-007-tool-specifications.md)); a `replace`-mode
media-only chunk has no text representation, a **permanent** condition, so
`open_file` MUST return the non-retryable `MEDIA_NO_TEXT`
([df-008](../data-formats/df-008-error-taxonomy.md)) — never raw binary and never
the retryable `OCR_NOT_READY`. The media **bytes** themselves are served only by
`dir2mcp_open_media_clip` ([bs-007](../behavior/bs-007-tool-specifications.md)),
whose own bounds/extraction failures use the `CLIP_TOO_LARGE` /
`MEDIA_CLIP_FAILED` codes ([df-008](../data-formats/df-008-error-taxonomy.md)).

### Capability matrix

The capability matrix ([td-001](td-001-provider-model.md)) is unchanged:
multimodality is a property of the chosen embed model, not a new capability cell.

## Changelog

- **0.1.0** — Migrated from SPEC.md §8.1.7 (Multimodal embeddings). Cross-references
  rewired to stable doc IDs: `§5`/`§5.4` → df-003; `§7.4`/`§7.4.B`/`§7.4.C` →
  td-004 (audio STT path also noted in td-003); per-document error handling
  (`§7.7`) and chunking (`§7.5`) → bs-002; `§8.1.2`/`§8.1.4`/`§8.1.5`/`§8.1.6`
  (capability matrix, embed identity, input role, Matryoshka) → td-001; `§8.6`
  STT/transcript → td-003; retrieval/`ask` (`§9`) → bs-003; `§14.2`
  (`MEDIA_NO_TEXT`, `OCR_NOT_READY`, `CONFIG_INVALID`) → df-008; `§15.1.1`
  (`document` span) → df-005; `§15.4` `open_file` / `open_media_clip` tools →
  bs-007; config knob (`§16`) → bs-011. Made explicit that a retrieved media
  chunk surfaces as a df-006 `Hit` via its optional `modality`/`media_ref`
  fields, and that media bytes are served out of band by `open_media_clip`
  (bs-007) whose `CLIP_TOO_LARGE`/`MEDIA_CLIP_FAILED` errors live in df-008 —
  both faithful to the source, which already required the `MEDIA_NO_TEXT`
  behavior and the shared-space retrieval of media chunks.
