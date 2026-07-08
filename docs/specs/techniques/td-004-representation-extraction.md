# td-004: Representation generation & structured extraction

- **ID:** td-004
- **Version:** 0.2.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §7.4

## Scope

How a classified document is turned into one or more **representations** — the
indexable text the rest of the pipeline chunks, embeds, and cites. This covers
the four representation paths: (A) code/text/markdown/data/html, (B)
PDF/image/document extraction (OCR and structured extraction via docling), (C)
audio transcription, and (D) on-demand structured annotations.

The persisted shapes referenced here are normative elsewhere and are **not**
redefined in this document: the `representations`/`chunks`/`spans` rows and the
per-type `rep_type` enum live in [df-003](../data-formats/df-003-sqlite-schema.md);
the citation-time `region` `Span` shape (bbox, section breadcrumb, label) lives
in [df-005](../data-formats/df-005-span.md). Type classification (which path a
document takes) and the rest of the ingest pipeline — re-indexing semantics,
chunking, and `dir2mcp doctor` diagnostics — live in
[bs-002](../behavior/bs-002-ingestion-pipeline.md). The capability-aware
extractor/transcriber selection and the model/provider bindings are
[td-001](td-001-provider-model.md); the audio transcription surface (timing,
diarization, translation, sidecars) is [td-003](td-003-transcription-translation-subtitles.md).

**Key principle.** Use the best available extractor by default **and** allow it
to be swapped per provider/transport. When the selected extractor is
unavailable, degrade gracefully and report coverage honestly — never silently
fall back across an explicit selection, and never report a broken extractor as
healthy.

## Specification (normative)

### A) Code / text / md / data / html

**Code / text / md / data.**

- Generate a `raw_text` representation: normalized UTF-8 with `\n` line endings.
- Route to index kind:
  - `code` → `index_kind=code`
  - all others → `index_kind=text`

**HTML.** HTML **MAY** be routed through structured extraction rather than flat
`raw_text`:

- When a structured extraction engine that accepts HTML is *available* — the
  docling family of §B, under the same `ingest.extractor` selection and the
  *Extractor availability* rules of §B — the pipeline **SHOULD** route HTML
  through it, producing an `extracted_markdown` representation and the
  structured `region` spans of §B (heading hierarchy → section breadcrumb;
  tables rendered atomically to Markdown; element labels in
  `extra_json.label`). HTML carries no page/`bbox` provenance, so its `region`
  spans carry the section breadcrumb and `label` and fall back to no page span,
  per the provenance-unavailable rule in §B.
- When no structured HTML engine is available — including `extractor: off`, an
  explicitly disabled/unavailable extractor
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)), or an engine that does
  not accept HTML — HTML **MUST** fall back to `raw_text`, exactly as before.
  `raw_text` remains the guaranteed baseline: HTML is never dropped and behavior
  **MUST NOT** regress when docling is absent.
- Either path routes to `index_kind=text`; the choice does not change the index
  kind and follows the re-indexing semantics of
  [bs-002](../behavior/bs-002-ingestion-pipeline.md) — a document previously
  indexed as flat `raw_text` keeps that representation until re-indexed.

The `raw_text` / `extracted_markdown` `rep_type` values and the `index_kind`
routing are persisted per
[df-003](../data-formats/df-003-sqlite-schema.md); the `region` `Span` shape is
[df-005](../data-formats/df-005-span.md).

> **Scope.** This governs only HTML's §A routing. The general per-format
> engine/type capability matrix is specified separately (dir2mcp #395); §A here
> narrowly permits a structured engine for the single HTML format (preferring
> it when available, `raw_text` otherwise) and defers the cross-format matrix to
> that work.

### B) PDF / image / document

Generate an `extracted_markdown` representation via the configured extractor
(`ingest.extractor`, [bs-011](../behavior/bs-011-configuration.md)):

- `auto` (default): prefer docling, fall back to Mistral OCR.
- `docling`: require a docling command/binary.
- `docling-serve`: require a reachable docling-serve HTTP endpoint (see below).
- `mistral`: require a Mistral OCR key/config.
- `off`: skip the extracted representation.

Route the extracted representation to `index_kind=text`. Cache extracted output
if caching is enabled.

#### Extractor transport

The `docling` *engine* produces the same structured document regardless of how
it is reached; the `ingest.extractor` value selects the transport explicitly:
`docling` invokes a local CLI subprocess, while `docling-serve` calls a
docling-serve HTTP service at the endpoint addressed by
`ingest.docling.serve_url` ([bs-011](../behavior/bs-011-configuration.md)). Both
transports **MUST** produce identical output (the same `extracted_markdown`
representation and `region` spans defined below); the choice is operational and
carries no wire- or schema-level difference. Extraction is selected via
`ingest.extractor` and is independent of the model/provider bindings in
[td-001](td-001-provider-model.md) — it is not a provider capability.

Selecting `docling-serve` **REQUIRES** a non-empty, reachable `serve_url`. An
empty or unreachable endpoint makes the `docling-serve` extractor
**unavailable** — a disabled extractor for diagnostic purposes
([bs-002](../behavior/bs-002-ingestion-pipeline.md)), exactly as a missing docling binary
disables `docling` — and **MUST NOT** silently fall back to the CLI. (Under
`extractor: auto` the transport is implementation-determined: an empty
`serve_url` simply means the HTTP transport is not considered, and `auto` may
use the CLI or another configured extractor as usual.)

#### Extractor availability

An extractor is *available* only when it can actually run, not merely when it is
configured. For the `docling` CLI this means the command both **resolves** (on
`PATH`, or via `ingest.docling.command`) **and** passes a lightweight functional
check — a successful probe invocation (for example `docling --version`). A
command that resolves but fails the probe — for example a bundled virtualenv
whose dependencies are ABI-incompatible — is **unavailable**, exactly as an
unreachable `serve_url` makes `docling-serve` unavailable. Implementations
**SHOULD** perform such a check and **MUST** treat a present-but-non-functional
extractor as unavailable (never as available), and **SHOULD** cache the result
for the run rather than probing per document.

- Under `extractor: auto`, an unavailable `docling` CLI is skipped and the
  cascade continues (docling-serve, then Mistral OCR, then disabled), so a
  broken docling install degrades gracefully instead of failing every document.
- Under `extractor: docling` (explicit), an unavailable command disables
  extraction — PDF/image/document contribute no `extracted_markdown` — and
  **MUST NOT** silently fall back to another engine, mirroring explicit
  `docling-serve`.
- The availability decision, and the reason when unavailable, **MUST** be
  surfaced in startup diagnostics and by `dir2mcp doctor`
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)), so a present-but-broken extractor
  is visible rather than reported as healthy.

#### Structured extraction (docling)

When the extractor emits a structured document model (docling's
`DoclingDocument`, obtained via `--to json`), the ingest pipeline **MUST**
preserve, not discard, the structure:

- Walk the document body in **reading order** (the `body` tree and group
  children), resolving internal references.
- Maintain a **section breadcrumb** from the heading hierarchy (`section_header`
  items and their levels); attach the active breadcrumb to every chunk emitted
  beneath it.
- Carry per-element **provenance**: page number and bounding box (`bbox`) from
  each element's provenance, stored as `region` spans
  ([df-003](../data-formats/df-003-sqlite-schema.md) `spans`;
  [df-005](../data-formats/df-005-span.md) `Span`). When provenance is
  unavailable for an element, fall back to a `page` span (or none).
- Preserve element **labels** (`paragraph`, `section_header`, `list_item`,
  `table`, `caption`, `code`, `formula`, `picture`) in span `extra_json.label`.
- **Tables** are rendered to faithful Markdown for the chunk text and kept
  atomic (a table is not split across chunks); cell structure (row/column spans)
  **MAY** additionally be retained in span `extra_json`.
- **Pictures/figures** contribute their captions and any classification
  annotations as searchable text, attributed to the figure's provenance.
- The document **title**, when the model exposes a `title` element, **SHOULD**
  be used to populate `documents.title` in preference to the text heuristic.

The `region` span emitted here is the at-rest form of the
[df-005](../data-formats/df-005-span.md) `region` `Span`: `start`/`end` carry the
first/last page (equal when single-page), `extra_json` **MUST** carry the `bbox`
and **SHOULD** carry the `section` breadcrumb, and `label` is a **single**
discrete enum value (not pipe-delimited). The exact field shape and the
primary-page / single-bbox-never-spans-pages constraints are normative in
[df-003 §5.4](../data-formats/df-003-sqlite-schema.md) and
[df-005](../data-formats/df-005-span.md).

#### What is persisted

The structured path does not change the persisted representation type or the
indexed content shape:

- The `extracted_markdown` representation stores **rendered Markdown** — the
  document's structure linearized to Markdown in reading order (tables as
  Markdown tables, figure captions inline). This is the text that is chunked,
  embedded, and returned in snippets, exactly as in the flat path. `rep_hash` is
  computed over this rendered Markdown.
- The structure that flat Markdown cannot carry — page, `bbox`, section
  breadcrumb, element label — is persisted as `region` **spans**
  ([df-003](../data-formats/df-003-sqlite-schema.md)) attached to each chunk, not
  as a separate representation.
- The raw `DoclingDocument` JSON is **not** a representation. Implementations
  **MAY** cache it (alongside the extracted output, when caching is enabled) to
  avoid re-running docling on re-index, but it is an implementation-private cache
  artifact, not part of the spec'd store contract.
- Re-indexing semantics are unchanged
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)): a document re-ingested under the
  structured path produces the same `extracted_markdown` representation; only the
  span provenance is richer. Documents previously ingested via flat Markdown keep
  their `page`/no spans until re-indexed.

See [Design 0002](../../design/0002-structured-extraction.md) for rationale and
the structure-to-provenance mapping.

#### Page-separated extraction (OCR fallback)

When the extractor emits only page-separated text (e.g. Mistral OCR), page-aware
behavior applies:

- store page numbers as `page` spans;
- chunk per page first.

### C) Audio (STT provider is configurable)

Generate a `transcript` representation via the STT provider. The transcription
surface itself — provider selection, timing modes, per-word timing, diarization,
translation, and sidecar ingestion — is normative in
[td-003](td-003-transcription-translation-subtitles.md); the transcript `meta_json` shape is
[df-003 §5.2](../data-formats/df-003-sqlite-schema.md). The representation-generation
rules are:

- Generate `transcript` via the STT provider:
  - default: **Mistral STT**;
  - optional: **ElevenLabs STT** (the provider enumeration is not closed — any
    STT-capable provider per [td-001](td-001-provider-model.md) is valid).
- If timestamps are available:
  - segment into time windows (e.g. 30s with 5s overlap);
  - store spans as `time` (`start_ms`/`end_ms`).
- If timestamps are not available:
  - fall back to text-size chunking;
  - omit time spans.
- Cache the transcript if caching is enabled.

### D) Structured extraction (annotations)

On-demand structured annotation of a document into typed fields:

- Default: **on-demand only**, via the MCP tool.
- Store an `annotation_json` representation.
- Optionally derive and embed an `annotation_text` representation:
  - flattened `key: value` lines;
  - route to `index_kind=text`.

The `annotation_json` / `annotation_text` `rep_type` values, and the requirement
that annotation `meta_json` record the provider/model, are normative in
[df-003 §5.2](../data-formats/df-003-sqlite-schema.md).

## Examples

A structured-extraction `region` span attached to a chunk (rendered shape; see
[df-005](../data-formats/df-005-span.md)):

```json
{ "kind": "region", "start_page": 4, "end_page": 4,
  "bbox": { "page": 4, "l": 72.0, "t": 90.5, "r": 523.0, "b": 410.2, "coord_origin": "TOPLEFT" },
  "section": ["Chapter 2", "2.1 Background"] }
```

A page-separated OCR fallback span:

```json
{ "kind": "page", "page": 4 }
```

## Changelog

- **0.1.0** — Migrated from SPEC.md §7.4 (parts A–D). Cross-references rewired to
  stable doc IDs: §5 → [df-003](../data-formats/df-003-sqlite-schema.md) (the
  `spans`/`representations` rows, formerly §5.4/§5.2); §15.1.1 →
  [df-005](../data-formats/df-005-span.md) (the `region` `Span`); §7 (re-index
  §7.6 and `doctor` diagnostics §7.7) →
  [bs-002](../behavior/bs-002-ingestion-pipeline.md); §8 (model/provider bindings) and
  the capability-aware extractor selection →
  [td-001](td-001-provider-model.md); §8.6 (audio transcription surface) →
  [td-003](td-003-transcription-translation-subtitles.md); §16.2 (`serve_url` config) →
  [bs-011](../behavior/bs-011-configuration.md); §14 →
  [df-008](../data-formats/df-008-error-taxonomy.md); §1 →
  [df-000](../data-formats/df-000-base.md). The region `Span` shape and the
  `spans` table layout are referenced, not redefined.
