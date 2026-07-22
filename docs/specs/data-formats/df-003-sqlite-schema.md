# df-003: SQLite metadata schema

- **ID:** df-003
- **Version:** 0.4.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §5 (migrated; minimum semantics)

## Scope

The metadata store `meta.sqlite` (df-002 layout): the `documents`,
`representations`, `chunks`, `spans`, and `settings` tables. The exact SQL types
MAY vary; the **semantics below are normative**. The `chunks.chunk_id` integer is
the same identifier surfaced in [df-006](df-006-hit-citation.md) and used as the
ANN label (bs-008); the `spans` row is the at-rest form of the
[df-005](df-005-span.md) `Span`.

## Specification (normative)

### 5.1 `documents`

- `doc_id` (PK)
- `rel_path` (unique, normalized `/`)
- `source_type` (`file | archive_member`)
- `doc_type` (`code | text | md | pdf | image | audio | video | data | html | archive | binary_ignored | …`)
- `size_bytes`
- `mtime_unix`
- `content_hash` (stable, e.g. blake3/sha256)
- `status` (`ok | skipped | error`)
- `error` (nullable)
- `deleted` (boolean; tombstone)
- `canonical_doc_id` (optional; `0`/self when canonical, otherwise the `doc_id`
  of the canonical document this row is an **alias** of — bs-002 dedup)
- `is_alias` (optional boolean; `true` for a non-canonical member of a duplicate
  group). Alias rows share the canonical `content_hash` and hold **no**
  representations, chunks, or embeddings.

### 5.2 `representations`

- `rep_id` (PK)
- `doc_id` (FK)
- `rep_type` (`raw_text | extracted_markdown | transcript | annotation_text | annotation_json | summary`)
- `rep_hash` (stable; changes when the rep changes)
- `created_unix`
- `meta_json` (MUST include provider/model for OCR/transcription/annotation when applicable)
- `deleted` (boolean; tombstone)

**Transcript `meta_json`** — `provider` (string; the enumeration is **not closed**
to `mistral|elevenlabs` — any STT-capable provider per td-001 is valid), `model`,
optional `model_version` (part of the derivation identity, td-003), `timestamps`
(bool), optional `timing` (`provider` default | `estimated` fallback), optional
`words` (per-word timing captured), optional `language` (source language),
optional `source` (`stt` | `sidecar`), optional `duration_ms`.

A **translated** transcript also records `source_language`, `translate_provider`,
`translate_model`. A **diarized** transcript (td-003) also records `diarized`
(bool), optional `diarize_provider` / `diarize_model` (part of the derivation
identity), and optional `speakers` (the distinct speaker ids, each optionally
with a `label`; per-segment attribution lives on the span's
`extra_json.speaker`).

**Summary `meta_json`** (hierarchical retrieval; SPEC §5.2/§9.7, td-003 §8.6.7). A
`summary` representation is a model-generated coarse view (`index_kind=text`)
embedded and BM25-indexed **alongside** the fine chunks of the same document, in the
**same** embedding space — an **additive** representation, **not** an embed-identity
change (bs-008 §8.1.4). Opt-in and off by default. Its `meta_json` records
`summary_level` (`document` | `section`), `provider` / `model` / optional
`model_version` (the chat/annotator generator identity), `prompt_version` (built-in
template tag), optional `prompt_hash` (stable hash of the **effective** prompt when
an operator `prompt` override is set — part of the derivation identity so an edited
override re-derives), and `coverage` — the parent→child linkage. `coverage` MUST
name **exactly one** source representation and a range within it:
`source_rep_id` (the `rep_id` whose chunks this summary summarizes — a summary
covers **one** representation, never a mix, so a multi-representation document gets a
distinct summary per summarized representation) plus a `range` that is **inclusive**
on both bounds — `{ "kind": "document" }` (every non-deleted chunk of
`source_rep_id`), `{ "kind": "ordinals", "start", "end" }` (a chunk-`ordinal` range,
§5.3), or `{ "kind": "time", "start_ms", "end_ms" }` (transcript segments / clips).
For a `section` summary `meta_json` also records the windowing inputs
(`section_units` **or** `section_seconds` + the underlying chunking/segmentation
identity). Expansion (SPEC §9.7) resolves a summary to the chunks of
`source_rep_id` whose `ordinal`/`time` falls in `range` — a **deterministic key**,
not a vector match — so `section` summaries over the same `source_rep_id` MUST
**tile without overlap** (each fine chunk maps to exactly one). A `summary` is
`index_kind=text` and lives on the **text** logical axis (SPEC §6.1): the coarse
match runs in text search while expansion crosses to the covered chunks by identity,
so a summary over a `code`-indexed representation still resolves to its `code`
chunks. The summary derivation identity (covered `source_rep_id` content +
generator/effective-prompt + section windowing inputs) re-derives **text and child
linkage** on change (td-003 §8.6.7). A `summary` is model-generated prose, **never**
a citation snippet (SPEC §9.7).

**Detected-language metadata (any representation).** Any representation MAY record
the natural language of its content in `meta_json`, independent of rep type, to
enable multilingual filtering and per-language retrieval (bs-003). Fields are
**optional and additive**; a representation that records none is **unknown
language** (never an error); detection is **best-effort** and MUST degrade
gracefully (td-001/td-003):

- `language` — the **effective** language as a BCP-47 tag (e.g. `en`, `pt-BR`);
  the value matched by the retrieval language filter. Absent ⇒ unknown.
- `language_source` — how `language` was obtained: `detected` (best-effort),
  `configured` (operator pin), or `declared` (asserted by the source, e.g. a
  sidecar suffix or an OCR provider's reported language). Absent ⇒ unspecified.
- `language_confidence` — detector confidence in `[0,1]` for a `detected`
  language; informational only and MUST NOT by itself mark a representation
  unknown.

When a configured (expected) language and a detected language disagree, the
recorded `language` is the **effective** value indexed under, and
`language_source` records which won. A translated transcript's `language` is its
**target** language; `source_language` records what it was translated *from* —
both are matchable per-language values (bs-003).

### 5.3 `chunks`

- `chunk_id` (PK; integer; used as the ANN label)
- `rep_id` (FK)
- `ordinal`
- `text` (or compressed blob)
- `text_hash`
- `tokens_est` (approx)
- `index_kind` (`text | code`) — routes to `vectors_text` or `vectors_code`
- `embedding_status` (`ok | pending | error`)
- `embedding_error` (nullable)
- `chunk_context` (nullable) — the generated document-aware context string for
  contextual retrieval (SPEC §8.1.8); prepended to the **embed** input only
  (`context + "\n\n" + text`), **never** to `text`. NULL when contextual
  retrieval is off, or when the chunk fell back to raw embedding.
- `embedding_mode` (`disabled | contextualized | fallback`) — per-chunk
  contextualization state (SPEC §8.1.8). Disambiguates a NULL `chunk_context`
  (feature off vs. context generated vs. generation failed → embedded raw). The
  re-embed gate reads it to retry `fallback` chunks while contextualization is on,
  and to drive honest coverage. **Not** part of the embed identity (SPEC §8.1.4).
- `deleted` (boolean; tombstone)

> `embedding_status` is the retrieval-eligibility gate: a chunk with status
> `error` MUST be excluded from BM25/lexical results as well as vector results
> (dir2mcp #443), and the embed worker's transient-vs-permanent classification
> governs whether a failed chunk stays `pending` or becomes `error`
> (dir2mcp #412).

> `chunk_context` and `embedding_mode` are **additive** columns for contextual
> retrieval (SPEC §8.1.8; dir2mcp #330). `text` (the persisted, displayed, and
> **cited** chunk text) is **never** the contextualized text: the context lives
> only in `chunk_context` and in the transient embed input, preserving citation
> faithfulness (#403).
>
> **Migration (in-place, no re-embed).** These columns bump this document's schema
> version, and adding them advances the on-disk schema/`index_format_version` fence
> (§5.5 / [df-000](df-000-base.md)). Because they are purely additive and
> back-compatible, an implementation MUST perform this as an **in-place additive
> migration**, in order: (1) **accept** a database at the immediately-prior schema
> version — it MUST NOT reject it with `INDEX_VERSION_MISMATCH`
> ([df-008](df-008-error-taxonomy.md)); (2) **add** the nullable `chunk_context`
> and `embedding_mode` columns and the `embed_contextual` `settings` key (§5.5),
> **backfilling** the pre-feature defaults (`chunk_context = NULL`,
> `embedding_mode = disabled`, `embed_contextual = off`); (3) **advance** the
> `index_format_version` / `PRAGMA user_version` fence to the new value; and
> (4) **preserve all existing vectors** — the embed identity now reads `…|off`
> (SPEC §8.1.4), which compares equal to what the corpus already recorded, so
> nothing re-embeds. `INDEX_VERSION_MISMATCH` is reserved for a version the
> implementation has **no** migration path for, not for this additive step.

### 5.4 `spans` (provenance for citations)

- `chunk_id` (FK)
- `span_kind` (`lines | page | time | region`)
- `start` (integer) — `start_line` / `page` / `start_ms` / first page (region)
- `end` (integer) — `end_line` / `page` / `end_ms` / last page (region)
- `extra_json` (nullable) — speaker, confidence, section breadcrumb, bbox, etc.

This is the at-rest form of the [df-005](df-005-span.md) `Span`. The `document`
span kind (df-005) has no row here — it is synthesized by `open_file` when no
finer provenance exists.

For `time` spans on a **diarized** transcript (td-003), `extra_json` MAY carry a
`speaker` (stable per-transcript id) and `speaker_label`; both are
optional/additive — consumers that don't recognize them MUST degrade to a flat
transcript citation. Diarization is **off by default** and provider-dependent.

For `region` spans (structured extraction, td-004), `start`/`end` carry the first
and last page (equal when single-page), and `extra_json` **MUST** carry the
bounding box and **SHOULD** carry the section breadcrumb:

```jsonc
{
  "bbox": { "page": 1, "l": 72.0, "t": 90.5, "r": 523.0, "b": 410.2, "coord_origin": "TOPLEFT" },
  "section": ["Chapter 2", "2.1 Background"],   // heading breadcrumb, outermost first
  "label": "paragraph",                          // a single value (enum below)
  "charspan": [120, 884]                          // optional char offsets into the source element
}
```

- `label` is a **single** discrete value (not pipe-delimited), one of:
  `paragraph`, `section_header`, `list_item`, `table`, `caption`, `code`,
  `formula`, `picture`. For a mixed-label chunk, the dominant (first/longest)
  element's label is used.
- `bbox` coordinates are in the source's point space; `coord_origin` is
  `TOPLEFT` or `BOTTOMLEFT`; implementations SHOULD normalize to `TOPLEFT` and
  record the origin stored.
- `bbox.page` is the **primary page** (first source element in reading order) and
  MUST satisfy `start ≤ bbox.page ≤ end` (single-page ⇒ `start == end == bbox.page`).
- For a multi-element chunk, `bbox` is the union of only the elements **on the
  primary page**; a single bounding box never spans pages.

### 5.5 `settings`

`key`/`value` for at least:

- `protocol_version` = `2025-11-25`
- `corpus_id`
- `index_format_version` — the schema/index version fence; see
  [df-000](df-000-base.md) (also surfaced as `PRAGMA user_version`) and the
  `INDEX_VERSION_MISMATCH` code ([df-008](df-008-error-taxonomy.md)).
- `embed_provider`, `embed_base_url` — `embed_base_url` normalized per
  SPEC §8.1.4 / td-001; `""` is a valid value (pre-existing indexes,
  non-meaningful/default endpoints)
- `embed_text_model`, `embed_text_dim`
- `embed_code_model`, `embed_code_dim`
- `embed_contextual` — the `contextual` component of the embed identity (SPEC
  §8.1.4 / §8.1.8): `off` when contextual retrieval is disabled or falls open to
  off, else `ctx:<hash>` over the canonical generator identity (provider +
  normalized endpoint, model, max_tokens, prompt_version, effective prompt).
  Absent on a pre-feature index ⇒ treated as `off`.
- `ocr_model`
- `stt_provider`, `stt_model`
- `chat_model`

## Changelog

- **0.4.0** — Added the `summary` `rep_type` and its `meta_json` (hierarchical /
  multi-resolution retrieval, SPEC §5.2/§9.7, td-003 §8.6.7): `summary_level`,
  generator identity (`provider`/`model`/`model_version`), `prompt_version` +
  optional `prompt_hash`, and the `coverage` parent→child linkage
  (`source_rep_id` + an inclusive `document`/`ordinals`/`time` range that names the
  single covered representation), plus the section-windowing derivation inputs and
  the tile-without-overlap rule. A summary is text-axis (§6.1) with identity-based
  expansion, so it works over `code`-indexed representations too. Additive and off
  by default; not an embed-identity change (bs-008 §8.1.4). Unblocks dir2mcp #329.
- **0.3.0** — Added the additive `chunks.chunk_context` (nullable) and
  `chunks.embedding_mode` (`disabled|contextualized|fallback`) columns and the
  `embed_contextual` persisted `settings` key for contextual retrieval (SPEC
  §8.1.8 / §8.1.4; dir2mcp #330). `chunk_context`/embed input never enter `text`
  (citation faithfulness, #443/#403); `embedding_mode` is per-chunk state, not
  part of the embed identity. Backward-compatible additive migration: a
  pre-feature index accepts the new nullable columns / `embed_contextual` key,
  backfills their pre-feature defaults (`embedding_mode = disabled`,
  `chunk_context = NULL`, `embed_contextual = off`) in place, advances the schema
  fence without an `INDEX_VERSION_MISMATCH` rejection, and **preserves existing
  vectors** — the embed identity still compares equal (`…|off`), so nothing
  re-embeds. Unblocks dir2mcp #330.
- **0.2.0** — Added the `embed_provider` and `embed_base_url` persisted
  `settings` keys mirroring the embed-identity amendment in SPEC §8.1.4 /
  td-001 §8.1.4 and the §6.4 / bs-008 tuple. `embed_base_url` is normalized per
  td-001 §8.1.4 and `""` is a valid value (pre-existing indexes and
  non-meaningful/default endpoints stay valid on reload). Unblocks dir2mcp #560.
- **0.1.0** — Migrated from SPEC.md §5. Cross-references rewired to doc IDs
  (bs-002/bs-003/bs-008, td-001/td-003/td-004, df-000/df-005/df-006/df-008).
  Added the `embedding_status` retrieval-eligibility note (dir2mcp #443/#412) and
  tied `index_format_version` to the df-000 version fence (dir2mcp #405).
