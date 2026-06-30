# df-002: State-directory outputs

- **ID:** df-002
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §4.1, §4.2, §4.4 (migrated; `connection.json` is [df-001](df-001-connection-json.md))

## Scope

Everything `dir2mcp` writes under the state directory besides
`connection.json` (df-001): the directory **layout**, `secret.token`, and
`corpus.json`. The state directory is always **local** ([df-000](df-000-base.md)
invariant), even when the corpus root is remote.

## Specification (normative)

### 4.1 Layout

All state lives under `<state-dir>` (default `<root>/.dir2mcp/`):

```
.dir2mcp/
  .dir2mcp.yaml.snapshot        # effective config snapshot (resolved values)
  connection.json               # connect info (df-001; no session id)
  secret.token                  # bearer token (0600)
  meta.sqlite                   # metadata store (documents/reps/chunks/spans — df-003)
  vectors_text.hnsw             # ANN index for text-like chunks
  vectors_code.hnsw             # ANN index for code chunks
  corpus.json                   # profile + progress summary
  ingest.log                    # optional
  cache/
    ocr/                        # cached OCR outputs (optional)
    transcribe/                 # cached transcripts (optional)
    annotations/                # cached annotation JSON (optional)
  payments/
    pricing.snapshot.json       # effective price policy (optional)
    settlement.log              # payment verification/settlement outcomes (optional)
  locks/
    index.lock
```

The config snapshot (`.dir2mcp.yaml.snapshot`) holds **resolved** config values
and **MUST NOT** contain credentials (secrets are never persisted to the
snapshot; see bs-011).

### 4.2 `secret.token`

- Contains a single bearer-token line.
- Permissions **MUST** be restrictive (`0600` on Unix-like systems).
- This is the token embedded in `connection.json` (df-001); rotating it
  invalidates that file's `Authorization` value.

### 4.4 `corpus.json`

A lightweight profile + progress summary:

```json
{
  "root": "/abs/root",
  "profile": {
    "doc_counts": { "code": 120, "md": 35, "pdf": 9, "audio": 3, "image": 14, "other": 7 },
    "code_ratio": 0.62
  },
  "models": {
    "embed_provider": "mistral",
    "embed_text": "mistral-embed",
    "embed_code": "codestral-embed",
    "ocr_provider": "mistral-ocr",
    "ocr": "mistral-ocr-latest",
    "stt_provider": "mistral",
    "stt_model": "voxtral-mini-latest",
    "chat_provider": "mistral",
    "chat": "mistral-small-2506"
  },
  "indexing": {
    "job_id": "job_...",
    "running": true,
    "scanned": 412,
    "indexed": 55,
    "skipped": 340,
    "deleted": 2,
    "representations": 88,
    "chunks_total": 1480,
    "embedded_ok": 920,
    "errors": 1
  }
}
```

> The `models` block shows the built-in Mistral default profile; the values are
> whatever providers are actually bound per capability (td-001), not a fixed set.

#### The `-1` "unavailable" sentinel

When indexing stats are unavailable (e.g. the `ListFiles`-only fallback path
where no live `IndexingState` is present), the fields `representations`,
`chunks_total`, and `embedded_ok` are set to `-1` to signal **"not derivable"**.
A value of `-1` is **not** an error: consumers **MUST** treat it as "data
unavailable" and **MUST NOT** treat it as a counter value.

Fallback-path example:

```json
{
  "root": "/abs/root",
  "profile": { "doc_counts": { "code": 120, "md": 35 }, "code_ratio": 0.77 },
  "models": { "embed_provider": "mistral", "embed_text": "mistral-embed", "embed_code": "codestral-embed", "ocr_provider": "mistral-ocr", "ocr": "mistral-ocr-latest", "stt_provider": "mistral", "stt_model": "voxtral-mini-latest", "chat_provider": "mistral", "chat": "mistral-small-2506" },
  "indexing": {
    "job_id": "", "running": false, "mode": "incremental",
    "scanned": 155, "indexed": 120, "skipped": 35, "deleted": 0,
    "representations": -1, "chunks_total": -1, "embedded_ok": -1, "errors": 0
  }
}
```

> Per [df-000](df-000-base.md), `corpus.json` SHOULD also carry a
> `format_version`; it is the same cross-version signal mandated for
> `connection.json` and the `stats` tool output.

## Changelog

- **0.1.0** — Migrated from SPEC.md §4.1/§4.2/§4.4. Cross-referenced
  `connection.json` → df-001, `meta.sqlite` → df-003, the model profile → td-001;
  noted the snapshot MUST-not-hold-secrets rule and the df-000 `format_version`
  recommendation.
