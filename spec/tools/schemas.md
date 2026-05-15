# Tool Schemas

**Spec version:** `0.4.0`
**MCP protocol target:** `2025-11-25`

This document defines the canonical input/output schemas for all dir2mcp MCP tools.
JSON Schema contract documents live in `spec/tools/schemas/*.json`.

## Tools

| Tool name | Status |
|-----------|--------|
| `dir2mcp_search` | stable |
| `dir2mcp_ask` | stable |
| `dir2mcp_open_file` | stable |
| `dir2mcp_list_files` | stable |
| `dir2mcp_stats` | stable |
| `dir2mcp_transcribe` | stable |
| `dir2mcp_annotate` | stable |
| `dir2mcp_transcribe_and_ask` | stable |
| `dir2mcp_ask_audio` | stable |

> **Note:** JSON Schema files for each tool are tracked in `spec/tools/schemas/` and are the authoritative machine-readable contracts. This markdown is an index only.

## Schema files

| Tool name | Schema file |
|-----------|-------------|
| `dir2mcp_search` | [`schemas/search.json`](schemas/search.json) |
| `dir2mcp_ask` | [`schemas/ask.json`](schemas/ask.json) |
| `dir2mcp_open_file` | [`schemas/open_file.json`](schemas/open_file.json) |
| `dir2mcp_list_files` | [`schemas/list_files.json`](schemas/list_files.json) |
| `dir2mcp_stats` | [`schemas/stats.json`](schemas/stats.json) |
| `dir2mcp_transcribe` | [`schemas/transcribe.json`](schemas/transcribe.json) |
| `dir2mcp_annotate` | [`schemas/annotate.json`](schemas/annotate.json) |
| `dir2mcp_transcribe_and_ask` | [`schemas/transcribe_and_ask.json`](schemas/transcribe_and_ask.json) |
| `dir2mcp_ask_audio` | [`schemas/ask_audio.json`](schemas/ask_audio.json) |

## Schema authoring rules

1. All tool names are prefixed with `dir2mcp_` to namespace them within the ecosystem.
2. Each schema file is a contract document with two top-level sections:
   - `input`: JSON Schema Draft-07 for tool arguments
   - `output`: JSON Schema Draft-07 for tool result payload
3. Generic validators should validate `input` and `output` sections explicitly (the root is a wrapper document, not a direct instance schema).
4. Breaking changes to any schema require a major version bump in `spec/versioning.md`.
5. Additive-only changes (new optional fields) require a minor version bump.
