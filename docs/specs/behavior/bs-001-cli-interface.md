# bs-001: CLI interface

- **ID:** bs-001
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §2

## Scope

The `dir2mcp` command-line surface: the subcommands, global and per-command
flags, process exit codes, and the startup interaction contract for `dir2mcp
up`. Config loading/merging semantics live in [df-001](../data-formats/df-001-connection-json.md)/[df-002](../data-formats/df-002-state-outputs.md);
provider/capability profiles referenced by preflight live in
[td-001](../techniques/td-001-provider-model.md).

## Specification (normative)

### Commands

- `dir2mcp up`
  Start MCP server and run indexing (incremental) in background.

- `dir2mcp status`
  Read state from disk and show progress.

- `dir2mcp ask "QUESTION"`
  Local convenience: runs RAG via the same engine (no MCP).

- `dir2mcp reindex`
  Force full rebuild.

- `dir2mcp config init`
  Interactive setup wizard (TTY default) that creates/updates `.dir2mcp.yaml`
  and configures secret sources.

- `dir2mcp config print`
  Print effective config (defaults + file + env + flags).

- `dir2mcp version`

### Global flags

- `--dir <path>`: root directory (default `.`)
- `--config <path>`: config file path (default: `./.dir2mcp.yaml`)
- `--state-dir <path>`: state directory (default: `<root>/.dir2mcp`)
- `--json`: NDJSON events for automation/logging
- `--non-interactive`: disable prompts; fail fast with actionable config
  instructions when required values are missing
- `--quiet`

### `up` flags

- `--listen <host:port>` (default `127.0.0.1:0`)
- `--mcp-path <path>` (default `/mcp`)
- `--public` (shortcut: bind `0.0.0.0` and require token)
- `--auth auto|none|file:<path>`
  Warning: do not pass bearer tokens on the command line — see
  [bs-009 (secure token handling)](bs-009-security-safety.md).
- `--tls-cert <path> --tls-key <path>`
- `--x402 off|on|required` (default `off`)
- `--x402-facilitator-url <url>`
- `--x402-resource-base-url <url>` (public base URL used in payment requirements)
- `--x402-network <network-id>` (e.g., `eip155:8453`)
- `--x402-price <value>` (default per-call price for paid routes)
- `--read-only` (dir2mcp is read-only by design; this hardens future additions)

### Exit codes

| Code | Meaning |
| ---- | ------- |
| `0`  | success |
| `1`  | generic error |
| `2`  | config invalid |
| `3`  | ingestion error (fatal; per-file errors remain non-fatal) |
| `4`  | server startup error (bind/listen/runtime startup failure) |
| `5`  | auth/payment error |
| `6`  | signal/interrupt |

### Startup interaction contract (`up`)

- **Fast happy path:** if config requirements are already satisfied, `dir2mcp
  up` MUST not prompt.
- **Interactive by default on TTY:** if required config is missing and
  stdin/stdout is a TTY, `up` SHOULD run a guided setup flow.
- **Scriptable always:** every prompted value MUST have an env/flag/config
  equivalent.
- **Non-interactive mode:** with `--non-interactive` (or non-TTY), missing
  required values MUST return exit code `2` with explicit remediation
  instructions.
- **Server-first semantics:** server starts immediately when preflight
  requirements are satisfied. If required credentials for enabled
  ingestion/retrieval paths are missing, setup/validation runs before bind.
- **Prompt masking:** secret inputs (API keys/tokens) MUST be masked and never
  echoed.
- **Preflight checks (minimum),** evaluated per capability against its selected
  (or auto-selected) provider profile (see [td-001](../techniques/td-001-provider-model.md)):
  - embeddings (required) → requires the embed provider's credential **or
    connector**; if no eligible profile (a credential is present, or a
    credential-less connector such as a local endpoint — see
    [td-001](../techniques/td-001-provider-model.md)) can serve `embed`, preflight
    fails
  - OCR enabled for present/targeted PDFs/images → requires the OCR provider's
    credential/connector
  - STT enabled → requires the selected STT provider's credentials/connectors
- **Prompt parity examples:**
  - provider credential → the profile's env var (for example `MISTRAL_API_KEY`,
    `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`) or config-managed secret source
  - STT provider credentials → provider-specific env vars or secret source
  - OCR/transcription enablement → config keys under `ingest.*` and `stt.*`

## Changelog

- **0.1.0** — Migrated from SPEC.md §2. Rewired cross-references from mutable
  section numbers to stable doc IDs: §17 (secure token handling) → bs-009; §8.1
  / §8.1.1 / §8.1.3 (provider profiles, auto-selection) → td-001. Reformatted
  the exit-code list as a table; no normative requirements changed, added, or
  weakened.
