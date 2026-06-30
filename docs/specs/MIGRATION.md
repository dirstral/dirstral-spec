# SPEC.md → numbered-docs migration map

This table maps every section of the legacy monolithic `docs/SPEC.md` to its
target document under `docs/specs/`. It is the checklist for the split
(dirstral-spec#24). A row is **Done** only when the target document is created,
marked `Status: Stable`, and the corresponding SPEC.md section is replaced by a
one-line pointer (`Moved to df-006`).

`SPEC.md` line numbers are as of the branch base (commit `b26728e`); they are a
migration aid only and are not citable.

| SPEC.md § | Lines | → Target doc | State |
|-----------|-------|--------------|-------|
| §0 Executive summary | 12–66 | `specs/README.md` + `df-000` | Folded |
| §1 Definitions & invariants | 67–91 | `df-000` (terms) | Exemplar (partial) |
| §2 CLI interface | 92–164 | `bs-001` | To do |
| §3 CLI output contract | 165–279 | `df-009` | To do |
| §4.3 connection.json | 321–339 | **`df-001`** | **Done** |
| §4 On-disk outputs (rest) | 280–411 | `df-002` (`secret.token`/`corpus.json`/`daemon.json`/pid) | To do |
| §5 SQLite metadata schema | 412–592 | `df-003`, `df-004` (model) | To do |
| §6 Vector index backends & identity | 593–686 | `bs-008` | To do |
| §7 Ingestion pipeline | 687–1104 | `bs-002`; §7.4 → `td-004` | To do |
| §8 Model/provider utilization | 1105–1807 | `td-001`; §8.1.7 → `td-002`; §8.6 → `td-003` | To do |
| §9 Retrieval & answer generation | 1808–1959 | `bs-003` | To do |
| §10 MCP Streamable HTTP | 1960–2023 | `bs-004` | To do |
| §11 MCP lifecycle (wire) | 2024–2095 | `bs-005` | To do |
| §12 MCP tools list/call | 2096–2168 | `bs-006` | To do |
| §13 Tool set | 2169–2191 | `bs-006` | To do |
| §14 Error taxonomy | 2192–2260 | **`df-008`** | **Done** |
| §15.1.1 Span | 2264–2320 | **`df-005`** | **Done (exemplar)** |
| §15.1.2 Hit | 2336–2360 | **`df-006`** | **Done (exemplar)** |
| §15.2+ Tool schemas | 2360–2929 | `df-007` (schemas) + `bs-007` (behavior) | To do |
| §16 Configuration | 2930–3206 | `bs-011` | To do |
| §17 Security & safety | 3207–3236 | `bs-009` | To do |
| §18 Native x402 | 3237–3253 | `bs-010` | To do |
| §19 Non-goals | 3254–3271 | `docs/scope.md` (non-normative) | To do |
| §20 Implementation guidance | 3272–end | `docs/guidance.md` (non-normative) | To do |

## Drift fixes folded into this work

- **dir2mcp #423** — `spec/tools/schemas/common.json` requires `chunk_id` as a
  **string** + `doc_type`/`rep`/`text`, contradicting both the prose (§15.1.2)
  and the implementation (integer + `snippet`/`span`). `df-006` records the
  authoritative shape (matching the prose + impl, including the `modality` /
  `media_ref` fields that dir2mcp #387 added but §15.1.2 still omits). `common.json`
  is reconciled to `df-006` in the `df-007` migration.
- **dir2mcp #422** — the code follows a quarantine model attributed to
  "spec 0.16.0" that conflicts with current SPEC.md. The behavior split
  (`bs-002`/`bs-007`) + the per-doc version header + `attic/` (dirstral-spec#25)
  make such a conflict visible instead of silent.
- **dir2mcp #468 / #404 / #405** — `df-000` introduces `format_version`, the
  cross-version signal the data currently lacks.
