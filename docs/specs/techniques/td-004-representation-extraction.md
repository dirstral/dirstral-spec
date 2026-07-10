# td-004: Representation generation & structured extraction

- **ID:** td-004
- **Version:** 0.4.0
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

**Markup boundary (html).** `html` is a *dual-path* format: it MAY be handled
here as flat `raw_text`, or routed to a structured extraction engine (§B.1) that
preserves headings/tables/links. §B.1 lists `html` as structured-capable so that
best-available selection is *permitted* to promote it; this section no longer
*requires* html to take the flat path. The **default** html routing is deferred
to dir2mcp #556 and left unchanged here — until #556 lands an implementation MAY
continue to route html to `raw_text` and MUST NOT be considered non-conforming
for doing so.

### B) PDF / image / document

Generate an `extracted_markdown` representation via a **capability-aware,
per-format** selection over the extraction-engine registry (§B.1).
`ingest.extractor` ([bs-011](../behavior/bs-011-configuration.md)) selects the
*policy*, not a single global engine:

- `auto` (default): **best available per format** — for each format, use the
  highest-fidelity *active* engine that supports it (§B.1), falling through the
  fidelity order; a format no active engine supports degrades per the
  strict/lenient contract (§B.2).
- `docling` / `docling-serve` / `mistral` / `pandoc`: **pin** a single engine. A
  format the pinned engine cannot read does not silently produce an empty
  representation — it degrades honestly per §B.2.
- `off`: skip the extracted representation.

Route the extracted representation to `index_kind=text`. Cache extracted output
if caching is enabled.

#### Extractor transport

The `docling` *engine* produces the same structured document regardless of how
it is reached; the `docling` vs `docling-serve` engine selection is the
transport: `docling` invokes a local CLI subprocess, while `docling-serve` calls
a docling-serve HTTP service at the endpoint addressed by
`ingest.docling.serve_url` ([bs-011](../behavior/bs-011-configuration.md)). Both
transports **MUST** produce identical output (the same `extracted_markdown`
representation and `region` spans defined below); the choice is operational and
carries no wire- or schema-level difference.

**Extraction is a §B routing decision, not a
[td-001](td-001-provider-model.md) §8.1.2 capability cell.** Per-format engine
selection lives here (§B.1), *not* in the td-001 §8.1.2 capability matrix:
extraction fidelity is per-format and ordered, and two of the engines
(`docling`, `pandoc` #393) are local tools with no provider profile.
Where an engine *is* a td-001 §8 surface — the `mistral` engine — it resolves
through that capability's binding: the `mistral` extraction engine is the active
`ocr` provider ([td-001](td-001-provider-model.md) §8.1.2/§8.1.3), so the
OCR-tier engine follows the `ocr` binding rather than being pinned to a vendor
name. The audio path (§C) already binds its engine to the `stt` capability; §B
generalizes the same best-available-by-default, swappable, honestly-degrading
shape to documents and images.

#### B.1) Extraction-engine capability matrix (normative)

The **extraction-engine registry** is the single source of truth for which
engine can ingest which format, replacing scattered MIME allowlists and coarse
`doc_type` routing. Each engine declares the format classes it supports and a
**fidelity tier** (lower = higher fidelity = preferred as the best-available
tiebreak):

| Tier | Engine | Nature | Provenance produced |
|---|---|---|---|
| T1 | `docling` / `docling-serve` | structured document model | reading-order, `region` (page+bbox), section breadcrumb, labels, atomic tables (§B "Structured extraction") |
| T2 | `pandoc` (#393) | born-digital markup/office/ebook → Markdown | structure without page/bbox: section breadcrumb + `label`; no `page`/`bbox` spans |
| T3 | `mistral` (= td-001 §8 `ocr` provider) | page-separated OCR | `page` spans (§B "Page-separated extraction") |
| T4 | `raw_text` (§A) | flat text | none |

**Format support** (`✅` = engine can ingest this format; tier from the table
above). The `pandoc` engine (#393) is **optional and capability-activated**: its
cells participate in selection whenever a `pandoc` binary is available (see
*Extractor availability*) and are inactive otherwise, exactly as a missing
`docling` binary deactivates the T1 cells:

| Format class | Examples | docling(-serve) | mistral (ocr) | pandoc† | raw_text |
|---|---|:--:|:--:|:--:|:--:|
| pdf | `.pdf` | ✅ T1 | ✅ T3 | ❌ | ❌ |
| raster-image (OCR-native) | `.png .jpg .jpeg .webp` | ✅ T1 | ✅ T3 | ❌ | ❌ |
| raster-image (extended) | `.tiff .bmp .gif` | ✅ T1 | ❌ | ❌ | ❌ |
| vector-image | `.svg` | ✅ T1 | ❌ | ❌ | ❌ |
| office (Word, OOXML) | `.docx` | ✅ T1 | ❌ | ✅ T2 | ❌ |
| office (slides/sheets, OOXML) | `.pptx .xlsx` | ✅ T1 | ❌ | ❌ | ❌ |
| office/ebook (ODF/RTF/EPUB) | `.odt .rtf .epub` | ❌ | ❌ | ✅ T2 | ❌ |
| legacy office (binary) | `.doc` | ❌ | ❌ | ❌ | ❌ |
| markup | `.html .htm` | ✅ T1 | ❌ | ✅ T2 | ✅ T4 (§A, #556) |

† `pandoc` (T2, #393) is a born-digital markup/office/ebook converter with a
**reader-only** support set: it ingests `.docx`, `.odt`, `.rtf`, `.epub`, and
`.html`, but **not** `.pptx`/`.xlsx` (pandoc has no PowerPoint/Excel reader —
those are docling-only) nor legacy binary `.doc` (docx-only), and no raster/PDF
input — so those cells are permanently `❌`. Its readable cells are active only
when a `pandoc` binary is available; an implementation or deployment without
`pandoc` treats them as inactive, exactly as a missing `docling` binary
deactivates T1.

**Best-available selection (`extractor: auto`).** For each classified document,
select the **active** engine of lowest fidelity tier whose cell for that format
is `✅`. "Active" means *available* in the §B "Extractor availability" sense
(resolves + passes its probe; a reachable `serve_url`; a present `ocr`
credential/binding). The selection is **per format**, deterministic, and cached
for the run. A format with an active engine at some tier is never routed to an
engine that cannot read it, and a higher-fidelity active engine is never
bypassed (fixing the "html→raw_text while docling is active" and
"tiff→mistral-rejected" defects, dir2mcp #394/#556).

**Pinned selection (`extractor: docling|docling-serve|mistral|pandoc`).** Only
the named engine is eligible; formats outside its `✅` set degrade per §B.2.
Pinning is honored exactly (no cross-engine fallback), matching the existing
explicit-`docling` / explicit-`docling-serve` no-silent-fallback rule. Pinning
`pandoc` when no `pandoc` binary is available disables extraction, exactly as
pinning an unavailable `docling`.

#### B.2) Degradation contract (strict / lenient)

When no active eligible engine supports a document's format (a coverage gap under
`auto`, or a pinned engine that cannot read the format), the outcome is governed
by `ingest.on_unsupported` ([bs-011](../behavior/bs-011-configuration.md)), a
kill-switch-shaped knob mirroring the tri-state opt-out used elsewhere (e.g.
`media.diarize`, [td-003](td-003-transcription-translation-subtitles.md)):

- **`lenient` (default, backward-compatible)** — **skip with warning**: no
  `extracted_markdown` is produced, the document is indexed with whatever other
  representations it has (or none), and the gap is surfaced as a warning in
  startup diagnostics and the honest coverage report
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)). This preserves the
  current not-indexed *outcome* for unsupported formats while replacing the
  former **silent** empty representation with an honest, named one.
- **`strict`** — the unsupported format is a **non-fatal per-document error**
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)): `documents.status=error`
  with an `UNSUPPORTED_FORMAT`-class reason; indexing continues for other
  documents. Intended for CI / correctness-sensitive corpora that must not
  silently under-cover.

In neither mode is an unsupported format allowed to yield a silent empty
representation reported as success.

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
  per-format tier order continues (docling-serve, then `pandoc` for the formats
  it covers, then Mistral OCR, then `raw_text` for HTML (§A), then disabled), so
  a broken docling install
  degrades gracefully instead of failing every document.
- Under `extractor: docling` (explicit), an unavailable command disables
  extraction — PDF/image/document contribute no `extracted_markdown` — and
  **MUST NOT** silently fall back to another engine, mirroring explicit
  `docling-serve`.
- The availability decision, and the reason when unavailable, **MUST** be
  surfaced in startup diagnostics and by `dir2mcp doctor`
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)), so a present-but-broken extractor
  is visible rather than reported as healthy.

The `pandoc` engine (T2, #393) follows the same availability rule: it is
*available* only when a `pandoc` binary both **resolves** (on `PATH`, or via
`ingest.pandoc.command`) **and** passes a `pandoc --version` functional check. It
is **capability-activated** — no enable flag; a working binary activates the T2
matrix cells and its absence deactivates them (opt-out only). Under
`extractor: auto` an unavailable `pandoc` is skipped and the per-format tier
order continues; under `extractor: pandoc` (explicit) an unavailable binary
disables extraction and **MUST NOT** silently fall back, mirroring explicit
`docling`.

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

#### Markup/office extraction (pandoc) (#393)

When the active engine is `pandoc` (T2) — a born-digital converter with no page
raster or layout model — the pipeline produces an `extracted_markdown`
representation by converting the source to Markdown, and **MUST NOT** fabricate
page/`bbox` provenance it does not have:

- Convert the document to Markdown (`pandoc -t gfm`), preserving reading order —
  pandoc emits a single linear document.
- An implementation **SHOULD**, where the Markdown heading hierarchy is
  available, carry a **section breadcrumb** onto the chunks beneath each heading
  as the structured path does, and **MAY** carry an element kind (e.g. table,
  code block) in span `extra_json.label`. Unlike docling's structured model this
  is a **progressive enhancement** over the guaranteed Markdown text, not a
  structured-model guarantee.
- **No page/`bbox` provenance exists** for born-digital formats: pandoc spans
  carry the section breadcrumb (and `label` where derivable) and otherwise fall
  back to **no `page` span** — the pipeline **MUST NOT** fabricate one. This is
  the provenance-unavailable rule of the structured path applied to an engine
  that never has page provenance. Citations are therefore section-granular,
  coarser than docling's `region` spans.
- **Tables** are rendered to Markdown and kept atomic where the converter
  preserves them.
- Route to `index_kind=text`. `rep_hash` is computed over the rendered Markdown,
  exactly as the docling and flat paths; the persisted representation type is
  unchanged (`extracted_markdown`), only the span provenance is coarser.
- Re-indexing semantics are unchanged
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)): under `auto`, a format
  later covered by a higher-fidelity active engine (e.g. docling installed) is
  re-extracted through it on re-index per the best-available rule; until then the
  pandoc representation stands.

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

- **0.4.0** — Made the §B.1 T2 `pandoc` engine binding rather than
  forward-looking (dir2mcp #393): pandoc is a **capability-activated** born-digital
  markup/office/ebook converter (active iff a `pandoc` binary resolves + passes a
  `pandoc --version` functional check, via `PATH` or `ingest.pandoc.command`);
  added it to the pinnable engine set; added its availability rule (skipped under
  `auto`, no-silent-fallback when pinned) and a "Markup/office extraction
  (pandoc)" output-shape section (Markdown conversion; section breadcrumb a
  SHOULD progressive enhancement, not a structured-model guarantee; **no**
  page/`bbox` provenance — section-granular citations). Corrected pandoc's
  matrix cells to its **reader-only** set — `.docx .odt .rtf .epub .html`;
  `.pptx`/`.xlsx` (docling-only) and legacy `.doc` are pandoc `❌`, split out into
  their own rows — and spelled out the §A `raw_text`-for-HTML step in the auto
  tier order. No behavior change when `pandoc` is absent.
- **0.3.0** — Reversed §B from single global extractor selection to
  capability-aware per-format selection: added the §B.1 engine capability matrix
  + fidelity ordering and the §B.2 strict/lenient degradation contract; recorded
  the governance call (extraction stays a §B routing decision, not a td-001
  §8.1.2 capability cell; the `mistral` engine binds the `ocr` capability); noted
  the §A markup (html) dual-path boundary (dir2mcp #556). Unblocks dir2mcp #395
  Stages 2–3.
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
