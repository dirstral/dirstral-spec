# bs-007: Tool specifications (behavior)

- **ID:** bs-007
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §15.2–§15.11 (behavior; schemas → df-007)

## Scope

The per-tool **behavioral** contract for every MCP tool dir2mcp exposes:
purpose, parameter semantics (meaning, defaults, validation ranges), runtime
behavior (pagination, partial-index reporting, selection/mode rules, clip
bounds), and the error conditions each tool returns.

The transport-level tool contract (`tools/list`, `tools/call`, the
`content[]`/`structuredContent`/`isError` result shape, tool tiers) is in
[bs-006](bs-006-mcp-tools-list-call.md). The canonical machine-readable
`inputSchema`/`outputSchema` for each tool — the source of truth for field
names, types, and `required` sets — is in
[df-007](../data-formats/df-007-tool-schemas.md) and the
`spec/tools/schemas/*.json` files; this document does **not** reproduce those
JSON schemas. Shared result shapes are normative elsewhere: the
[df-005 `Span`](../data-formats/df-005-span.md) and the
[df-006 `Hit`/`Citation`](../data-formats/df-006-hit-citation.md). Canonical
error codes are in [df-008](../data-formats/df-008-error-taxonomy.md).

## Specification (normative)

Every tool returns errors as an `isError` tool result whose
`structuredContent.error.code` is a canonical
[df-008](../data-formats/df-008-error-taxonomy.md) code (see bs-006). The codes
called out below are the conditions specific to each tool; the auth/transport,
input-validation, and provider codes in df-008 apply across all tools.

### dir2mcp_search

**Purpose.** Semantic retrieval across indexed content. Schema: `search.json`
(df-007); `hits[]` items are [df-006 `Hit`](../data-formats/df-006-hit-citation.md).

**Parameters.**

- `query` (**required**, non-empty) — the retrieval query.
- `k` — number of hits, integer in `[1, 50]`, default **15**.
- `index` — which vector space to search: `auto | text | code | both`, default
  **`auto`**. The resolved space is reported back as `index_used`
  (`text | code | both`).
- `path_prefix` — restrict to documents under this path prefix.
- `file_glob` — restrict to documents matching this glob.
- `doc_types` — array of doc-type strings to restrict to.
- `speaker` — optional (td-003): restrict time-spanned transcript hits to this
  speaker id. A corpus **without** diarized transcripts returns **no**
  speaker-filtered hits.
- `languages` — optional (bs-003): restrict hits to representations recorded in
  any of these BCP-47 languages (case-insensitive **primary-subtag** match).
  Absent/empty ⇒ no filtering. **Unknown-language** representations never match
  a specific filter.

**Behavior.** Output carries `query`, `k`, `index_used`, `hits[]`, and
`indexing_complete` (the latter three plus `query` are required).
`indexing_complete` reflects the partial-index state — results MAY be returned
while indexing is still in progress, with `indexing_complete: false`. The
result `content[]` MUST include at least one `text` item summarizing results
(top hits + citations).

### dir2mcp_ask

**Purpose.** RAG answer with citations; can also run search-only. Schema:
`ask.json` (df-007); `citations[]` are [df-006 `Citation`](../data-formats/df-006-hit-citation.md),
`hits[]` are df-006 `Hit`.

**Parameters.**

- `question` (**required**, non-empty).
- `k` — integer in `[1, 50]`, default **15**.
- `mode` — `answer | search_only`, default **`answer`**. In `search_only` no
  answer is generated; only retrieval results are returned.
- `index` — `auto | text | code | both`, default **`auto`**.
- `path_prefix`, `file_glob`, `doc_types` — same scoping semantics as
  `dir2mcp_search`.
- `languages` — optional (bs-003): restrict retrieved **contexts** to
  representations in any of these BCP-47 languages (case-insensitive
  primary-subtag match). Absent/empty ⇒ no filtering; unknown-language
  representations never match a specific filter.

**Behavior.** Output carries `question`, `answer`, `citations[]`, `hits[]`, and
`indexing_complete` (`question`, `citations`, `hits`, `indexing_complete` are
required). A `Citation` is lean — `chunk_id` + `rel_path` + `span` (df-006); the
cited text is resolved via `dir2mcp_open_file` or the matching `hits[]` entry.
The result `content[]` MUST include a `text` item containing the final answer
(when `mode=answer` **and** answer generation is enabled) with inline citations.

### dir2mcp_open_file

**Purpose.** Open an exact source slice for verification (lines / page / time).
Schema: `open_file.json` (df-007); returns `content` plus a
[df-005 `Span`](../data-formats/df-005-span.md) (including the `document`
variant).

**Exclusion-engine precondition (normative).** Before reading or returning any
data, the server MUST run the requested `rel_path` and any extracted content
through the configured exclusion engine (pattern matcher + path excludes). If a
match occurs the tool **MUST NOT** return the secret content; it MUST either
return an error (e.g. `FORBIDDEN`) or an empty/plain-text placeholder. This
makes tool-level bypass of ingestion filters impossible.

**Parameters.**

- `rel_path` (**required**, non-empty).
- `start_line` / `end_line` — integers `≥ 1`.
- `page` — integer `≥ 1`.
- `start_ms` / `end_ms` — integers `≥ 0`.
- `max_chars` — integer in `[200, 50000]`, default **20000**.

**Selection rules (priority order).**

1. If `page` is provided → return OCR page text (if available; else
   `DOC_TYPE_UNSUPPORTED`).
2. Else if `start_ms`/`end_ms` provided → return the transcript excerpt (if
   available).
3. Else if `start_line`/`end_line` provided → return file lines.
4. Else (default, no span argument):
   - text/code/markdown/html → return the first `max_chars` of the file with
     **no** `span` set;
   - PDF → return the cached full-document OCR markdown with
     `span.kind="document"`; if the OCR cache is not yet populated (e.g. ingest
     still running) the tool MUST return `OCR_NOT_READY` rather than the raw
     bytes;
   - audio → return the cached full-document transcript with
     `span.kind="document"`; same `OCR_NOT_READY` semantics when no transcript
     exists yet;
   - a `replace`-mode multimodal media chunk (td-002) with **no** text
     representation → return the **non-retryable** `MEDIA_NO_TEXT` (the absence
     is permanent — distinct from the retryable `OCR_NOT_READY`), never the raw
     media bytes.

**Behavior.** The handler MUST NOT emit raw binary bytes through
`content[].text` — that field is documented as text. PDFs and audio without a
span argument resolve through the OCR / transcript cache, never through a direct
file read. Output carries `rel_path`, `doc_type`, optional `span`, `content`,
and `truncated` (`rel_path`, `doc_type`, `content`, `truncated` required).

**Errors:** `DOC_TYPE_UNSUPPORTED`, `OCR_NOT_READY`, `MEDIA_NO_TEXT`,
`FORBIDDEN` (df-008).

### dir2mcp_list_files

**Purpose.** List files under root for navigation and filter selection. Schema:
`list_files.json` (df-007).

**Parameters.**

- `path_prefix` — restrict to this path prefix.
- `glob` — restrict to matching files.
- `limit` — integer in `[1, 5000]`, default **200**.
- `offset` — integer `≥ 0`, default **0**.

**Behavior.** Pagination is via `limit`/`offset`; the output echoes both and
reports `total` (the full match count, independent of the page window). `files[]`
entries carry `rel_path`, `doc_type`, `size_bytes`, `mtime_unix`,
`status` (`ok | skipped | error`), and `deleted` (all required per entry). The
metadata mirrors the df-003 `documents` row.

### dir2mcp_stats

**Purpose.** Status / progress / health for indexing and models. Schema:
`stats.json` (df-007); SHOULD carry `format_version` (df-000).

**Parameters.** None (empty input object).

**Behavior.** Output carries `root`, `state_dir`, `protocol_version`,
`doc_counts` (map of doc-type → integer), `total_docs`, `doc_counts_available`,
`indexing`, and `models` (all required); `recent_failures` is optional.

- `indexing` carries `job_id`, `running`, `mode` (`incremental | full`),
  `scanned`, `indexed`, `skipped`, `deleted`, `representations`, `chunks_total`,
  `embedded_ok`, and `errors` (all required). `representations`, `chunks_total`,
  and `embedded_ok` have minimum **−1**: **−1** means "not derivable" (the
  ListFiles-only fallback path) and MUST be treated as **unavailable**, not as
  an error.
- `models` carries `embed_text`, `embed_code`, `ocr`, `stt_provider`,
  `stt_model`, `chat` (all required). `stt_provider` is **not** a closed enum —
  any STT-capable provider (e.g. `mistral | elevenlabs | openai | gemini |
  self-hosted`; see td-001) is valid.
- `recent_failures` (optional) lists up to `recent_failures_limit` (default
  **20**) of the most-recent documents with `status="error"`, newest first by
  `mtime_unix`. Each entry carries `rel_path`, `doc_type`, `mtime_unix`, and a
  short, sanitized `error_message`. Implementations MAY omit the field when no
  failures are recorded; clients MUST treat omission as "no recent failures",
  **not** "unsupported". `error_message` is single-line, length-capped
  (implementations SHOULD cap at 512 bytes on a UTF-8 rune boundary), with
  control characters stripped, and MUST NOT contain secrets or raw file content.

### dir2mcp_transcribe

(Recommended tool.) Schema: `transcribe.json` (df-007).

**Purpose.** Force transcription for an audio file, persist the transcript
representation, and (optionally) return segments.

**Parameters.**

- `rel_path` (**required**, non-empty) — the audio file.
- `language` — optional source-language hint.
- `timestamps` — boolean, default **true**.
- `retranscribe` — boolean, default **false** (when true, re-transcribe even if
  a transcript already exists).

**Behavior.** Output carries `rel_path`, `stt_provider`, `model`, `indexed`
(required), and optional `segments[]` (`start_ms`, `end_ms`, `text` per
segment). `stt_provider` is **not** a closed enum (any STT-capable provider; see
td-001).

### dir2mcp_annotate

(Recommended tool.) Schema: `annotate.json` (df-007).

**Purpose.** Run structured extraction on a document against a provided JSON
schema; store the JSON; optionally index the flattened text.

**Parameters.**

- `rel_path` (**required**, non-empty).
- `schema_json` (**required**) — the extraction JSON schema (object).
- `index_flattened_text` — boolean, default **true**.

**Behavior.** Output carries `rel_path`, `stored`, `flattened_indexed`,
`annotation_json` (required), and optional `annotation_text_preview`.
`flattened_indexed` reflects whether the flattened text was indexed (governed by
`index_flattened_text`).

### dir2mcp_transcribe_and_ask

(Recommended tool.) Schema: `transcribe_and_ask.json` (df-007).

**Purpose.** Ensure a transcript exists (transcribe if missing/stale), then
answer a question using the transcript (and optionally the whole corpus, if so
configured).

**Parameters.**

- `rel_path` (**required**, non-empty) — the audio file.
- `question` (**required**, non-empty).
- `k` — integer in `[1, 50]`, default **15**.

**Behavior.** Output is the same shape as `dir2mcp_ask` plus `stt_provider`,
`transcript_model`, and a `transcribed` boolean (whether transcription ran on
this call).

### dir2mcp_ask_audio

(Optional extension.) Schema: `ask_audio.json` (df-007).

**Purpose.** Same as `dir2mcp_ask` but additionally returns audio output (TTS).
Optional and additive.

**Parameters.** The input **inherits all** `dir2mcp_ask` fields (`question`,
`k`, `mode`, `index`, `path_prefix`, `file_glob`, `doc_types`) plus
audio-specific additive options:

- `voice_id` — optional.
- `format` — `mp3 | wav`, default **`mp3`**.

**Behavior.** The result `content[]` MUST include a `text` item for the answer
and an `audio` item with a base64 payload and `mimeType`.

### dir2mcp_open_media_clip

(Recommended tool. **Status: Planned** — returns the **actual audio/video
snippet** for a media search/ask hit (dir2mcp #264), rather than only a
`path@t=...` citation; additive, lands in a follow-up dir2mcp code PR.) Schema:
`open_media_clip.json` (df-007).

**Purpose.** Extract and return the media snippet for a transcript/media hit,
identified either by `chunk_id` (resolved to its source media + `time` span) or
by an explicit `rel_path` + `start_ms`/`end_ms` range.

**Relationship to `dir2mcp_open_file`.** It is the time-media analogue of
`open_file`: `open_file` with `start_ms`/`end_ms` on an audio document returns
the **transcript excerpt** (text); `open_media_clip` returns the **media bytes**
for the same span. The two share `time`-span semantics
([df-005](../data-formats/df-005-span.md)), so one hit can be cited, read as
text, and played.

**Parameters** (input requires `anyOf`: `chunk_id`, **or**
`rel_path`+`start_ms`+`end_ms`).

- `chunk_id` — integer; resolved to its source media (`rel_path` / media ref)
  and the chunk's `time` span.
- `rel_path` — non-empty; required (with `start_ms`/`end_ms`) when `chunk_id`
  is absent.
- `start_ms` / `end_ms` — integers `≥ 0`.
- `return` — `inline | reference`, default **`inline`**.

**Selection rules.**

- If `chunk_id` is provided, the server resolves it to its source media and the
  chunk's `time` span. Explicit `start_ms`/`end_ms` supplied **alongside**
  `chunk_id` **override** the chunk's span (still bounded to the same source
  media).
- Else `rel_path` plus `start_ms`/`end_ms` MUST be provided.
- The target document MUST be audio/video; a non-media `rel_path` returns
  `DOC_TYPE_UNSUPPORTED`. A missing source returns `FILE_NOT_FOUND`. A
  `start_ms >= end_ms` (or out-of-bounds) range returns `INVALID_RANGE`.

**Bounds (normative).** Implementations MUST enforce a **maximum clip duration**
(`media.clip.max_duration_ms`, default **120000** = 2 min) and a **maximum clip
byte size** (`media.clip.max_bytes`, default **25 MiB**) (bs-011). A request
whose span exceeds the duration bound, or whose extraction would exceed the byte
bound, returns the **non-retryable** `CLIP_TOO_LARGE`; the caller must request a
shorter span. Extraction failures (unreadable media, missing `ffmpeg`) return
`MEDIA_CLIP_FAILED` (df-008).

**Return shape.** The clip is returned in **one** of two modes selected by
`return` (default `inline`):

- `inline` — the clip is base64-encoded in the structured output (`data` +
  `mime_type`) and carried as an `audio`/`video`-typed `content[]` item. Inline
  return is subject to the byte bound above.
- `reference` — the clip is materialized to a short-lived, server-managed
  location and a `uri` (plus `expires_unix`) is returned instead of bytes, for
  clients that fetch out-of-band. Implementations that do not support
  `reference` MUST fall back to `inline` (and SHOULD note it), never error
  solely because `reference` was requested. For `return=reference` the
  `content[]` carries a text item with the `uri` and a `resource_link` where
  supported.

The handler MUST NOT emit raw binary bytes through a `text` content item (media
bytes travel only via `data`/`uri`). The exclusion-engine precondition and x402
request gating that apply to `open_file` apply equally to `open_media_clip`.

**Word-level deep-linking (optional refinement).** When the source transcript
carries per-word timing (td-003 `words`), an implementation MAY accept the same
`start_ms`/`end_ms` snapped to word boundaries for tighter clips; this is an
optional refinement and MUST NOT change the bounds or error semantics above.

**Output.** Carries `rel_path`, `doc_type`, `span`, `mime_type`, and `return`
(required), plus `duration_ms`, `size_bytes`, `data` (present when
`return=inline`), `uri` and `expires_unix` (present when `return=reference`).

**Errors:** `DOC_TYPE_UNSUPPORTED`, `FILE_NOT_FOUND`, `INVALID_RANGE`,
`CLIP_TOO_LARGE`, `MEDIA_CLIP_FAILED` (df-008).

## Changelog

- **0.1.0** — Migrated the **behavioral** semantics of all ten MCP tools from
  SPEC.md §15.2–§15.11 (`dir2mcp_search`, `_ask`, `_open_file`, `_list_files`,
  `_stats`, `_transcribe`, `_annotate`, `_transcribe_and_ask`, `_ask_audio`,
  `_open_media_clip`). Per the schema/behavior split, the raw JSON
  `inputSchema`/`outputSchema` blocks are **not** reproduced here — they remain
  canonical in [df-007](../data-formats/df-007-tool-schemas.md) and the
  `spec/tools/schemas/*.json` files; this document captures purpose, parameter
  semantics (defaults, ranges, validation), runtime behavior (pagination,
  partial-index reporting, `open_file` selection rules, `open_media_clip` modes
  and bounds), and per-tool error conditions. Cross-references rewired to stable
  doc IDs: §5 → df-003; §7 → bs-002; §8.1.7 → td-002; §8.6 → td-003; §9 →
  bs-003; §14 → df-008; §15.1.1 → df-005; §15.1.2 → df-006; §15 schemas →
  df-007; §16 → bs-011. The `§8.2` STT-provider reference was rewired to td-001
  (matching the df-003 precedent). Drift note: [bs-006](bs-006-mcp-tools-list-call.md)
  links to this document as `bs-007-tool-behavior.md`, but the migrated file is
  `bs-007-tool-specifications.md` — bs-006's link target is stale (not fixed
  here, per scope).
