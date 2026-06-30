# dir2mcp Specification — Document Index

> **Status: PROPOSAL (dirstral-spec#24).** This restructures the monolithic
> `docs/SPEC.md` (~3300 lines) into small, **numbered, independently-versioned**
> documents grouped by concern, modeled on
> [ooni/spec](https://github.com/ooni/spec). This PR establishes the directory
> layout, the per-document convention, the full old→new mapping
> ([MIGRATION.md](MIGRATION.md)), and the **complete `df-*` data-format class**
> (`df-000`–`df-003`, `df-005`–`df-009`; `df-004` folded into df-000+df-003). The
> df-007 migration also **reconciled `spec/tools/schemas/common.json`** to the
> implementation, fixing the published-schema drift (dir2mcp #423). The
> `behavior/` and `techniques/` documents are listed below with their source
> sections and are migrated in follow-up PRs so each is reviewable on its own.
> `docs/SPEC.md` stays authoritative until a document is migrated and marked
> **Stable** here.

## Why

The codebase cites the spec by section number (`SPEC §8.1.7`, `§15.1.1`,
`§9.5`). Section numbers **renumber whenever SPEC.md is edited**, so those
references silently rot. A 3300-line monolith also makes per-area versioning
impossible, reviews coarse, and deprecation invisible — which is how
contradictory spec generations came to coexist (dir2mcp #422) and how the
published JSON-schema mirror drifted from the prose (dir2mcp #423).

Numbered documents give **stable, citable identifiers** (`df-006` never moves),
let each concern version on its own, and make a single source of truth
enforceable per document.

## Naming convention

`<class>-<NNN>-<slug>.md`, where `<class>` is:

| Class | Directory | Concern (OONI analogue) |
|-------|-----------|--------------------------|
| `df-` | `data-formats/` | The **shapes** — wire and at-rest payloads (OONI `data-formats/df-*`) |
| `bs-` | `behavior/`     | The **behavior** — semantics/contracts (OONI `nettests/ts-*`) |
| `td-` | `techniques/`   | The **techniques** — implementation strategies (OONI `techniques/`) |

A document's number is **permanent** once assigned (retired docs move to
`attic/` per dirstral-spec#25, never reused). Code SHOULD cite the document ID
(`df-006`), not a section number.

## Per-document header (template)

Every spec document begins with:

```
# df-006: Hit and Citation
- **ID:** df-006
- **Version:** 0.1.0          (semver; bump on any normative change)
- **Status:** Draft | Stable | Superseded
- **Supersedes:** —           (doc ID, if any)
- **Superseded-by:** —        (doc ID, if any)
- **Source:** SPEC.md §15.1.2 (provenance during migration)

## Scope
## Specification (normative — MUST/SHOULD/MAY)
## Example
## Changelog
```

## Document map

### `data-formats/` — payload shapes
| ID | Title | Source (SPEC.md) | Status |
|----|-------|------------------|--------|
| [df-000](data-formats/df-000-base.md) | Base conventions & `format_version` | §0, §1, NEW (dir2mcp #468) | **Draft** |
| [df-001](data-formats/df-001-connection-json.md) | `connection.json` | §4.3 | **Draft** |
| [df-002](data-formats/df-002-state-outputs.md) | State-dir outputs (layout, `secret.token`, `corpus.json`) | §4.1, §4.2, §4.4 | **Draft** |
| [df-003](data-formats/df-003-sqlite-schema.md) | SQLite metadata schema (documents/representations/chunks/spans/settings) | §5 | **Draft** |
| ~~df-004~~ | Document / representation / chunk model | §1.1, §5, §7 | **Folded** into df-000 (terms) + df-003 (schema) |
| [df-005](data-formats/df-005-span.md) | `Span` (lines/page/time/region/document + bbox) | §15.1.1 | **Draft** |
| [df-006](data-formats/df-006-hit-citation.md) | `Hit` and `Citation` | §15.1.2 | **Draft** |
| [df-007](data-formats/df-007-tool-schemas.md) | Tool input/output JSON schemas | §15.2–§15.11 + `spec/tools/schemas/*.json` | **Draft** (reconciled `common.json`, #423) |
| [df-008](data-formats/df-008-error-taxonomy.md) | Canonical error taxonomy | §14 | **Draft** |
| [df-009](data-formats/df-009-cli-output-contract.md) | CLI output contract | §3 | **Draft** |

### `behavior/` — semantics & contracts
| ID | Title | Source | Status |
|----|-------|--------|--------|
| bs-001 | CLI interface | §2 | To migrate |
| bs-002 | Ingestion pipeline | §7 | To migrate |
| bs-003 | Retrieval & answer generation | §9 | To migrate |
| bs-004 | MCP Streamable-HTTP transport | §10 | To migrate |
| bs-005 | MCP lifecycle (wire-level) | §11 | To migrate |
| bs-006 | MCP tools: list & call; tool set | §12, §13 | To migrate |
| bs-007 | Tool specifications (behavioral) | §15 | To migrate |
| bs-008 | Vector index backends & embed identity | §6 | To migrate |
| bs-009 | Security & safety requirements | §17 | To migrate |
| bs-010 | Native x402 integration | §18 (+ [x402-payment-adapter-spec.md](../x402-payment-adapter-spec.md)) | To migrate |
| bs-011 | Configuration (single file) | §16 | To migrate |

### `techniques/` — implementation strategies
| ID | Title | Source | Status |
|----|-------|--------|--------|
| td-001 | Model/provider utilization & capability activation | §8.1–§8.5 | To migrate |
| td-002 | Multimodal media chunks (PDF page / A-V window) | §8.1.7 | To migrate |
| td-003 | STT / transcription / translation / subtitles | §8.6 | To migrate |
| td-004 | Structured document extraction (docling) | §7.4 | To migrate |

### Meta / non-normative (kept at `docs/` root or moved to `attic/`)
- §0 Executive summary → folded into `df-000` + this index.
- §1 Definitions & invariants → `df-000` (terms) + per-doc invariants.
- §19 Non-goals, §20 Implementation guidance → keep as non-normative docs.

## Single source of truth

Per dirstral-spec#26: each data-format document is the **one** authoritative
shape. The machine-readable schemas under `spec/tools/schemas/` MUST be
generated from, or conformance-checked against, the `df-*` example payloads —
closing the prose↔schema↔code drift behind dir2mcp #423. The
`data_format_version` defined in `df-000` is the cross-version signal
(dir2mcp #468).
