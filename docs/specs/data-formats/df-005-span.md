# df-005: Span

- **ID:** df-005
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §15.1.1 (migrated verbatim; one normative clarification added, see Changelog)

## Scope

A `Span` is the provenance coordinate for a citation: a line range, a page, a
time range, a page-region with a bounding box, or a whole document. It is a
shared type referenced by [df-006 (`Hit`/`Citation`)](df-006-hit-citation.md)
and by the `open_file` tool output.

## Specification (normative)

A `Span` is exactly one of five variants, selected by `kind`. Each variant is
`additionalProperties: false`.

```json
{
  "type": "object",
  "oneOf": [
    {
      "additionalProperties": false,
      "properties": { "kind": { "const": "lines" }, "start_line": { "type": "integer" }, "end_line": { "type": "integer" } },
      "required": ["kind", "start_line", "end_line"]
    },
    {
      "additionalProperties": false,
      "properties": { "kind": { "const": "page" }, "page": { "type": "integer" } },
      "required": ["kind", "page"]
    },
    {
      "additionalProperties": false,
      "properties": {
        "kind": { "const": "time" },
        "start_ms": { "type": "integer" },
        "end_ms": { "type": "integer" },
        "speaker": { "type": "string", "description": "Optional (td-003): stable per-transcript speaker id on a diarized transcript." },
        "speaker_label": { "type": "string", "description": "Optional human-readable speaker name (td-003)." }
      },
      "required": ["kind", "start_ms", "end_ms"]
    },
    {
      "additionalProperties": false,
      "properties": {
        "kind": { "const": "region" },
        "start_page": { "type": "integer" },
        "end_page": { "type": "integer" },
        "bbox": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "page": { "type": "integer" },
            "l": { "type": "number" }, "t": { "type": "number" },
            "r": { "type": "number" }, "b": { "type": "number" },
            "coord_origin": { "enum": ["TOPLEFT", "BOTTOMLEFT"] }
          },
          "required": ["page", "l", "t", "r", "b", "coord_origin"]
        },
        "section": { "type": "array", "items": { "type": "string" } }
      },
      "required": ["kind", "start_page", "end_page", "bbox"]
    },
    {
      "additionalProperties": false,
      "properties": { "kind": { "const": "document" } },
      "required": ["kind"]
    }
  ]
}
```

A producer **MUST** emit exactly one of the five defined `kind` values. A chunk
that lacks finer provenance **MUST** be serialized as a `page` span (when a page
is known) or a `document` span — **never** with an empty or unrecognized `kind`.
A `Span` whose `kind` is absent, empty, or outside the enum is non-conforming
and **MUST** be rejected by a strict client (it matches no `oneOf` branch). *(This
clarification codifies the fix for dir2mcp #397: a BM25 hit lacking span
metadata previously serialized `{"kind":""}`, which a strict MCP client rejects
as "Failed to call tool"; the corrected behavior is a `document` fallback.)*

The `region` variant is emitted by structured document extraction (td-004). It
localizes a chunk to a page range (`start_page`/`end_page`, equal when
single-page) and always carries a bounding box (`bbox`); an element without
provenance is recorded as a `page` span instead, never a `region` span with a
missing `bbox`. The section breadcrumb (`section`) is optional (`[]` when none).
The `region` kind and its `section` field are **additive**: clients that do not
recognize the `region` kind, or that ignore `section`, **MUST** degrade
gracefully (treat as a page-level citation on `start_page`).

The `document` variant is emitted by `dir2mcp_open_file` when the requested
`rel_path` is a binary doc type (PDF, audio) and the caller did not supply
`page`, `start_ms`/`end_ms`, or `start_line`/`end_line`. It signals that
`content` is the full extracted / transcript representation rather than a paged
or timed slice.

## Examples

```json
{ "kind": "lines", "start_line": 10, "end_line": 18 }
{ "kind": "page", "page": 4 }
{ "kind": "time", "start_ms": 120000, "end_ms": 135000, "speaker": "S1", "speaker_label": "Anchor" }
{ "kind": "region", "start_page": 4, "end_page": 5,
  "bbox": { "page": 4, "l": 72.0, "t": 90.0, "r": 523.0, "b": 410.0, "coord_origin": "TOPLEFT" },
  "section": ["VIRGIN ISLANDS", "Power to provide assistance"] }
{ "kind": "document" }
```

## Changelog

- **0.1.0** — Migrated from SPEC.md §15.1.1. Added the normative MUST that a
  producer emit a defined non-empty `kind` (codifies dir2mcp #397). Updated
  internal cross-references from `§8.6.8`/`§7.4.B` to `td-003`/`td-004`.
