# Tool Schemas

**Spec version:** `0.4.0`
**MCP protocol target:** `2025-11-25`

This document defines the canonical input/output schemas for all dir2mcp MCP tools.
JSON Schema definitions live alongside this file in `spec/tools/*.schema.json`.

## Tools

| Tool name | Status |
|-----------|--------|
| `dir2mcp.search` | stable |
| `dir2mcp.ask` | stable |
| `dir2mcp.open_file` | stable |
| `dir2mcp.list_files` | stable |
| `dir2mcp.stats` | stable |
| `dir2mcp.transcribe` | stable |
| `dir2mcp.annotate` | stable |
| `dir2mcp.transcribe_and_ask` | stable |
| `dir2mcp.ask_audio` | stable |

> **Note:** JSON Schema files for each tool are tracked in this directory and are the authoritative machine-readable contracts. This markdown is an index only.

## Schema authoring rules

1. All tool names are prefixed with `dir2mcp.` to namespace them within the ecosystem.
2. Input schemas use JSON Schema Draft-07.
3. Output schemas are documented in `spec/tools/<tool>.output.schema.json`.
4. Breaking changes to any schema require a major version bump in `spec/versioning.md`.
5. Additive-only changes (new optional fields) require a minor version bump.
