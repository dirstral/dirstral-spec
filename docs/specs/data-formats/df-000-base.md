# df-000: Base conventions & `format_version`

- **ID:** df-000
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §0, §1; introduces `format_version` (dir2mcp #468), NEW

## Scope

The conventions every other `df-*` document inherits: shared terminology, the
core invariants, and the **`format_version`** field that lets a consumer detect
which spec generation produced a payload. Modeled on ooni/spec's
`df-000-base.md`, whose `data_format_version` is the keystone of that project's
cross-version interoperability.

## `format_version` (normative)

Every self-describing payload dir2mcp writes at a boundary **MUST** carry a
`format_version` string (semver, e.g. `"0.1.0"`), so an independent
implementation always knows the shape it is reading and can adapt or reject:

- **`connection.json`** (df-001) MUST include `format_version`.
- **`stats` tool output** SHOULD include `format_version`; if it is absent, an
  MCP client MAY fall back to the daemon's implemented spec version to branch on
  payload evolution (e.g. the df-006 `Hit`/`Citation` changes).
- **SQLite** MUST set `PRAGMA user_version` to a monotonic schema version and
  check it on open: a database newer than the binary understands MUST be
  refused with a clear error, and a non-additive migration MUST be gated on the
  version (closes dir2mcp #405). `PRAGMA user_version` is an **independent**
  monotonic integer for the on-disk schema; it does not map to the semver
  `format_version` of the wire payloads above.
- **MCP `initialize`** MUST advertise the spec's pinned `protocolVersion`
  (`2025-11-25`) rather than echoing the client's requested version
  (closes dir2mcp #404).

A consumer that encounters a **major**-incompatible `format_version` MUST fail
closed (reject), not silently mis-parse. Additive (minor/patch) changes MUST be
backward-compatible: unknown fields are ignored, new optional fields are absent
on older producers.

## Terms (shared)

*(migrated from SPEC.md §1.1)*

- **Root directory** — the directory being indexed.
- **State directory** — storage for index state (default `<root>/.dir2mcp/`);
  always **local**, even when the corpus root is remote.
- **Document** — an ingestible unit (a file or an archive member).
- **Representation (rep)** — a text view derived from a document: `raw_text`,
  `extracted_markdown` (extractor output; formerly `ocr_markdown`), `transcript`
  (STT), `annotation_json`, `annotation_text`.
- **Chunk** — a span of a representation used for embedding and retrieval.
- **Span** — provenance coordinates for citations; see [df-005](df-005-span.md).

## Core invariants

*(migrated from SPEC.md §1.2)*

- The MCP server accepts lifecycle requests immediately after `dir2mcp up`
  prints the endpoint URL; indexing continues in the background and tools
  operate on a partial index if needed.
- No content outside the root is accessible via tools (no path traversal; no
  symlink escape).
- The default vector index is **embedded/on-disk** and requires **no external
  service**. An external store MAY be configured (bs-008, Tier C) but MUST NOT
  be required: a conforming deployment MUST run with zero external
  infrastructure beyond the model providers.
- The state directory is always **local**, even when the corpus root is remote
  (bs-002): SQLite metadata, the embedded index, and caches never live on the
  remote source.

## Changelog

- **0.1.0** — Initial. Established `format_version` (dir2mcp #468, folding in
  #404 and #405); migrated shared terms and invariants from SPEC.md §1.
