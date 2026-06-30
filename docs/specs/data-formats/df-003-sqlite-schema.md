# df-003: SQLite metadata schema

- **ID:** df-003
- **Version:** 0.1.0
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
- `rep_type` (`raw_text | extracted_markdown | transcript | annotation_text | annotation_json`)
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
- `deleted` (boolean; tombstone)

> `embedding_status` is the retrieval-eligibility gate: a chunk with status
> `error` MUST be excluded from BM25/lexical results as well as vector results
> (dir2mcp #443), and the embed worker's transient-vs-permanent classification
> governs whether a failed chunk stays `pending` or becomes `error`
> (dir2mcp #412).

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
- `embed_text_model`, `embed_text_dim`
- `embed_code_model`, `embed_code_dim`
- `ocr_model`
- `stt_provider`, `stt_model`
- `chat_model`

## Changelog

- **0.1.0** — Migrated from SPEC.md §5. Cross-references rewired to doc IDs
  (bs-002/bs-003/bs-008, td-001/td-003/td-004, df-000/df-005/df-006/df-008).
  Added the `embedding_status` retrieval-eligibility note (dir2mcp #443/#412) and
  tied `index_format_version` to the df-000 version fence (dir2mcp #405).
