# Design 0002: Structured Document Extraction (docling)

## Status

Accepted

## Context

dir2mcp normalizes PDFs, images, and office documents into
`extracted_markdown` via a configurable extractor (`ingest.extractor`),
with docling preferred and Mistral OCR as the fallback (SPEC §7.4.B).

Historically the docling backend invoked `docling --to md` and consumed
the resulting flat Markdown string. That collapses everything docling's
`DoclingDocument` model carries — reading order, section hierarchy,
per-element page and bounding-box provenance, structured tables, and
figure captions/classifications — into undifferentiated text. The
information is computed by docling and then thrown away at the process
boundary.

dir2mcp is a citations product: precise provenance is the differentiator.
Flat Markdown limits citations to whole-page granularity and forces naive
form-feed page chunking that ignores document structure.

## Decision

When the configured extractor exposes a structured model, ingest consumes
the structured `DoclingDocument` (via `docling --to json`) rather than flat
Markdown, and preserves the structure through to citations:

1. **Reading order** — walk the `body`/group tree, resolving internal
   references, instead of relying on serialized text order.
2. **Section breadcrumb** — track the heading hierarchy and attach the
   active breadcrumb to every chunk beneath it.
3. **Provenance** — carry page number + bounding box per element as a new
   `region` span kind (SPEC §5.4), stored in the existing `spans.extra_json`
   column (no schema migration).
4. **Tables** — render to faithful Markdown and keep atomic; optionally
   retain cell structure in `extra_json`.
5. **Figures** — index captions and classification annotations as
   searchable text attributed to the figure's provenance.
6. **Title** — prefer the model's title element over the text heuristic.

The richer provenance is surfaced in the client-facing `Citation` shared
type (SPEC §15.1) as additive, optional `bbox` and `section` fields, so a
`region` span renders region-accurate citations with a section trail.

Page-separated extractors (Mistral OCR) are unaffected: they continue to
emit `page` spans and per-page chunking. The structured path is selected
by capability detection on the extractor, not by a new mode, preserving
the `auto|docling|mistral|off` contract.

## Consequences

* Citations gain page + region + section granularity; chunking respects
  document structure rather than form feeds.
* `region` spans and the `bbox`/`section` citation fields are additive;
  existing `lines|page|time` provenance and clients that ignore the new
  fields are unaffected.
* Re-indexing is required to populate `region` spans for documents
  ingested under the flat-Markdown path; old chunks remain valid without
  bbox data.
* The default docling invocation changes to JSON output; an explicit
  `docling_command` override still takes precedence for advanced users.
* docling schema drift is a risk: the parser tolerates unknown fields and
  guards on the document `version`/`schema_name`.
