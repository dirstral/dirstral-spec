# dir2mcp Specification — Document Index

> **Status: PROPOSAL (dirstral-spec#24).** This restructures the monolithic
> `docs/SPEC.md` (~3300 lines) into small, **numbered, independently-versioned**
> documents grouped by concern, modeled on
> [ooni/spec](https://github.com/ooni/spec). **All three classes are now drafted**
> — 9 `df-*` (data formats), 11 `bs-*` (behavior), 5 `td-*` (techniques) — each
> migrating one SPEC.md section per the [migration map](MIGRATION.md), with the
> per-document convention and the full old→new mapping established here. The
> df-007 migration also **reconciled `spec/tools/schemas/common.json`** to the
> implementation, fixing the published-schema drift (dir2mcp #423). Every
> inter-document link is validated to resolve. `docs/SPEC.md` retains a pointer
> banner and **stays authoritative** until each document is reviewed and marked
> **Stable** here (the docs are currently **Draft**).

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
| [bs-001](behavior/bs-001-cli-interface.md) | CLI interface | §2 | **Draft** |
| [bs-002](behavior/bs-002-ingestion-pipeline.md) | Ingestion pipeline | §7 (excl. §7.4) | **Draft** |
| [bs-003](behavior/bs-003-retrieval-and-answer.md) | Retrieval & answer generation | §9 | **Draft** |
| [bs-004](behavior/bs-004-mcp-transport.md) | MCP Streamable-HTTP transport | §10 | **Draft** |
| [bs-005](behavior/bs-005-mcp-lifecycle.md) | MCP lifecycle (wire-level) | §11 | **Draft** |
| [bs-006](behavior/bs-006-mcp-tools-list-call.md) | MCP tools: list, call & tool set | §12, §13 | **Draft** |
| [bs-007](behavior/bs-007-tool-specifications.md) | Tool specifications (behavior) | §15.2–§15.11 | **Draft** |
| [bs-008](behavior/bs-008-vector-index.md) | Vector index backends & embed identity | §6 | **Draft** |
| [bs-009](behavior/bs-009-security-safety.md) | Security & safety requirements | §17 | **Draft** |
| [bs-010](behavior/bs-010-x402-integration.md) | Native x402 integration | §18 (+ [adapter spec](../x402-payment-adapter-spec.md)) | **Draft** |
| [bs-011](behavior/bs-011-configuration.md) | Configuration (single file) | §16 | **Draft** |

### `techniques/` — implementation strategies
| ID | Title | Source | Status |
|----|-------|--------|--------|
| [td-001](techniques/td-001-provider-model.md) | Provider model & capability activation | §8.1–§8.5, §8.8 | **Draft** |
| [td-002](techniques/td-002-multimodal-media-chunks.md) | Multimodal media chunks (PDF page / A-V window) | §8.1.7 | **Draft** |
| [td-003](techniques/td-003-transcription-translation-subtitles.md) | Transcription / translation / subtitles | §8.6 | **Draft** |
| [td-004](techniques/td-004-representation-extraction.md) | Representation generation & structured extraction | §7.4 | **Draft** |
| [td-005](techniques/td-005-distributed-embedding.md) | Distributed embedding (coordinator + workers) | §8.7 | **Draft** |

### Meta / non-normative (kept at `docs/` root or moved to `attic/`)
- §0 Executive summary → folded into `df-000` + this index.
- §1 Definitions & invariants → `df-000` (terms) + per-doc invariants.
- §19 Non-goals, §20 Implementation guidance → keep as non-normative docs.

## Single source of truth

Per dirstral-spec#26: each data-format document is the **one** authoritative
shape. The machine-readable schemas under `spec/tools/schemas/` MUST be
generated from, or conformance-checked against, the `df-*` example payloads —
closing the prose↔schema↔code drift behind dir2mcp #423. The
`format_version` defined in `df-000` is the cross-version signal
(dir2mcp #468).
