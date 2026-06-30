# bs-006: MCP tools — list, call, and tool set

- **ID:** bs-006
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §12, §13

## Scope

How an MCP client discovers dir2mcp's tools (`tools/list`), invokes them
(`tools/call`), and the result contract every tool call returns — including
structured output and tool-execution error (`isError`) handling. It also
enumerates the **tool set**: the core, recommended, and optional tools the
server exposes.

This document covers the *transport-level* tool contract only. Per-tool JSON
input/output schemas are normative in [df-007](../data-formats/df-007-tool-schemas.md);
per-tool behavior (semantics, arguments, error mapping) is normative in
[bs-007](bs-007-tool-specifications.md). The MCP session lifecycle that precedes
`tools/list` (`initialize`, `notifications/initialized`) is in
[bs-005](bs-005-mcp-lifecycle.md); the canonical error taxonomy referenced by
`isError` results is in [df-008](../data-formats/df-008-error-taxonomy.md).

## Specification (normative)

### tools/list & tools/call

#### Tool naming

All tools are prefixed with `dir2mcp_`. The historical dotted form
`dir2mcp.<tool>` is **superseded** as of spec `0.5.0`.

#### Tool discovery: `tools/list`

Request:

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {} }
```

The response contains an array of tools. Each tool object MUST include:

- `name`
- `description`
- `inputSchema` (a valid JSON Schema object)

Each tool object SHOULD include:

- `outputSchema` (a valid JSON Schema object)

The per-tool `inputSchema`/`outputSchema` shapes are defined in
[df-007](../data-formats/df-007-tool-schemas.md).

#### Tool invocation: `tools/call`

Request:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": { "name": "dir2mcp_search", "arguments": { "query": "..." } }
}
```

#### Tool result contract (MCP-native)

A tool-call response MUST return:

- `result.content[]` — at least one item.
- `result.structuredContent` — when supported by the negotiated protocol
  version (see [bs-005](bs-005-mcp-lifecycle.md) for version negotiation).
- `result.isError` set to `true` for **tool execution failures** (these are
  *not* JSON-RPC errors).

Tool execution failures MUST be returned as a successful JSON-RPC response with
`result.isError: true` — never as a JSON-RPC `error`. The error code carried in
`structuredContent.error.code` MUST be one of the canonical codes in
[df-008](../data-formats/df-008-error-taxonomy.md).

### Tool set

The server exposes three tiers of tools. Per-tool behavior is in
[bs-007](bs-007-tool-specifications.md); per-tool schemas are in
[df-007](../data-formats/df-007-tool-schemas.md).

#### Core tool set

- `dir2mcp_search`
- `dir2mcp_ask`
- `dir2mcp_open_file`
- `dir2mcp_list_files`
- `dir2mcp_stats`

#### Recommended extended tools

- `dir2mcp_transcribe` (audio → transcript, uses configured provider)
- `dir2mcp_annotate` (document → structured JSON + flattened text)
- `dir2mcp_transcribe_and_ask` (audio → transcript → ask)
- `dir2mcp_open_media_clip` (media hit → extracted audio/video snippet for a
  time span; see [bs-007](bs-007-tool-specifications.md))

#### Optional extension

- `dir2mcp_ask_audio` (answer → audio via ElevenLabs TTS)

## Examples

Example success result:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [{ "type": "text", "text": "..." }],
    "structuredContent": { "...": "..." }
  }
}
```

Example tool-execution error result (`isError`, not a JSON-RPC error):

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "isError": true,
    "content": [{ "type": "text", "text": "ERROR: FILE_NOT_FOUND: audio/meeting.wav" }],
    "structuredContent": {
      "error": { "code": "FILE_NOT_FOUND", "message": "audio/meeting.wav not found", "retryable": false }
    }
  }
}
```

## Changelog

- **0.1.0** — Migrated from SPEC.md §12 (MCP tools: list and call) and §13
  (Tool set). Cross-references rewired to stable doc IDs: the session lifecycle
  reference (§11) → [bs-005](bs-005-mcp-lifecycle.md); the error-taxonomy
  reference (§14) → [df-008](../data-formats/df-008-error-taxonomy.md); per-tool
  schemas (§15 schemas) → [df-007](../data-formats/df-007-tool-schemas.md);
  per-tool behavior (§15 behavior, incl. the `open_media_clip` §15.11 pointer) →
  [bs-007](bs-007-tool-specifications.md). The `tools/list` requirements were
  retightened into explicit MUST (`name`/`description`/`inputSchema`) and SHOULD
  (`outputSchema`) clauses without changing their force. Note: the original
  §12.1 pointer to `spec/versioning.md` for the superseded dotted form was
  dropped as that path is not part of the restructured spec tree; the `0.5.0`
  supersession date is preserved inline.
