# df-000: Base conventions & `format_version`

- **ID:** df-000
- **Version:** 0.2.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §0, §1, §1.3; introduces `format_version` (dir2mcp #468), NEW

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
- **`stats` tool output** SHOULD include `format_version`. When it is absent, a
  client MUST treat the payload as a pre-`format_version` (baseline) shape and
  MUST NOT infer the payload shape from `protocol_version` — that is the pinned
  MCP protocol generation, orthogonal to and unaffected by payload evolution
  (e.g. a df-006 `Hit`/`Citation` change).
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
backward-compatible — including under the closed (`additionalProperties: false`)
tool-output schemas df-007 mandates. That rests on a producer invariant: a
producer MUST NOT emit a field its **advertised** schema omits, so a new optional
field lands in the schema (and a tool's advertised `outputSchema`) in the same
spec version that first emits it. A conformant client validates against the
server-advertised schema (equivalently, the schema for the payload's declared
`format_version`), never a stale schema pinned out of band; tolerant readers
ignore unknown fields. New optional fields are absent on older producers. See
SPEC.md §1.3 for the normative statement.

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

- **0.2.0** — The `format_version` contract is now mirrored authoritatively in
  SPEC.md §1.3 (with the four boundaries wired into §4.3 connection.json, §5.6
  the `PRAGMA user_version` fence, §11.2 the `initialize` protocolVersion pin,
  and §15.6 + `stats.json` the optional stats field), so implementations have an
  authoritative surface to conform to while this Draft is not yet Stable. Two
  clarifications vs 0.1.0: (a) absence of `format_version` means a pre-field
  baseline shape — a client MUST NOT infer the shape from `protocol_version`
  (pinned, orthogonal to payload evolution); (b) additive backward-compat under
  the closed df-007 tool schemas rests on a producer invariant (emit only what
  the advertised schema declares; new optional fields land in the schema in the
  same version that emits them), not on lenient consumers.
- **0.1.0** — Initial. Established `format_version` (dir2mcp #468, folding in
  #404 and #405); migrated shared terms and invariants from SPEC.md §1.
