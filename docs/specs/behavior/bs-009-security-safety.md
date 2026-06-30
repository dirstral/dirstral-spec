# bs-009: Security & safety requirements

- **ID:** bs-009
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §17

## Scope

The minimum security and safety obligations of a dir2mcp server: filesystem
root isolation, symlink and archive traversal safety, authentication and token
handling, secret handling, HTTP `Origin` enforcement, and sensitive-file
exclusion. These requirements are normative for any conforming server,
independent of CLI surface (bs-001). Filesystem layout and state outputs
referenced here live in [df-001](../data-formats/df-001-connection-json.md) /
[df-002](../data-formats/df-002-state-outputs.md); error codes in
[df-008](../data-formats/df-008-error-taxonomy.md).

## Specification (normative)

### Root isolation

- Reject any `rel_path` resolving outside root with `PATH_OUTSIDE_ROOT`
  ([df-008](../data-formats/df-008-error-taxonomy.md)).

### Symlink policy

- Default **no-follow**; a server MAY follow a symlink only if the resolved
  target is under root.

### Archive safety

- Prevent zip-slip / path traversal within archives (a member MUST NOT resolve
  outside root).

### Auth

- A bearer token is **required by default** (the `--public` exception and its
  `--force-insecure` override are normative in bs-011).
- Tokens MUST **not** be passed on the command line; arguments may be exposed to
  other users or processes.
- Use `--auth file:<path>` to point to a user-provided token file with
  restrictive permissions (the auto-generated `.dir2mcp/secret.token` is created
  with `0600`, but any secured path works).
- Alternatively, set `DIR2MCP_AUTH_TOKEN` for environment-based tokens.
- The token file path and the environment variable are **equivalent sources**;
  `--auth file:` tells dir2mcp where to read the token.
- Config parity: `security.auth.token_file` specifies a file path and
  `security.auth.token_env` specifies an environment variable name; when either
  is set in `.dir2mcp.yaml` the behavior is equivalent (providing a token from
  the named source), but they refer to different **source types**.

### Secret handling

- Secret input in interactive prompts MUST be masked.
- Plaintext secrets MUST **never** be written to logs, terminal progress lines,
  NDJSON events, or config snapshots.
- Preferred storage is the OS keychain when available; file storage is the
  fallback and MUST enforce `0600` permissions.
- If `secrets.provider=session`, secrets are process-memory only and are
  discarded at exit.

### Origin checks

- If an `Origin` header is present, enforce the allowlist.

### Sensitive-file defaults

- Default excludes include the secret regex patterns defined in the configuration
  spec (the `security.secret_patterns` defaults), and these patterns are
  configurable via `security.secret_patterns`.
- The exclusion engine is consulted on **every** file access, including
  `open_file` ([bs-004](bs-004-mcp-transport.md)); tool handlers MUST reject or return an
  empty result for any path or content matching the configured patterns or path
  excludes.

## Changelog

- **0.1.0** — Migrated from SPEC.md §17, preserving every normative MUST/SHOULD/MAY
  verbatim in meaning. Cross-references rewired to stable doc IDs:
  `PATH_OUTSIDE_ROOT` / error codes (§14) → [df-008](../data-formats/df-008-error-taxonomy.md);
  base/filesystem layout (§1/§4) → df-000 / df-001 / df-002; `open_file` and the
  exclusion engine (§10) → [bs-004](bs-004-mcp-transport.md); CLI surface (§2) →
  bs-001; the `--public` / `--force-insecure` auth-default exception (§16) → bs-011.
  The §17 narrative cross-ref to the §7.2
  secret-pattern defaults was rewritten to point at the `security.secret_patterns`
  configuration surface, since §7 has no migrated doc ID yet — *drift note: when §7
  (configuration) is migrated, re-point this reference to its stable doc ID.*
