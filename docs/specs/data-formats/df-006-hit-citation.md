# df-006: Hit and Citation

- **ID:** df-006
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §15.1.2 (reconciled with the implementation — see Changelog)

## Scope

A `Hit` is a single retrieval result returned by `dir2mcp_search` and inside
`dir2mcp_ask` (`hits[]`). A `Citation` is the answer-grounding reference
returned by `dir2mcp_ask` (`citations[]`). Both reference a [df-005
`Span`](df-005-span.md).

> **This document is the single source of truth for the Hit/Citation shape.**
> It resolves dir2mcp #423: the published machine-readable mirror
> `spec/tools/schemas/common.json` currently requires `chunk_id` as a **string**
> and fields `doc_type`/`rep`/`text`, which contradicts both the prose (legacy
> §15.1.2) and the implementation. A client validating a real response against
> that mirror fails on every hit. `common.json` MUST be reconciled to the schema
> below (tracked in the `df-007` migration); until then, `df-006` is authoritative.

## Specification (normative)

### `Hit`

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "chunk_id":  { "type": "integer" },
    "rel_path":  { "type": "string" },
    "title":     { "type": "string" },
    "doc_type":  { "type": "string" },
    "rep_type":  { "type": "string" },
    "score":     { "type": "number" },
    "snippet":   { "type": "string" },
    "span":      { "$ref": "df-005-span.md" },
    "modality":  { "type": "string", "description": "Media/multimodal chunks (td-002): e.g. \"text\", \"image\", \"audio\", \"video\". Omitted for plain text." },
    "media_ref": { "type": "string", "description": "Media/multimodal chunks (td-002): reference to the source media used to embed this chunk. Omitted for plain text." }
  },
  "required": ["chunk_id", "rel_path", "score", "snippet", "span"]
}
```

- `chunk_id` is an **integer** (NOT a string).
- The object is `additionalProperties: false`. Every field a producer can emit
  **MUST** be declared here. `title`, `modality`, and `media_ref` are optional
  and omitted when absent; a strict MCP client validates `structuredContent`
  against the tool `outputSchema`, so an **undeclared** field makes the whole
  tool call fail ("Failed to call tool"). *(This is the dir2mcp #387 class:
  `modality`/`media_ref` were emitted by the implementation but absent from the
  schema — they are now declared above.)*
- `span` is a [df-005 `Span`](df-005-span.md) and is required; see df-005 for the
  empty-`kind` rule (dir2mcp #397).

### `Citation`

A `Citation` is a `Hit` restricted to the fields needed to ground an answer.
It carries the same `chunk_id` (integer), `rel_path`, `title`, `score`, and
`span`; `snippet` is the cited text. Producers **MUST** use the same field
names and types as `Hit` (no `chunk_id`-as-string, no `text`/`rep` aliases).

## Conformance

The `df-007` tool-schema documents inline this shape as `definitions.Hit` /
`definitions.Citation` in each tool's `outputSchema`. Per dirstral-spec#26,
a CI fixture (an example `Hit`/`Citation` payload) MUST validate against the
inlined schema, and the implementation's serializer output MUST validate against
the published schema — the two-sided check that prevents the #387 and #423
drift classes from recurring (mirrored by the dir2mcp conformance test,
dir2mcp #428).

## Example

```json
{
  "chunk_id": 8421,
  "rel_path": "ActNo29of2024-FinancialInvestigationAgency(Amendment)Act,2024.pdf",
  "title": "No. 29 of 2024",
  "doc_type": "pdf",
  "rep_type": "extracted_markdown",
  "score": 0.0312,
  "snippet": "5P. (1) Subject to subsection (2), the Agency may, on the written request of a foreign financial investigation agency …",
  "span": {
    "kind": "region", "start_page": 13, "end_page": 13,
    "bbox": { "page": 13, "l": 72.0, "t": 88.0, "r": 523.0, "b": 240.0, "coord_origin": "TOPLEFT" },
    "section": ["VIRGIN ISLANDS", "Power to provide assistance to foreign financial investigation agency"]
  }
}
```

## Changelog

- **0.1.0** — Migrated from SPEC.md §15.1.2. Reconciled to the implementation:
  added the optional `title`, `modality`, and `media_ref` fields (dir2mcp #387);
  recorded `chunk_id` as **integer** and flagged the `common.json` mirror for
  reconciliation (dir2mcp #423); added the `Citation` shape and the conformance
  requirement.
