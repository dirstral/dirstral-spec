# df-007: Tool input/output JSON schemas

- **ID:** df-007
- **Version:** 0.5.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §15.2–§15.12 + `spec/tools/schemas/*.json`

## Scope

The canonical machine-readable JSON Schemas for every MCP tool's `inputSchema`
and `outputSchema`, and the shared `definitions` they build on. The schema
**files** under [`spec/tools/schemas/`](../../../spec/tools/schemas/) are the
source of truth; this document catalogs them, maps each to its tool, and states
the conformance contract. Shared shapes — [df-005 `Span`](df-005-span.md) and
[df-006 `Hit`/`Citation`](df-006-hit-citation.md) — live in `common.json` and are
`$ref`-ed by the per-tool files.

## Specification (normative)

### Schema files

| Tool (SPEC §) | Schema file | Notes |
|---------------|-------------|-------|
| shared types | `common.json` | `Span` (df-005), `Hit` + `Citation` (df-006). |
| `dir2mcp_search` (§15.2) | `search.json` | `outputSchema.hits[]` `$ref`s `common.json#/definitions/Hit`. |
| `dir2mcp_ask` (§15.3) | `ask.json` | `citations[]` `$ref`s `Citation`; `hits[]` `$ref`s `Hit`. |
| `dir2mcp_open_file` (§15.4) | `open_file.json` | Returns `content` + a df-005 `Span` (incl. the `document` variant). |
| `dir2mcp_list_files` (§15.5) | `list_files.json` | |
| `dir2mcp_stats` (§15.6) | `stats.json` | SHOULD carry `format_version` (df-000). Optional additive `skip_reasons` coverage array (reason→count; closed reason enum). |
| `dir2mcp_transcribe` (§15.7) | `transcribe.json` | recommended tool. |
| `dir2mcp_annotate` (§15.8) | `annotate.json` | recommended tool. |
| `dir2mcp_transcribe_and_ask` (§15.9) | `transcribe_and_ask.json` | recommended tool. |
| `dir2mcp_ask_audio` (§15.10) | `ask_audio.json` | optional extension. |
| `dir2mcp_open_media_clip` (§15.11) | `open_media_clip.json` | `CLIP_TOO_LARGE`/`MEDIA_CLIP_FAILED` (df-008). |
| `dir2mcp_related` (§15.12) | `related.json` | Query-by-example; `hits[]` `$ref`s `Hit`. One of `chunk_id`/`rel_path` (`INVALID_FIELD` otherwise). Optional extension. |

### Single source of truth

Each shared shape is defined **once** in `common.json` and `$ref`-ed everywhere
else; a per-tool file MUST NOT inline a divergent copy of `Hit`, `Citation`, or
`Span`. The published schema is what an independent implementation validates
against, so it MUST equal the server's served `outputSchema` for the same tool.

Because the wire payload is `structuredContent` validated by strict MCP clients
against `outputSchema`, every object schema that a tool can emit is
`additionalProperties: false`, and **every field the serializer can produce MUST
be declared** — an undeclared field fails the whole tool call ("Failed to call
tool"). This is the df-006 `modality`/`media_ref` lesson (dir2mcp #387) and the
df-005 non-empty-`kind` lesson (dir2mcp #397).

## Conformance

Per dirstral-spec#26, two checks keep prose, schema, and code in agreement:

1. **Fixture validates against schema** — each shared shape ships an example
   payload (an example `Hit`/`Citation`/`Span`) that MUST validate against its
   `common.json` definition in CI.
2. **Serializer output validates against schema** — the implementation's served
   `outputSchema` and emitted `structuredContent` MUST validate against the
   published `common.json`/per-tool schemas. dir2mcp's dependency-free
   conformance test (dir2mcp #428, PR #467) performs the code side of this over
   all tools; it is the regression guard for the #387/#423 drift classes.

## Reconciliation note (dir2mcp #423)

`common.json`'s `Hit`/`Citation` previously diverged from the implementation
(`chunk_id` as **string**; `rep`/`text`/`quote` instead of `rep_type`/`snippet`;
missing `title`/`modality`/`media_ref`; a `Citation` with phantom
`doc_type`/`rep`/`score`/`quote`). A client validating a real response against
the published schema failed on every hit. This document's migration **rewrote
`common.json`'s `Hit` and `Citation`** to match the served `outputSchema`
(`hitDefinitionSchema` / `askOutputSchema` in `internal/mcp/tools.go`) verbatim;
the `Span` definition was already correct. `search.json`/`ask.json` `$ref`
`common.json`, so the fix propagates without further edits.

## Changelog

- **0.5.0** — Added `related.json` (dir2mcp_related §15.12, dir2mcp #324): the
  query-by-example 'more like this' tool, cataloged in the tool map; its `hits[]`
  `$ref` `common.json#/definitions/Hit`, so no shared-shape change.
- **0.4.0** — `stats.json` `skip_reasons[].reason` enum gains `language_uncovered`
  (additive; SPEC §8.2.1/§15.2, dir2mcp #566): media skipped under
  `media.stt.on_uncovered_language=skip` because its source language is outside
  the STT model's declared coverage. This is a forward-extensible enum: a
  conformant client validates against the **server-advertised** schema (which
  carries the new value, per SPEC §1.3) and, per the enum's own rule, SHOULD render
  an unrecognized `reason` verbatim rather than reject — so a client on this spec
  or newer accepts it. Strict-validating `reason` against a stale, out-of-band copy
  of an older closed enum is the non-conformant case the render-verbatim rule
  exists to prevent.
- **0.3.0** — `stats.json` now declares the optional top-level `format_version`
  string (the df-000 cross-version signal; SHOULD, additive), matching the prose
  contract in this document and SPEC §1.3/§15.6 (dir2mcp #468).
- **0.1.0** — Cataloged the `spec/tools/schemas/*.json` files and the tool map;
  stated the single-source-of-truth + conformance contract; recorded the
  `common.json` `Hit`/`Citation` reconciliation (dir2mcp #423).
