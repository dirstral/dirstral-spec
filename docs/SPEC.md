# SPEC.md
## dir2mcp Output & Integration Specification (Go)

> **Restructure in progress (dirstral-spec#24).** This monolith is being split
> into numbered, independently-versioned documents under
> [`docs/specs/`](specs/README.md) (OONI-style), so spec references stop relying
> on mutable section numbers. Migrated documents are authoritative once marked
> **Stable** there; until then this file remains the source of truth. See the
> [document index](specs/README.md) and the [migration map](specs/MIGRATION.md).
> **All three classes are now drafted** — 9 `df-*` (data formats), 11 `bs-*`
> (behavior), 5 `td-*` (techniques), one per SPEC.md section (§2–§18). The df-007
> migration also reconciled `spec/tools/schemas/common.json` to the
> implementation, fixing the published-schema drift (dir2mcp #423). Only the
> non-normative §19 (non-goals) and §20 (implementation guidance) remain. These
> docs are **Draft**; this file stays authoritative until each is reviewed and
> marked **Stable**.

**Spec version:** `0.36.0`  
**MCP protocol target:** `2025-11-25` (Streamable HTTP transport, sessions, tools, structured tool output)  
**Primary goal:** one-command “deploy-now” directory RAG exposed as an **MCP Streamable HTTP** server, with an embedded on-disk index by default (**zero external infra required beyond model providers**; an external vector store MAY be configured but is never required — §6) and a single config file.  
**Implementation goal:** a **provider-agnostic** model pipeline (embeddings, chat/RAG, OCR, STT, rerank) where each capability binds to a configurable provider profile. An OpenAI-compatible adapter is the backbone for chat + embeddings (OpenAI, OpenRouter, Groq, Azure, local Ollama/vLLM, **and Mistral**); bespoke adapters cover genuinely non-OpenAI surfaces (Mistral OCR, Anthropic, Cohere rerank, ElevenLabs). Mistral is the default profile but not privileged. See [Design 0001](design/0001-multi-provider.md).  
**Scope note:** x402 support is optional and additive; retrieval and MCP interoperability remain first-class regardless of payment mode.

---

## 0) Executive summary

`dir2mcp up` in any directory will:

1) Start an MCP server immediately (connect right away).  
2) Index the directory in the background (incremental, safe-by-default).  
3) Normalize non-text files into text representations:
   - PDFs/images → OCR markdown
   - audio → transcripts (STT provider configurable)
   - structured extraction → JSON + flattened text (on-demand by default)
4) Provide a small MCP tool surface for agents:
   - search, ask, open_file, list_files, stats
   - plus (recommended) transcribe, annotate, transcribe_and_ask
5) Optionally enable native x402 on selected routes/tools:
  - return HTTP 402 with payment requirements for unpaid requests (`PAYMENT-REQUIRED`)
  - accept client payment proofs via `PAYMENT-SIGNATURE`
  - verify/settle via configured facilitator
  - include settlement receipt metadata via `PAYMENT-RESPONSE` on successful paid calls
  - continue serving standard MCP responses after successful payment

### 0.1 External normative references for x402 mode

When `x402` mode is enabled, implementations SHOULD align with these references:

- x402 v2 specification: <https://github.com/coinbase/x402/tree/main/specs>
- CDP x402 facilitator API reference (`/v2/x402/verify`, `/v2/x402/settle`): <https://docs.cdp.coinbase.com/api-reference/v2/rest-api/x402-facilitator/x402-facilitator>
- x402 core flow and headers (`PAYMENT-REQUIRED`, `PAYMENT-SIGNATURE`, `PAYMENT-RESPONSE`):
  - <https://docs.cdp.coinbase.com/x402/core-concepts/how-it-works>
  - <https://docs.cdp.coinbase.com/x402/core-concepts/http-402>
  - <https://docs.cdp.coinbase.com/x402/migration-guide>
- Network and facilitator support (CAIP-2 identifiers): <https://docs.cdp.coinbase.com/x402/network-support>
- Bazaar discovery extension model: <https://docs.cdp.coinbase.com/x402/bazaar>

### 0.2 Implementation status notes (June 2026)

Status tags used in this spec:

- **Implemented:** available in current repository/runtime behavior.
- **Partially implemented:** interface exists, but not all target behavior is complete.
- **In progress:** work underway but not yet complete (may overlap with "partially implemented").
- **Planned:** target behavior not yet fully implemented.

Current high-level status:

- CLI + MCP server lifecycle, indexing pipeline, and core tool surface: **Implemented**
- Multimodal ingestion (OCR/transcription/annotation) and retrieval workflows: **Implemented** (with ongoing quality/perf hardening)
- Native multimodal embeddings (`gemini-embedding-2`, §8.1.7 — images/PDF/audio/video into one shared space; retrieval dedup + `open_file` `MEDIA_NO_TEXT` + ask grounding): **Implemented**, default-off; **GA-gated** (model is Public Preview, so the compatibility row stays pending until release against the GA-verified model)
- Retrieval `Stats()` service contract: **Implemented** (issue #71 closed)
- Retrieval answer generation path (`Engine.Ask()` / `AskWithContext`): **Implemented** (issue #70 closed)
- Native x402 tools/call gating path: **Implemented** (optional and facilitator-backed)
- Hosted smoke/runbook guidance: **Implemented** (see issue #19)
- Release-completion checklist hardening: **In progress** (see issue #12)

---

## 1) Definitions and invariants

### 1.1 Terms
- **Root directory**: directory being indexed.
- **State directory**: storage location for index state (default: `<root>/.dir2mcp/`).
- **Document**: ingestible unit (file or archive member).
- **Representation (rep)**: a text view derived from a document:
  - `raw_text` (code/text/md/data/html converted to text)
  - `extracted_markdown` (extractor output for PDFs/images/documents; formerly `ocr_markdown`)
  - `transcript` (STT output for audio)
  - `annotation_json` (structured JSON result)
  - `annotation_text` (flattened `key: value` text derived from annotation_json)
- **Chunk**: span of a representation used for embedding and retrieval.
- **Span**: provenance coordinates for citations: line range, page number, or time range.

### 1.2 Invariants
- The MCP server accepts lifecycle requests immediately after `dir2mcp up` prints the endpoint URL.
- Indexing continues in the background; tools operate on partial index if needed.
- No content outside root is accessible via tools (no path traversal; no symlink escape).
- The default vector index is **embedded/on-disk** and requires **no external service**.
- An external vector store MAY be configured (§6, Tier C) but MUST NOT be required: a conforming deployment MUST be able to run with **zero external infrastructure beyond the model providers** (the embedded default).
- The state directory is always **local**, even when the corpus root is remote (§7.8): SQLite metadata, the embedded index, and caches never live on the remote source.

### 1.3 `format_version` (cross-version signal)

Every self-describing payload dir2mcp writes at a boundary **MUST** or **SHOULD**
(as specified below) carry a `format_version` string (semver, e.g. `"0.1.0"`), so
an independent implementation always knows the payload shape it is reading and can
adapt or reject rather than silently mis-parse (dir2mcp #468; folds in #404/#405).
`format_version` is an **independent** payload-shape version — it is **not** the
`Spec version` of this document and does not track it.

- **`connection.json`** (§4.3) **MUST** include `format_version`.
- **`dir2mcp_stats` output** (§15.6) **SHOULD** include `format_version`. When it
  is absent, a client **MUST** treat the payload as a pre-`format_version`
  (baseline) shape and **MUST NOT** infer the payload shape from
  `protocol_version` — that value is the pinned MCP protocol generation
  (`2025-11-25`), orthogonal to and unaffected by payload evolution (e.g. a
  `Hit`/`Citation` change).
- **SQLite** (§5.6) **MUST** set `PRAGMA user_version` to a monotonic schema
  version and check it on open: a database written by a **newer** schema than the
  binary understands **MUST** be refused with a clear error, and any non-additive
  migration **MUST** be gated on that version (closes #405). `PRAGMA user_version`
  is an **independent** monotonic integer for the on-disk schema; it does **not**
  map to the semver `format_version` of the wire payloads above.
- **MCP `initialize`** (§11.2) **MUST** advertise this spec's pinned
  `protocolVersion` (`2025-11-25`) rather than echoing the client's requested
  value (closes #404).

A consumer that encounters a **major**-incompatible `format_version` **MUST** fail
closed (reject), not silently mis-parse.

Additive (minor/patch) changes **MUST** be backward-compatible, and this **MUST**
hold even for the closed schemas (`additionalProperties: false`) that df-007
mandates for tool outputs. That compatibility rests on a producer invariant, not
on lenient consumers: a producer **MUST NOT** emit a field its own
**advertised** schema omits, so a new optional field is added to the relevant
schema — and, for a tool, to the `outputSchema` the server advertises via
`tools/list` — in the **same** spec version that first emits it (this is why
`format_version` itself ships in the 0.34.0 stats schema, not ahead of it). A
conformant client therefore validates each payload against the **server-advertised
schema** (equivalently, the schema for the payload's declared `format_version`),
never a copy of an older schema pinned out of band; validated that way, a new
optional field is present in the schema and accepted. Tolerant readers that do
not enforce a closed schema simply ignore unknown fields; either way an additive
change never rejects a payload. New optional fields are absent on older producers,
and their absence carries meaning only where this spec assigns one.

The canonical, independently-versioned home for these conventions is
[df-000](specs/data-formats/df-000-base.md); this section stays authoritative
until that document is marked **Stable**.

---

## 2) CLI interface

### 2.1 Commands
- `dir2mcp up`  
  Start MCP server and run indexing (incremental) in background.

- `dir2mcp status`  
  Read state from disk and show progress.

- `dir2mcp ask "QUESTION"`  
  Local convenience: runs RAG via the same engine (no MCP).

- `dir2mcp reindex`  
  Force full rebuild.

- `dir2mcp config init`  
  Interactive setup wizard (TTY default) that creates/updates `.dir2mcp.yaml` and configures secret sources.

- `dir2mcp config print`  
  Print effective config (defaults + file + env + flags).

- `dir2mcp version`

### 2.2 Global flags
- `--dir <path>`: root directory (default `.`)
- `--config <path>`: config file path (default: `./.dir2mcp.yaml`)
- `--state-dir <path>`: state directory (default: `<root>/.dir2mcp`)
- `--json`: NDJSON events for automation/logging
- `--non-interactive`: disable prompts; fail fast with actionable config instructions when required values are missing
- `--quiet`

### 2.3 `up` flags
- `--listen <host:port>` (default `127.0.0.1:0`)
- `--mcp-path <path>` (default `/mcp`)
- `--public` (shortcut: bind `0.0.0.0` and require token)
- `--auth auto|none|file:<path>`  
  Warning: do not pass bearer tokens on the command line—see Section 17 for secure token handling.
- `--tls-cert <path> --tls-key <path>`
- `--x402 off|on|required` (default `off`)
- `--x402-facilitator-url <url>`
- `--x402-resource-base-url <url>` (public base URL used in payment requirements)
- `--x402-network <network-id>` (e.g., `eip155:8453`)
- `--x402-price <value>` (default per-call price for paid routes)
- `--read-only` (dir2mcp is read-only by design; this hardens future additions)

### 2.4 Exit codes
- `0` success
- `1` generic error
- `2` config invalid
- `3` ingestion error (fatal; per-file errors remain non-fatal)
- `4` server startup error (bind/listen/runtime startup failure)
- `5` auth/payment error
- `6` signal/interrupt

### 2.5 Startup interaction contract (`up`)

- Fast happy path: if config requirements are already satisfied, `dir2mcp up` MUST not prompt.
- Interactive by default on TTY: if required config is missing and stdin/stdout is a TTY, `up` SHOULD run a guided setup flow.
- Scriptable always: every prompted value MUST have an env/flag/config equivalent.
- Non-interactive mode: with `--non-interactive` (or non-TTY), missing required values MUST return exit code `2` with explicit remediation instructions.
- Server-first semantics: server starts immediately when preflight requirements are satisfied. If required credentials for enabled ingestion/retrieval paths are missing, setup/validation runs before bind.
- Prompt masking: secret inputs (API keys/tokens) MUST be masked and never echoed.
- Preflight checks (minimum), evaluated per capability against its selected (or auto-selected) provider profile (see 8.1):
  - embeddings (required) -> requires the embed provider's credential **or connector**; if no eligible profile (a credential is present, or a credential-less connector such as a local endpoint — see 8.1.1/8.1.3) can serve `embed`, preflight fails
  - OCR enabled for present/targeted PDFs/images -> requires the OCR provider's credential/connector
  - STT enabled -> requires the selected STT provider's credentials/connectors
- Prompt parity examples:
  - provider credential -> the profile's env var (for example `MISTRAL_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`) or config-managed secret source
  - STT provider credentials -> provider-specific env vars or secret source
  - OCR/transcription enablement -> config keys under `ingest.*` and `stt.*`

---

## 3) CLI output contract

### 3.1 Human output (default)
On `dir2mcp up`, stdout MUST print:

1) Index/state location + mode  
2) MCP connection block (URL, headers, where token is stored)  
3) Progress lines during indexing

Example (illustrative):
```txt
Index: /repo/.dir2mcp  (meta.sqlite + vectors_text.hnsw + vectors_code.hnsw)
Mode: incremental  (server-first; indexing in background)

MCP endpoint:
  URL:    http://127.0.0.1:52143/mcp
  Auth:   Bearer (source=file, from --auth file:/run/secrets/dir2mcp.token)
  Headers:
    MCP-Protocol-Version: 2025-11-25
    Authorization: Bearer <token>
    MCP-Session-Id: (assigned after initialize response)

Progress: scanned=412 indexed=55 skipped=340 deleted=2 reps=88 chunks=1480 embedded=920 errors=1
```

Progress line fields (minimum):

* `scanned`, `indexed`, `skipped`, `deleted`
* `reps` (representations created/updated)
* `chunks` (chunks total known/created)
* `embedded` (chunks embedded successfully)
* `errors` (non-fatal per-document failures)

### 3.2 NDJSON output (`--json`)

Emit NDJSON (one JSON object per line), schema:

```json
{
  "ts": "2026-02-25T12:34:56.789Z",
  "level": "info|warn|error",
  "event": "index_loaded|server_started|connection|scan_progress|embed_progress|file_error|file_skip|payment_required|payment_verified|payment_settled|payment_failed|fatal",
  "data": {}
}
```

Required events for `up`:

* `index_loaded`
* `server_started`
* `connection` (endpoint + headers + token reference)
* periodic `scan_progress` and `embed_progress`
* `file_error` for per-document failures (non-fatal)
* `file_skip` for per-document skips (never-indexed documents)
* if x402 is enabled: `payment_required`, `payment_verified`, `payment_settled`, `payment_failed`

`file_error` is emitted **per document**, at `level: "error"`, once for each
document set to `status="error"` during ingest. It MUST NOT be used to report a
whole-run fatal failure — that is what `fatal` is for. `file_error.data` MUST
include:

* `rel_path` — the document's corpus-relative path
* `doc_type`
* `message` — single-line, length-capped, secret-redacted per §15.6. The NDJSON
  stream is an untrusted sink (it is commonly redirected to a log file or piped
  to another process), so the same redaction that applies to
  `recent_failures.error_message` applies here.

`file_skip` is emitted **per document**, at `level: "warn"`, once for each
document set to `status="skipped"` during ingest. It is the streaming
counterpart of the `skip_reasons` honest-coverage aggregate (§15.2) — the
aggregate answers "what was not indexed, in total"; the event answers "which
file, right now". `file_skip.data` MUST include:

* `rel_path`
* `doc_type`
* `reason` — a value from the `skip_reasons` enum (§15.2): `unsupported_format`,
  `binary_ignored`, `archive`, `ignore_rule`, `secret_excluded`, `path_excluded`,
  `size_cap`. New reasons are added only by a minor version bump; a client MAY
  receive an unrecognized value from a newer server and SHOULD render it
  verbatim rather than error.

A skipped document MUST NOT also produce a `file_error`, and vice versa: the two
events partition the never-indexed set. The count of `file_skip` events emitted
for a run equals the run's terminal `indexing.skipped`.

`connection.data` must include:

* `transport: "mcp_streamable_http"`
* `url`
* `headers` (include MCP-Protocol-Version, Authorization placeholder)
* `token_source` (`secret.token|env|file`)

If `--auth file:<path>` is used, `token_source` MUST be `file`, and the connection payload SHOULD include either `token_file` (preferred) or `token_source_details.path` to distinguish user-provided token files from auto-generated `.dir2mcp/secret.token`.

Example `connection.data` (file-auth mode):

```json
{
  "transport": "mcp_streamable_http",
  "url": "http://127.0.0.1:52143/mcp",
  "headers": {
    "MCP-Protocol-Version": "2025-11-25",
    "Authorization": "Bearer <token>"
  },
  "token_source": "file",
  "token_file": "/run/secrets/dir2mcp.token"
}
```

### 3.3 Non-interactive missing-config error contract

When required config is missing and prompts are disabled (`--non-interactive` or non-TTY), CLI output MUST be actionable and MUST NOT print secret values.

Example:

```txt
ERROR: CONFIG_INVALID: Missing MISTRAL_API_KEY
Set env: MISTRAL_API_KEY=...
Or run: dir2mcp config init
```

### 3.4 Hosted demo smoke probe (operational runbook)

For a hosted endpoint readiness check, use `./scripts/smoke_hosted_demo.sh`.
Set `DIR2MCP_DEMO_TOKEN` whenever the hosted MCP endpoint enforces auth
(bearer token required). It is optional only for deployments with no auth; if
you omit it against an auth-enabled endpoint the script can fail early (for
example with HTTP `401`) before the MCP pass conditions below are evaluated.

```bash
DIR2MCP_DEMO_URL="https://your-host.example/mcp" \
DIR2MCP_DEMO_TOKEN="<optional-bearer-token>" \
./scripts/smoke_hosted_demo.sh
```

Expected pass conditions:

* `initialize` returns HTTP `200` and includes `MCP-Session-Id`.
* `tools/list` returns HTTP `200` and includes tool metadata.
* `tools/call` against `dir2mcp_list_files` returns either:
  * HTTP `200` with JSON-RPC body, or
  * HTTP `402` with `PAYMENT-REQUIRED` when x402 route gating is enabled.

---

## 4) On-disk outputs (state)

All state lives under `<state-dir>` (default `<root>/.dir2mcp/`).

### 4.1 Layout

```
.dir2mcp/
  .dir2mcp.yaml.snapshot        # effective config snapshot (resolved values)
  connection.json               # connect info (no session id; assigned at initialize)
  secret.token                  # bearer token (0600)
  meta.sqlite                   # metadata store (documents/reps/chunks/spans)
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

### 4.2 `secret.token`

* Contains a single bearer token line.
* Permissions MUST be restrictive (0600 on Unix-like systems).

### 4.3 `connection.json`

Written on `up`. It **MUST** carry `format_version` (§1.3) so a reader can detect
an incompatible payload shape before connecting:

```json
{
  "format_version": "0.1.0",
  "transport": "mcp_streamable_http",
  "url": "http://127.0.0.1:52143/mcp",
  "headers": {
    "MCP-Protocol-Version": "2025-11-25",
    "Authorization": "Bearer <token-from-secret.token>"
  },
  "session": {
    "uses_mcp_session_id": true,
    "header_name": "MCP-Session-Id",
    "assigned_on_initialize": true
  }
}
```

### 4.4 `corpus.json`

A lightweight summary:

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
    "errors": 1,
    "watch_overflows": 0
  }
}
```

`watch_overflows` (optional, additive; dir2mcp #591) is the count of fsnotify
kernel event-buffer overflows observed over the file-watcher's lifetime. A
non-zero value means a burst of filesystem changes exceeded the kernel buffer and
some events were dropped, so those changes were reconciled by a periodic safety
rescan rather than per-event — a signal that the index may briefly lag reality
after heavy churn. It is omitted when the server is not running the watcher (e.g.
a one-shot index) or on platforms without overflow reporting; consumers MUST treat
its absence as "unknown / not applicable", not zero.

When indexing stats are unavailable (e.g., the ListFiles-only fallback path where no live
`IndexingState` is present), the fields `representations`, `chunks_total`, and `embedded_ok`
are set to `-1` to signal "not derivable". A value of `-1` is **not** an error; consumers
MUST treat it as "data unavailable" and MUST NOT treat it as a counter value.

Example snapshot emitted via the ListFiles-only fallback path:

```json
{
  "root": "/abs/root",
  "profile": {
    "doc_counts": { "code": 120, "md": 35 },
    "code_ratio": 0.77
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
    "job_id": "",
    "running": false,
    "mode": "incremental",
    "scanned": 155,
    "indexed": 120,
    "skipped": 35,
    "deleted": 0,
    "representations": -1,
    "chunks_total": -1,
    "embedded_ok": -1,
    "errors": 0
  }
}
```

---

## 5) SQLite metadata schema (minimum semantics)

The exact SQL types may vary; semantics must match.

### 5.1 `documents`

* `doc_id` (PK)
* `rel_path` (unique, normalized `/`)
* `source_type` (`file|archive_member`)
* `doc_type` (`code|text|md|pdf|image|audio|video|data|html|archive|binary_ignored|...`)
* `size_bytes`
* `mtime_unix`
* `content_hash` (stable, e.g., blake3/sha256)
* `status` (`ok|skipped|error`)
* `error` (nullable)
* `deleted` (boolean; tombstone)
* `canonical_doc_id` (optional; `0`/self when the document is canonical, otherwise
  the `doc_id` of the canonical document this row is an **alias** of — §7.9)
* `is_alias` (optional boolean; `true` for a non-canonical member of a duplicate
  group — §7.9). Alias rows share the canonical `content_hash` and hold **no**
  representations, chunks, or embeddings.

### 5.2 `representations`

* `rep_id` (PK)
* `doc_id` (FK)
* `rep_type` (`raw_text|extracted_markdown|transcript|annotation_text|annotation_json`)
* `rep_hash` (stable; changes when rep changes)
* `created_unix`
* `meta_json` (must include provider/model for OCR/transcription/annotations when applicable)
* `deleted` (boolean; tombstone)

**Transcript meta_json requirements**

* `provider`: string — the STT/transcription provider. The enumeration is **not
  closed** to `mistral|elevenlabs`: any STT-capable provider (§8.2 — e.g.
  `openai`, `gemini`, a self-hosted `kind: openai` endpoint, §8.5) is valid.
* `model`: string
* `model_version`: optional — provider model version, part of the derivation
  identity (§8.6.7).
* `timestamps`: boolean — whether provider-authoritative timestamps are present.
* `timing`: optional — `provider` (default, provider-authoritative) or
  `estimated` (no-timestamp fallback, §8.6.1).
* `words`: optional — present when per-word timing was captured (a `words` array
  lives on the segment span's `extra_json`, §8.6.1; this flag records that the
  transcript carries word timing).
* `language`: optional — source language (auto-detected or pinned, §8.6.2).
* `source`: optional — `stt` (machine-transcribed) or `sidecar` (authored
  subtitle ingested per §8.6.4). Sidecar transcripts are not model-derived
  (§8.6.7).
* `duration_ms`: optional

A **translated** transcript additionally records:

* `source_language`: the language it was translated *from*.
* `translate_provider`: the translation provider.
* `translate_model`: the translation model.

A **diarized** transcript (§8.6.8) additionally records:

* `diarized`: boolean — whether speaker attribution is present on the transcript.
* `diarize_provider`: optional — the diarization-capable provider/backend that
  produced the speaker attribution (e.g. a WhisperX / pyannote-backed STT
  endpoint, §8.6.8). Part of the derivation identity (§8.6.7).
* `diarize_model`: optional — the diarization model/pipeline version.
* `speakers`: optional — an array of the distinct speaker identifiers present in
  the transcript (e.g. `["S1", "S2"]`), each optionally paired with a
  human-readable `label`. The per-segment attribution lives on the segment span's
  `extra_json.speaker` (§5.4).

**Detected-language metadata (any representation)**

Any representation MAY record the natural language of its content in `meta_json`,
independent of representation type — a `transcript` (§8.6.2), an
`extracted_markdown` from OCR, or a plain `raw_text` document — to enable
multilingual-corpus filtering and per-language retrieval (§9.5). The fields are
**optional and additive**; a representation that records none is treated as
**unknown language** (never an error). Detection is **best-effort** and MUST
degrade gracefully (§8.8).

* `language`: optional — the **effective** language of the representation as a
  BCP-47 language tag (e.g. `en`, `pt-BR`). This is the value matched by the
  retrieval language filter (§9.5). For a `transcript` this is the existing
  source-language field (§8.6.2); for other representation types it carries the
  same meaning (the language of the indexed text). Absent ⇒ unknown.
* `language_source`: optional — how `language` was obtained: `detected`
  (auto-detected, best-effort, §8.8), `configured` (pinned by an operator, e.g.
  `media.language` / per-provider `stt_language`, §16.2), or `declared`
  (asserted by the source itself, e.g. a sidecar's language suffix §8.6.4, a
  document language tag, or an OCR provider's reported language). Absent ⇒
  unspecified provenance.
* `language_confidence`: optional — a detector-reported confidence in `[0,1]`
  for an auto-`detected` language. Informational only; it MUST NOT by itself
  cause a representation to be treated as unknown (an implementation MAY apply a
  configured floor at detection time per §8.8, but the recorded `language`, once
  written, is authoritative for retrieval matching).

The **configured/expected** language (an operator pin) and the **detected**
language are distinct concepts: when both are known and they disagree, the
recorded `language` is the **effective** value the implementation chose to index
under (§8.8 defines the resolution), and `language_source` records which won. A
translated transcript's `language` is its **target** language, while its
`source_language` (above) records what it was translated *from* — both are
matchable per-language values (§9.5).

### 5.3 `chunks`

* `chunk_id` (PK; integer; used as ANN label)
* `rep_id` (FK)
* `ordinal`
* `text` (or compressed blob)
* `text_hash`
* `tokens_est` (approx)
* `index_kind` (`text|code`)  # routes to vectors_text or vectors_code
* `embedding_status` (`ok|pending|error`)
* `embedding_error` (nullable)
* `deleted` (boolean; tombstone)

### 5.4 `spans` (provenance for citations)

* `chunk_id` (FK)
* `span_kind` (`lines|page|time|region`)
* `start` (integer)  # start_line / page / start_ms / page (region)
* `end` (integer)    # end_line / page / end_ms / page (region)
* `extra_json` (nullable)  # speaker, confidence, section breadcrumb, bbox, etc.

For `time` spans on a **diarized** transcript (§8.6.8), `extra_json` MAY carry a
`speaker` field — a stable per-transcript speaker identifier (e.g. `"S1"`,
`"S2"`) — and MAY carry a human-readable `speaker_label` when one is known. The
`speaker` field is **optional and additive**: consumers that do not recognize it
MUST treat the span as un-attributed (degrade to a flat transcript citation).
Diarization is **off by default** and **provider-dependent** (§8.6.8); a
non-diarized transcript carries no `speaker` field.

The `region` span kind localizes a chunk to a rectangular area on a page.
For `region` spans, `start` and `end` carry the first and last page the
chunk's source elements appear on (equal when single-page), and
`extra_json` MUST carry the bounding box and SHOULD carry the section
breadcrumb:

```jsonc
{
  "bbox": { "page": 1, "l": 72.0, "t": 90.5, "r": 523.0, "b": 410.2, "coord_origin": "TOPLEFT" },
  "section": ["Chapter 2", "2.1 Background"],   // heading breadcrumb, outermost first
  "label": "paragraph",                          // a single value (see enum below)
  "charspan": [120, 884]                          // optional, char offsets into the source element
}
```

* `label` is a **single** discrete value, not a pipe-delimited set. It MUST be
  one of: `paragraph`, `section_header`, `list_item`, `table`, `caption`,
  `code`, `formula`, `picture`. When a chunk aggregates elements of mixed
  labels, the label of the chunk's dominant (first/longest) element is used.
* `bbox` coordinates are in the source document's point space. `coord_origin`
  is `TOPLEFT` or `BOTTOMLEFT`; implementations SHOULD normalize to `TOPLEFT`
  and record the origin actually stored.
* `bbox.page` is the **primary page**: the page of the chunk's first source
  element in reading order. It MUST satisfy `start ≤ bbox.page ≤ end`. For a
  single-page chunk `start == end == bbox.page`.
* When a chunk aggregates multiple source elements, `extra_json.bbox` is the
  union (smallest enclosing rectangle) of only those elements **on the primary
  page**; elements on other pages within `start..end` contribute to the page
  range but not to the rectangle. A single bounding box never spans pages.
* `region` spans are produced by structured document extraction (§7.4.B).
  Extractors that emit only page-separated text continue to use `page` spans.

### 5.5 `settings`

* `key`, `value` for:

  * `protocol_version` = `2025-11-25`
  * `corpus_id`
  * `index_format_version`
  * `embed_provider`, `embed_base_url`   # embed_base_url = normalized per §8.1.4; "" is a valid value (pre-existing indexes and non-meaningful/default endpoints)
  * `embed_text_model`, `embed_text_dim`
  * `embed_code_model`, `embed_code_dim`
  * `ocr_model`
  * `stt_provider`, `stt_model`
  * `chat_model`

### 5.6 Schema version fence (`PRAGMA user_version`)

The on-disk metadata database **MUST** carry a monotonic schema version and
enforce it on open (§1.3; closes #405):

* On create/migrate, the daemon **MUST** set `PRAGMA user_version` to the integer
  schema version its binary implements.
* On open, it **MUST** read `PRAGMA user_version`. If the stored value is
  **greater** than the binary's known schema version, the database was written by
  a newer build; the daemon **MUST** refuse to open it with a clear, actionable
  error (naming the found vs. supported version) rather than operate on a shape it
  does not understand. A **lower** stored value selects the appropriate forward
  migration.
* Any **non-additive** migration (a column/table drop or a type/semantics change,
  as opposed to a purely additive column) **MUST** be gated on this version and
  **MUST** bump it, so a downgrade lands on the refusal path above instead of
  silently corrupting a mixed-version database.

`PRAGMA user_version` is an integer for the **on-disk schema** and is independent
of both the wire `format_version` (§1.3) and the settings-row
`index_format_version` (§5.5, which fences **index/embedding** compatibility, not
table shape).

---

## 6) Vector index backends and identity

The vector index is selected by `index.backend` (§16.2). The **default** backend
is embedded and requires no external service (§1.2); an external store MAY be
selected but is **optional, never required**. Whatever backend is chosen, the two
logical axes and the embed-identity binding below are invariant.

### 6.1 Logical axes (text/code)

Independent of backend, vectors are partitioned into two logical axes:

* **text** axis: embeddings for `index_kind=text` chunks (raw text, OCR/extracted markdown, transcripts, annotation_text, and — under §8.1.7 — media chunks).
* **code** axis: embeddings for `index_kind=code` chunks (source code and code-like configs).

Dimensions MAY differ between axes; each axis MUST be internally consistent. The
**label / payload key** for every vector MUST be its `chunk_id` (integer), so a
query result maps directly to chunk metadata. In the embedded backends the two
axes are the two on-disk files (`vectors_text.*`, `vectors_code.*`); in an
external store they are two collections/namespaces (§6.3).

### 6.2 Backend tiers (`index.backend`)

| `index.backend` | Tier | Description | External infra | Default |
|---|:--:|---|:--:|:--:|
| `memory` | **A** | In-memory HNSW, **pure-Go**, persisted/snapshotted to the local state dir | none | **✅ default** |
| `disk`   | **B** | Pure-Go on-disk / memmapped single-node index in the local state dir | none | |
| `qdrant` | **C** | External Qdrant collection | required | |
| `pgvector` | **C** | External PostgreSQL + pgvector | required | |

* **Tier A (`memory`, default)** — an in-memory HNSW graph built in pure Go,
  snapshotted to the local state dir (`vectors_text.*` / `vectors_code.*`) so it
  survives restarts. Requires no external service. This is the zero-infra default
  (§1.2).
* **Tier B (`disk`)** — a pure-Go on-disk / memory-mapped index for single-node
  corpora too large to hold fully in RAM. It is single-node (no clustering) and,
  like Tier A, MUST remain buildable with `CGO_ENABLED=0` (§6.5).
* **Tier C (`qdrant` / `pgvector`)** — an external vector store. It is
  **optional and MUST NOT be required**: a conforming deployment runs on Tier A
  with no external infrastructure (§1.2, §19). Tier C is for operators who
  already run such a store or who need horizontal scale.

### 6.3 External store addressing (Tier C)

* A Tier C backend is addressed by a **collection / namespace derived from
  `corpus_id`** (§5.5), so multiple corpora can share one external store without
  collision. The two axes map to two collections/namespaces (one for text, one
  for code).
* Connection parameters for Tier C live under `index:` (§16.2); credentials
  follow §16.1.1 and MUST NOT be persisted to the snapshot.
* **No silent fallback.** If a configured Tier C backend is **unreachable at
  preflight** (§2.5), startup MUST fail with `CONFIG_INVALID` and remediation.
  An unreachable external store MUST NOT silently downgrade to an embedded tier
  — that would change the corpus's vector home invisibly.

### 6.4 Embed identity binds every backend

The corpus-lifetime **embed identity** — `provider | base_url | text_model |
code_model | text_dim | code_dim | multimodal` (§8.1.4; `base_url` **normalized**
per §8.1.4, empty for providers where the endpoint is not meaningful) — binds the
index **regardless of backend**. On load, if the configured embed identity differs
from the one recorded for the index (embedded snapshot or external collection
metadata), the server MUST refuse to mix vector spaces: it either errors
(`CONFIG_INVALID`) or triggers a full reindex (§8.1.4). A backend MUST NOT silently
serve a collection built under a different embed identity — including one built
under the same provider/model at a **different endpoint**.

### 6.5 Pure-Go / `CGO_ENABLED=0` (normative)

The embedded backends (Tier A and Tier B) MUST be implementable in **pure Go**
and buildable with **`CGO_ENABLED=0`** — the reference store uses
`modernc.org/sqlite` (a pure-Go SQLite) and a pure-Go ANN implementation
specifically to keep the single-binary, cross-compiled, CGO-free build.

* **`sqlite-vec` is rejected** for the embedded path: it is a C extension and is
  incompatible with the pure-Go `modernc.org/sqlite` driver under
  `CGO_ENABLED=0`. Implementations MUST NOT make `sqlite-vec` (or any other C
  SQLite extension) a requirement of an embedded backend.
* Tier C backends are out-of-process (network clients), so they impose no CGO
  requirement on the dir2mcp binary.

### 6.6 Deletions

* **Embedded backends (Tier A/B)** are treated as **append-only**: deleting
  documents/representations/chunks sets `deleted=1` in SQLite (the tombstone),
  and retrieval uses **oversampling** — ask the index for `k * oversample_factor`
  results, filter out `deleted=1`, return the first `k` remaining. Default
  `oversample_factor`: 5 (configurable).
* **External backends (Tier C)** MAY delete vectors **natively** (e.g. delete by
  `chunk_id` payload) instead of relying solely on oversampling. A Tier C backend
  MUST still **honor the SQLite `deleted=1` tombstone** as the source of truth —
  a vector whose `chunk_id` is tombstoned MUST NOT appear in results even if its
  native deletion has not yet propagated — so retrieval semantics are identical
  across backends.

---

## 7) Ingestion pipeline

### 7.1 Discovery

* Recursive walk from root.
* Default ignore list includes: `.git/`, `node_modules/`, `dist/`, `build/`, `.venv/`, `.dir2mcp/`.
* Optional `.gitignore` support.
* Symlink policy:

  * default: do not follow symlinks
  * if enabled: follow only if target resolves under root

### 7.2 Safety exclusions (default)

* Exclude obvious secrets/credentials patterns (regexes applied to file **contents**):

  * AWS Access Key ID: `AKIA[0-9A-Z]{16}`
  * AWS/Secret assignment heuristic: `(?i)(?:aws(?:[_\s.]{0,20})?secret(?:[_\s.]*(?:access[_\s.]*)?key)?|secret[_\s.]*access[_\s.]*key)\s*[:=]\s*[0-9A-Za-z/+=]{20,}`
  * JWTs: `(?i)(?:authorization\s*[:=]\s*bearer\s+|(?:access|id|refresh)_token\s*[:=]\s*)[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}` (context-anchored)
  * Generic bearer token: `(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}`
  * Common API key formats (e.g. `sk_[a-z0-9]{32}`, `api_[A-Za-z0-9]{32}`)

  These patterns are the **defaults**; they live in configuration under `security.secret_patterns` and can be extended or overridden by users.

  Expected false positives and tuning notes (map each note to its default rule):

  * **AWS Access Key ID** (`AKIA[0-9A-Z]{16}`): may match synthetic examples in docs/tests or random uppercase identifiers of the same shape.
  * **AWS/Secret assignment heuristic** (`(?i)(?:aws(?:[_\s.]{0,20})?secret(?:[_\s.]*(?:access[_\s.]*)?key)?|secret[_\s.]*access[_\s.]*key)\s*[:=]\s*[0-9A-Za-z/+=]{20,}`): reduces prose false positives (for example “AWS Secrets Manager”) by requiring assignment-like context.
  * **JWTs** (`(?i)(?:authorization\s*[:=]\s*bearer\s+|(?:access|id|refresh)_token\s*[:=]\s*)[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}`): reduced false positives via auth/key context and minimum segment lengths; can still match synthetic token-like test strings with those contexts.
  * **Generic bearer token** (`(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}`): can match innocuous config values named `token` (feature tokens, cache tokens) that are not credentials.
  * **Common API key formats** (`sk_[a-z0-9]{32}`, `api_[A-Za-z0-9]{32}`): can match placeholders, test fixtures, or generated IDs that happen to share the prefix/length.

  Refinement guidance via `security.secret_patterns`:

  * Tighten permissive rules (for example JWT/bearer) with context anchors such as preceding auth headers, key names, or delimiters.
  * Replace broad alternations with environment-specific patterns (known provider prefixes, expected lengths/alphabets).
  * Exclude known-safe paths with `security.path_excludes` (for example fixtures, snapshots, generated test vectors) instead of weakening global rules.
  * Keep broad defaults as a baseline, then add narrower allowlist/exception handling in path rules where operationally safe.

  Example tuning directions:

  * JWT: anchor to `Authorization: Bearer` or token key names and enforce minimum segment lengths.
  * Bearer token: constrain key names (`access_token`, `bearer_token`) and reduce accidental matches in generic `token=` fields.
  * AWS secret heuristic: keep assignment/credential context anchors and avoid broad prose matching.

  Testing approach for pattern updates:

  * Build a small positive/negative corpus per rule (must-hit secret samples and must-not-hit benign samples).
  * Run scanner tests in CI on both corpora and assert precision/recall thresholds appropriate for your risk posture.
  * Add regression fixtures for every incident-driven rule change (new false positive or false negative).
  * Review CI diffs of matched files/lines before merging pattern changes; iterate by tightening context anchors or path excludes.

* Exclude large binaries by default:

  * configurable max file size per `doc_type`.

* Path-based exclusions use optional `.gitignore`-style syntax. Users may provide
  additional ignore files or patterns via `security.path_excludes` in config
  (a list of glob patterns); the default set includes the same patterns used for
  ingestion (`.git/`, `node_modules/`, `.dir2mcp/`, etc.) plus any sensitive
  filenames detected.

### 7.3 Type classification

Use extension + MIME sniff + binary heuristics to classify:

* `code`: go/rs/py/js/ts/java/c/cpp/…
* `md/text/data/html`
* `pdf`, `image`, `audio`, `video`
* `archive` (zip/tar/tar.gz) optionally deep extracts members
* `binary_ignored`

### 7.4 Representation generation rules

#### A) Code/text/md/data/html

* **Code/text/md/data.** Generate `raw_text` (normalized UTF-8, `\n` line
  endings). Route to index kind:

  * code → `index_kind=code`
  * others → `index_kind=text`

**Markup boundary (html).** `html` is a *dual-path* format: it MAY be handled
here as flat `raw_text`, or routed to a structured extraction engine that
preserves headings/tables/links. Which path applies is governed by the §7.4.B.1
capability matrix (which lists `html` as structured-capable) and the *Extractor
availability* rules there:

* **When a structured extraction engine that accepts HTML is available** — the
  docling family of §7.4.B, subject to the same `ingest.extractor` selection and
  the *Extractor availability* rules — the pipeline SHOULD route HTML through it,
  producing an `extracted_markdown` representation and the structured `region`
  spans of §7.4.B (heading hierarchy → section breadcrumb; tables rendered
  atomically to Markdown; element labels in `extra_json.label`). HTML carries no
  page/`bbox` provenance, so its `region` spans carry the section breadcrumb and
  `label` and fall back to no page span, per the provenance-unavailable rule in
  §7.4.B.
* **When no structured HTML engine is available** — including when the extractor
  is `off`, explicitly disabled/unavailable (§7.7), or does not accept HTML —
  HTML MUST fall back to `raw_text` (tier T4, §7.4.B.1), exactly as before.
  `raw_text` remains the guaranteed baseline: HTML is never dropped, and behavior
  MUST NOT regress when docling is absent.
* Either path routes to `index_kind=text`. The path choice does not change the
  index kind and follows the re-indexing semantics of §7.6 — a document
  previously indexed as flat `raw_text` keeps that representation until it is
  re-indexed.

The **default** html routing (whether best-available auto promotes html from
flat `raw_text` to a structured engine by default) is governed by **dir2mcp
#556** and is intentionally left unchanged by this revision: until #556 lands, an
implementation MAY continue to route html to `raw_text` and MUST NOT be
considered non-conforming for doing so.

#### B) PDF/image/document

* Generate `extracted_markdown` via a **capability-aware, per-format** selection
  over the extraction-engine registry (§7.4.B.1). `ingest.extractor`
  (§16.2) selects the *policy*, not a single global engine:
  * `auto` (default): **best available per format** — for each format, use the
    highest-fidelity *active* engine that supports it (§7.4.B.1), falling
    through the fidelity order; a format no active engine supports degrades per
    the strict/lenient contract (§7.4.B.2).
  * `docling` / `docling-serve` / `mistral` / `pandoc`: **pin** a single engine.
    A format the pinned engine cannot read does not silently produce an empty
    representation — it degrades honestly per §7.4.B.2.
  * `off`: skip the extracted representation.
* Route to `index_kind=text`.
* Cache extracted output if enabled.

**Extractor transport.** The `docling` *engine* produces the same structured
document regardless of how it is reached; the `docling` vs `docling-serve`
engine selection is the transport: `docling` invokes a local CLI subprocess,
while `docling-serve` calls a docling-serve HTTP service at the endpoint
addressed by `ingest.docling.serve_url` (§16.2). Both transports MUST produce
identical output (the same `extracted_markdown` representation and `region`
spans defined below); the choice is operational and carries no wire- or
schema-level difference.

**Extraction is a §7.4-owned routing decision, not a §8 provider-capability
cell.** Per-format engine selection lives here (§7.4.B.1), *not* in the §8.1.2
capability matrix: extraction fidelity is per-format and ordered, and two of the
engines (`docling`, `pandoc` #393) are local tools with no §8.1.1
provider profile. Where an engine *is* an §8 surface — the `mistral` engine — it
resolves through that capability's binding: the `mistral` extraction engine is
the active `ocr` provider (§8.1.2/§8.1.3), so the OCR-tier engine follows the
`ocr` binding rather than being pinned to a vendor name. The audio path (§7.4.C)
already binds its engine to the §8 `stt` capability; §7.4.B generalizes the same
best-available-by-default, swappable, honestly-degrading shape to documents and
images.

Selecting `docling-serve` REQUIRES a non-empty, reachable `serve_url`. An empty
or unreachable endpoint makes the `docling-serve` extractor **unavailable** — a
disabled extractor for diagnostic purposes (§7.7), exactly as a missing docling
binary disables `docling` — and MUST NOT silently fall back to the CLI. (Under
`extractor: auto` the transport is implementation-determined: an empty
`serve_url` simply means the HTTP transport is not considered, and `auto` may
use the CLI or another configured extractor as usual.)

##### 7.4.B.1 Extraction-engine capability matrix (normative)

The **extraction-engine registry** is the single source of truth for which
engine can ingest which format, replacing scattered MIME allowlists and coarse
`doc_type` routing. Each engine declares the format classes it supports and a
**fidelity tier** (lower = higher fidelity = preferred as the best-available
tiebreak):

| Tier | Engine | Nature | Provenance produced |
|---|---|---|---|
| T1 | `docling` / `docling-serve` | structured document model | reading-order, `region` (page+bbox), section breadcrumb, labels, atomic tables (§7.4.B "Structured extraction") |
| T2 | `pandoc` (#393) | born-digital markup/office/ebook → Markdown | structure without page/bbox: section breadcrumb + `label`; no `page`/`bbox` spans |
| T3 | `mistral` (= §8 `ocr` provider) | page-separated OCR | `page` spans (§7.4.B "Page-separated extraction") |
| T4 | `raw_text` (§7.4.A) | flat text | none |

**Format support** (`✅` = engine can ingest this format; tier from the table
above). The `pandoc` engine (#393) is **optional and capability-activated**: its
cells participate in selection whenever a `pandoc` binary is available (see
*Extractor availability*) and are inactive otherwise, exactly as a missing
`docling` binary deactivates the T1 cells:

| Format class | Examples | docling(-serve) | mistral (ocr) | pandoc† | raw_text |
|---|---|:--:|:--:|:--:|:--:|
| pdf | `.pdf` | ✅ T1 | ✅ T3 | ❌ | ❌ |
| raster-image (OCR-native) | `.png .jpg .jpeg .webp` | ✅ T1 | ✅ T3 | ❌ | ❌ |
| raster-image (extended) | `.tiff .bmp .gif` | ✅ T1 | ❌ | ❌ | ❌ |
| vector-image | `.svg` | ✅ T1 | ❌ | ❌ | ❌ |
| office (Word, OOXML) | `.docx` | ✅ T1 | ❌ | ✅ T2 | ❌ |
| office (slides/sheets, OOXML) | `.pptx .xlsx` | ✅ T1 | ❌ | ❌ | ❌ |
| office/ebook (ODF/RTF/EPUB) | `.odt .rtf .epub` | ❌ | ❌ | ✅ T2 | ❌ |
| legacy office (binary) | `.doc` | ❌ | ❌ | ❌ | ❌ |
| markup | `.html .htm` | ✅ T1 | ❌ | ✅ T2 | ✅ T4 (§7.4.A, #556) |

† `pandoc` (T2, #393) is a born-digital markup/office/ebook converter with a
**reader-only** support set: it ingests `.docx`, `.odt`, `.rtf`, `.epub`, and
`.html`, but **not** `.pptx`/`.xlsx` (pandoc has no PowerPoint/Excel reader —
those are docling-only) nor legacy binary `.doc` (docx-only), and no raster/PDF
input — so those cells are permanently `❌`. Its readable cells are active only
when a `pandoc` binary is available; an implementation or deployment without
`pandoc` treats them as inactive, exactly as a missing `docling` binary
deactivates T1.

**Best-available selection (`extractor: auto`).** For each classified document,
select the **active** engine of lowest fidelity tier whose cell for that
format is `✅`. "Active" means *available* in the §7.4.B "Extractor availability"
sense (resolves + passes its probe; a reachable `serve_url`; a present `ocr`
credential/binding). The selection is **per format**, deterministic, and cached
for the run. A format with an active engine at some tier is never routed to an
engine that cannot read it, and a higher-fidelity active engine is never
bypassed (fixing the "html→raw_text while docling is active" and
"tiff→mistral-rejected" defects, dir2mcp #394/#556).

**Pinned selection (`extractor: docling|docling-serve|mistral|pandoc`).** Only
the named engine is eligible; formats outside its `✅` set degrade per §7.4.B.2.
Pinning is honored exactly (no cross-engine fallback), matching the existing
explicit-`docling` / explicit-`docling-serve` no-silent-fallback rule. Pinning
`pandoc` when no `pandoc` binary is available disables extraction, exactly as
pinning an unavailable `docling`.

##### 7.4.B.2 Degradation contract (strict / lenient)

When no active eligible engine supports a document's format (a coverage gap under
`auto`, or a pinned engine that cannot read the format), the outcome is governed
by `ingest.on_unsupported` (§16.2), a kill-switch-shaped knob mirroring the
tri-state opt-out used elsewhere (e.g. `media.diarize`, §8.6.8):

* **`lenient` (default, backward-compatible)** — **skip with warning**, recorded
  honestly and **durably**. No `extracted_markdown` is produced; the outcome then
  depends on whether the document is searchable by any other means:
  * If it retains **at least one searchable representation** (e.g. a
    direct-embedding media chunk, a sidecar transcript), it stays **indexed**
    (`documents.status=ok`) and the missing extraction is named in the honest
    coverage report (§7.7).
  * If it has **no searchable representation** — extraction was its only path to
    being searchable — it is **not indexed** and MUST be recorded as a **durable
    skip**: `documents.status=skipped` with a `skip_reason` in the
    unsupported-format class (§7.7 `skip_reasons`). It counts toward
    `indexing.skipped`, contributes to the durable coverage aggregate (so
    `status`/`reindex` name it **after a restart**, not only during the run that
    produced it), and emits a per-document `file_skip` event on the `--json`
    stream (§3.2) — the streaming counterpart of the aggregate. An implementation
    MUST NOT leave such a document at `status=ok`: that mislabels an unsearchable
    document as indexed and makes the gap invisible once the producing run ends
    (dir2mcp #584).

  This preserves the **not-indexed outcome** for unsupported formats while making
  it honest, durable, and observable — never a silent empty representation, and
  never an unsearchable document reported as indexed.
* **`strict`** — the unsupported format is a **non-fatal per-document error**
  (§7.7): `documents.status=error` with an `UNSUPPORTED_FORMAT`-class reason;
  indexing continues for other documents. Intended for CI / correctness-sensitive
  corpora that must not silently under-cover.

In neither mode is an unsupported format allowed to yield a silent empty
representation reported as success.

**Extractor availability.** An extractor is *available* only when it can
actually run, not merely when it is configured. For the `docling` CLI this means
the command both **resolves** (on `PATH`, or via `ingest.docling.command`) **and**
passes a lightweight functional check — a successful probe invocation (for
example `docling --version`). A command that resolves but fails the probe — for
example a bundled virtualenv whose dependencies are ABI-incompatible — is
**unavailable**, exactly as an unreachable `serve_url` makes `docling-serve`
unavailable. Implementations SHOULD perform such a check and MUST treat a
present-but-non-functional extractor as unavailable (never as available), and
SHOULD cache the result for the run rather than probing per document.

* Under `extractor: auto`, an unavailable `docling` CLI is skipped and the
  per-format tier order continues (docling-serve, then `pandoc` for the formats
  it covers, then Mistral OCR, then `raw_text` for HTML (§7.4.A), then disabled),
  so a broken docling install degrades gracefully instead of failing every
  document.
* Under `extractor: docling` (explicit), an unavailable command disables
  extraction — PDF/image/document contribute no `extracted_markdown` — and MUST
  NOT silently fall back to another engine, mirroring explicit `docling-serve`.
* The availability decision, and the reason when unavailable, MUST be surfaced
  in startup diagnostics and by `dir2mcp doctor` (§7.7), so a present-but-broken
  extractor is visible rather than reported as healthy.

**pandoc availability (#393).** The `pandoc` engine (T2) is *available* only when
a `pandoc` binary both **resolves** (on `PATH`, or via `ingest.pandoc.command`)
**and** passes a lightweight functional check (a successful `pandoc --version`),
mirroring the docling CLI rule above; a resolved-but-non-functional binary is
**unavailable**. It is **capability-activated**: no enable flag is required — the
presence of a working `pandoc` binary activates the matrix's T2 cells, and its
absence deactivates them (the tri-state kill-switch shape, opt-out only). Under
`extractor: auto` an unavailable `pandoc` is simply skipped and the per-format
tier order continues; under `extractor: pandoc` (explicit) an unavailable binary
disables extraction and MUST NOT silently fall back to another engine, mirroring
explicit `docling`. The decision and its reason MUST be surfaced in startup
diagnostics and by `dir2mcp doctor` (§7.7).

**Structured extraction (docling).** When the extractor emits a structured
document model (docling's `DoclingDocument`, obtained via `--to json`), the
ingest pipeline MUST preserve, not discard, the structure:

* Walk the document body in **reading order** (the `body` tree and group
  children), resolving internal references.
* Maintain a **section breadcrumb** from the heading hierarchy
  (`section_header` items and their levels); attach the active breadcrumb to
  every chunk emitted beneath it.
* Carry per-element **provenance**: page number and bounding box (`bbox`) from
  each element's provenance, stored as `region` spans (§5.4). When provenance
  is unavailable for an element, fall back to a `page` span (or none).
* Preserve element **labels** (`paragraph`, `section_header`, `list_item`,
  `table`, `caption`, `code`, `formula`, `picture`) in span `extra_json.label`.
* **Tables** are rendered to faithful Markdown for the chunk text and kept
  atomic (a table is not split across chunks); cell structure (row/column
  spans) MAY additionally be retained in span `extra_json`.
* **Pictures/figures** contribute their captions and any classification
  annotations as searchable text, attributed to the figure's provenance.
* The document **title**, when the model exposes a `title` element, SHOULD be
  used to populate `documents.title` in preference to the text heuristic.

**What is persisted.** The structured path does not change the persisted
representation type or the indexed content shape:

* The `extracted_markdown` representation stores **rendered Markdown** — the
  document's structure linearized to Markdown in reading order (tables as
  Markdown tables, figure captions inline). This is the text that is chunked,
  embedded, and returned in snippets, exactly as in the flat path. `rep_hash`
  is computed over this rendered Markdown.
* The structure that flat Markdown cannot carry — page, `bbox`, section
  breadcrumb, element label — is persisted as `region` **spans** (§5.4)
  attached to each chunk, not as a separate representation.
* The raw `DoclingDocument` JSON is **not** a representation. Implementations
  MAY cache it (alongside the extracted output, when caching is enabled) to
  avoid re-running docling on re-index, but it is an implementation-private
  cache artifact, not part of the spec'd store contract.
* Re-indexing semantics are unchanged (§7.6): a document re-ingested under the
  structured path produces the same `extracted_markdown` representation; only
  the span provenance is richer. Documents previously ingested via flat
  Markdown keep their `page`/no spans until re-indexed.

**Markup/office extraction (pandoc) (#393).** When the active engine is `pandoc`
(T2) — a born-digital converter with no page raster or layout model — the
pipeline produces an `extracted_markdown` representation by converting the source
to Markdown, and MUST NOT fabricate page/`bbox` provenance it does not have:

* Convert the document to Markdown (`pandoc -t gfm`), preserving reading order —
  pandoc emits a single linear document.
* An implementation **SHOULD**, where the Markdown heading hierarchy is
  available, carry a **section breadcrumb** onto the chunks beneath each heading
  as the structured path does, and **MAY** carry an element kind (e.g. table,
  code block) in span `extra_json.label`. Unlike docling's structured model this
  is a **progressive enhancement** over the guaranteed Markdown text, not a
  structured-model guarantee.
* **No page/`bbox` provenance exists** for born-digital formats: pandoc spans
  carry the section breadcrumb (and `label` where derivable) and otherwise fall
  back to **no `page` span** — the pipeline MUST NOT fabricate one. This is the
  provenance-unavailable rule of the structured path applied to an engine that
  never has page provenance; citations are therefore section-granular — coarser
  than docling's `region` spans, an accepted trade for covering formats no
  higher-fidelity active engine reads.
* **Tables** are rendered to Markdown and kept atomic where the converter
  preserves them.
* Route to `index_kind=text`. `rep_hash` is computed over the rendered Markdown,
  exactly as the docling and flat paths; the persisted representation type is
  unchanged (`extracted_markdown`), only the span provenance is coarser.
* Re-indexing semantics are unchanged (§7.6): under `auto`, a format later
  covered by a higher-fidelity active engine (e.g. docling installed) is
  re-extracted through it on re-index per the best-available rule; until then the
  pandoc representation stands.

See [Design 0002](design/0002-structured-extraction.md) for rationale and the
structure-to-provenance mapping.

**Page-separated extraction (OCR fallback).** When the extractor emits only
page-separated text (e.g. Mistral OCR), page-aware behavior applies:

* store page numbers as `page` spans
* chunk per page first

#### C) Audio (STT provider is configurable)

* Generate `transcript` via STT provider:

  * default: **Mistral STT**
  * optional: **ElevenLabs STT**
* If timestamps available:

  * segment into time windows (e.g., 30s with 5s overlap)
  * store spans as `time` (start_ms/end_ms)
* If timestamps not available:

  * fall back to text-size chunking
  * omit time spans
* Cache transcript if enabled.

#### D) Structured extraction (annotations)

* Default: on-demand only, via MCP tool.
* Store `annotation_json` representation.
* Optionally derive and embed `annotation_text`:

  * flattened `key: value` lines
  * route to `index_kind=text`

### 7.5 Chunking defaults

* Global character-based chunking:

  * `max_chars`, `overlap_chars`, `min_chars`
* Code:

  * line-window chunking (max_lines, overlap_lines)
  * store `lines` spans
* Structured document (docling):

  * section/element-aware: group consecutive elements under the same section
    breadcrumb, then split by size constraints (`max_chars`, `overlap_chars`,
    `min_chars`)
  * keep tables atomic (never split a table across chunks)
  * store `region` spans (page + bbox + section breadcrumb); fall back to
    `page` spans where provenance is missing
* OCR (page-separated):

  * per page, then within page by size constraints
  * store `page` spans
* Transcript:

  * segment by time if available
  * store `time` spans

### 7.6 Incremental indexing

* Document-level:

  * compute `content_hash`; if unchanged and not deleted → skip rep generation
* Representation-level:

  * compute `rep_hash`; if unchanged → skip chunk rebuild
* Chunk-level:

  * compute `text_hash`; if unchanged → skip embedding

### 7.7 Per-document error handling

Non-fatal per-doc errors:

* mark `documents.status=error`, record error
* continue indexing

Fatal errors:

* root inaccessible
* cannot write state (disk full, permissions)
* irrecoverable state corruption

**Honest coverage report (normative).** Startup diagnostics and `dir2mcp doctor`
MUST report extraction coverage honestly, extending the existing requirement
that a present-but-broken extractor be visible rather than reported as healthy
(§7.4.B). The report MUST:

* list the **active extraction engines** and, per engine, its availability and
  (when unavailable) the reason;
* name every **corpus format class present but not covered** by any active
  engine (per the §7.4.B.1 matrix) — e.g. "`.odt`, `.tiff` present, no active
  engine covers them";
* for each uncovered class, name a **remediation** — the engine/config to add
  (e.g. "install docling for `.tiff`; install `pandoc` for `.odt` (#393); or
  set `ingest.on_unsupported: strict` to fail instead of skip").

Under `ingest.on_unsupported: lenient` the uncovered classes are warnings, and a
document left with no searchable representation is recorded as a durable
`status=skipped` (skip_reason in the unsupported-format class) so the gap survives
the run that produced it (§7.4.B.2); under `strict` the affected documents are
recorded as `status=error` (§7.4.B.2). A coverage gap MUST never be silent, and
MUST never be reported as an indexed document.

### 7.8 Remote corpus sources

The corpus root MAY live on a remote source. `source.kind` (§16.2) selects the
scheme:

* `local` (default) — a local filesystem path.
* `nfs` — a mounted network filesystem path.
* `s3` — objects under an S3 bucket + prefix (`source.s3.bucket`,
  `source.s3.prefix`, plus region/endpoint; credentials per §16.1.1, never
  persisted).

**Enumeration.** `local` and `nfs` are walked as filesystems and obey the same
discovery, symlink, and ignore rules as §7.1 (they are ordinary directory trees).
`s3` enumerates objects under `bucket`/`prefix` (a flat object listing, not a
filesystem walk).

**Stable `rel_path` across schemes.** `rel_path` (§5.1) is defined relative to
the corpus root for every scheme: for `local`/`nfs` it is the path under the root
directory; for `s3` it is the **object key minus the configured prefix**. The
normalization MUST be chosen so that the *same logical corpus* yields the *same*
`rel_path` set under any scheme — a corpus may be relocated `local ⇄ nfs ⇄ s3`
**without changing its document identity** (and therefore without a forced
reindex on relocation alone). Traversal / root-escape protections (§17) apply to
**every** scheme: an object key or path that resolves outside the configured
root/prefix MUST be rejected (`PATH_OUTSIDE_ROOT`).

**Change-detection identity.** Incremental indexing (§7.6) keys off a cheap
signal first, then confirms with `content_hash`:

* `local` / `nfs`: the cheap pre-check is `(size, mtime)`; on a change,
  `content_hash` over the file body **confirms** before re-ingest.
* `s3`: the cheap signal is the object **ETag** (alongside `size` and
  `last_modified`). The ETag MUST NOT be treated as a content hash: multipart and
  SSE-KMS ETags are **not** MD5 of the body. `content_hash` therefore still
  requires **reading the object body**; the ETag only decides *whether* a re-read
  is warranted.

**Deletions.** A source object/file that is no longer present at enumeration is a
deletion → it is **tombstoned** (`deleted=1`, §5.1), exactly as for a removed
local file.

**State stays local.** Regardless of `source.kind`, the **state directory**
(SQLite metadata, the embedded index, and caches) is always **local** (§1.2):
dir2mcp never writes its index/state back to the remote source. Only the corpus
*content* is remote.

### 7.9 Cross-file canonicalization (optional)

Real corpora contain **duplicates**: the same logical content present at multiple
paths (mirrored directories, the same file copied across folders) or in
byte-identical copies. Indexing every copy bloats the index and returns the same
content multiple times for one query, degrading answer quality. Cross-file
canonicalization collapses duplicates to a single **canonical** document while
keeping the others discoverable as **aliases**. It is **optional and off by
default**; when disabled, behavior is exactly as before (every file is indexed
independently).

**Duplicate grouping (exact).** When `dedup.exact: true`, documents that share an
identical `content_hash` (§7.6) form a **duplicate group**. Grouping is by content
identity, not by name — it therefore also collapses the same bytes stored under
different paths.

**Canonical selection.** The pipeline selects exactly one canonical document per
group **deterministically**, using the same policy vocabulary as media variant
selection (§8.6.5):

* `dedup.select: best` (default) — prefer the **richest/largest** rendition:
  highest detected resolution (when applicable), then largest `size_bytes`, then
  the lexically-lowest `rel_path`.
* `dedup.select: first` — the lexically-lowest `rel_path`.

The choice MUST NOT depend on enumeration order beyond the stated tiebreaks, so
re-runs over an unchanged corpus are stable.

**Canonical vs alias behavior.** The pipeline generates representations, chunks,
and embeddings **only for the canonical** document. Non-canonical members are
recorded as **aliases** (§5.1 `is_alias`/`canonical_doc_id`): they remain
discoverable (`list_files`) and resolvable (`open_file` returns their own
byte-identical content), are **tombstoned** on removal exactly like any document,
but contribute **no** chunks or embeddings and therefore **no** retrieval hits.

**Canonical removal.** When the canonical document of a group is removed
(tombstoned, §5.1), an alias of that group MUST be **promoted** to canonical and
(re-)indexed deterministically by the same selection policy, so the group's
content does not silently disappear from retrieval.

**Relationship to media variants (§8.6.5).** Variant/multi-rendition selection is
the **media-specific special case** of this rule: it groups by *normalized name*
and selects the best rendition. `media.variants` and `dedup` share the
`best|first` canonical-selection vocabulary. When both are configured, variant
selection applies first (within a logical media's renditions) and cross-file
dedup then applies across the remaining distinct-content documents.

**Near-duplicates (non-normative, future).** Re-encodes and same-document-in-
another-format (e.g. PDF + DOCX) have **different bytes** and are therefore *not*
collapsed by exact grouping. Similarity-based near-duplicate detection (e.g.
embedding-centroid or MinHash) is **out of scope** for this version and, if added
later, MUST remain opt-in and additive on top of the alias machinery defined here.

**Retrieval-time de-duplication.** See §9.2: a query MUST NOT return multiple hits
whose source documents belong to the same duplicate group.

### 7.10 CorpusFS — corpus filesystem abstraction

> **Status: Planned.** This subsection formalizes the **logical contract** that
> the §7.8 corpus schemes (`local`, `nfs`, `s3`) implement, so discovery and media
> byte-reads work against any backing store without callers caring which one is in
> use. It is **domain-general** and **implementation-agnostic**: it names
> *capabilities*, not Go types or wire calls. Implementation lands in a follow-up
> dir2mcp code PR (dir2mcp #242) once this spec change is merged.

§7.8 defines *which* corpus locations exist; **CorpusFS** defines the small,
backend-neutral surface every such location MUST present. A conforming corpus
source is anything that can satisfy the three capabilities below; the §7.8 schemes
are the reference bindings (`local`/`nfs` ⇒ filesystem, `s3` ⇒ object store), and
adding a new backing store is adding a new CorpusFS binding, **not** a change to
any caller.

**Capabilities (normative).** A CorpusFS MUST provide exactly these three:

* **list** — enumerate the documents under the corpus root. Each entry MUST carry
  enough metadata to drive incremental indexing (§7.6, §7.8) without opening the
  body: a `rel_path` (§7.8 stable-`rel_path` rule), a `size`, a modification
  signal, and the backend's **cheap change signal** — `(size, mtime)` for
  `local`/`nfs`, the object **ETag** (plus `size`/`last_modified`) for `s3` (§7.8).
  Enumeration obeys the discovery, symlink, and ignore rules of §7.1 for
  filesystem schemes and the flat object-listing model for `s3` (§7.8).
* **stat** — return the same metadata as a `list` entry for a single `rel_path`,
  so a caller can refresh one document's change signal without a full
  re-enumeration. `stat` of a missing `rel_path` MUST be distinguishable from an
  error (it drives the deletion → tombstone path, §7.8).
* **open / range-read** — open a document's bytes for reading and support
  **random-access range reads** (read *N* bytes at offset *O*) — not only a
  whole-file stream. Range reads are required so media windowing (§8.1.7), PDF
  per-page extraction, and `dir2mcp_open_media_clip` (§15.11) can fetch only the
  byte ranges they need; on `s3` a range read maps to a ranged `GET`, on
  `local`/`nfs` to a positioned file read. `content_hash` (§7.6) is computed over
  the **bytes returned by open**, identically across backends, so document
  identity is backend-independent (§7.8 relocation invariant).

**Invariants.**

* **Identity is backend-independent.** The `rel_path` set, `content_hash`, and
  therefore document/representation/chunk identity MUST be identical for the same
  logical corpus regardless of which CorpusFS backs it (§7.8). Relocating a corpus
  `local ⇄ nfs ⇄ s3` MUST NOT, by itself, force a reindex.
* **Root/prefix isolation applies to every capability.** A `list`, `stat`, or
  `open` for a `rel_path` (or object key) that resolves outside the configured
  root/prefix MUST be rejected (`PATH_OUTSIDE_ROOT`, §17), on every backend.
* **State stays local.** A CorpusFS exposes the corpus **content** only; it is
  never the home of the state directory (SQLite metadata, the embedded index,
  caches), which is always local (§1.2, §7.8). A CorpusFS is **read-only** with
  respect to dir2mcp — the pipeline never writes corpus content back through it.
* **Selection.** The active CorpusFS is chosen by `source.kind` (§7.8, §16.2);
  `local` is the default. No new config surface is introduced by this
  subsection — `source:` (§16.2) already declares the backing store.

---

## 8) Model/provider utilization requirements

### 8.1 Provider model (provider-agnostic)

dir2mcp is **provider-agnostic**. Each model capability — `embed`, `chat`, `ocr`, `stt`, `tts`, `rerank` — binds to a named **provider profile**. Mistral is the default profile but is **not** privileged. Rationale and full design: [Design 0001](design/0001-multi-provider.md).

#### 8.1.1 Provider profiles

A profile declares a `kind` (the adapter / wire protocol), a `base_url` (defaulted per kind; overridable), an **optional** `api_key` secret reference (resolved per 16.1.1, never persisted), and per-capability default model names. A profile with no `api_key` is **credential-less** (e.g. a local Ollama/vLLM/LM Studio endpoint that requires no key); credential-less profiles are first-class and count as **eligible** for selection and preflight (8.1.3). Defined `kind`s:

* `openai` — the OpenAI-compatible **backbone**: OpenAI, OpenRouter, Groq, Together, Azure-style, and local Ollama/vLLM/LM Studio — **and Mistral chat/embeddings** (`api.mistral.ai` already serves `/v1/chat/completions` and `/v1/embeddings`). Endpoints that expose audio also serve STT (`/v1/audio/transcriptions`, Whisper / `gpt-4o-transcribe`) and TTS (`/v1/audio/speech`) — endpoint-dependent, see 8.1.2.
* `mistral` — native `/v1/ocr` (and Voxtral STT); the only genuinely non-OpenAI Mistral surface.
* `anthropic` — Messages API (chat only).
* `gemini` — native embed (**asymmetric** via `taskType`, with Matryoshka output dimensionality — see 8.1.5/8.1.6), chat, STT (audio transcription), and TTS. The native embed surface (`models/{model}:batchEmbedContents`) is required for `taskType`/`outputDimensionality`; STT and TTS likewise use the native `models/{model}:generateContent` surface (see 8.2/8.3) — Gemini's OpenAI-compatible layer does **not** expose `/v1/audio/*`, so only chat may ride the `kind: openai` path. A `gemini` profile MAY alternatively be configured as a `kind: openai` profile via Gemini's OpenAI-compatible endpoint, which serves chat only and forgoes `taskType` (and thus the asymmetric/role behavior).
* `cohere` — embed, chat, and rerank (8.4). Cohere embeddings are **asymmetric** (see 8.1.5).
* `elevenlabs` — STT/TTS.

Built-in profiles ship for common providers so operators typically only supply a credential.

#### 8.1.2 Capability matrix (normative)

| `kind` | embed | chat | ocr | stt | tts | rerank |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| `openai` | ✅ | ✅ | ❌ | ✅³ | ✅³ | ❌ |
| `mistral` | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| `anthropic` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `gemini` | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| `cohere` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `elevenlabs` | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |

Binding a capability to a `kind` whose cell is `❌` MUST be rejected as `CONFIG_INVALID` (static validation). ³ = `kind: openai` audio (STT/TTS) is **endpoint-dependent** and cannot be statically validated (an arbitrary OpenAI-compatible `base_url` may omit `/v1/audio/*`). The adapter implements it; if the configured endpoint lacks it, the failure surfaces **at first use** as a provider error — a required STT path fails that ingest item, optional TTS fails open (8.3) — never as `CONFIG_INVALID`. All other `✅` cells are statically valid.

**Extraction is not a cell in this matrix.** Document/image *extraction-engine*
selection (docling / docling-serve / mistral-ocr / pandoc) is a per-format,
fidelity-ordered routing decision owned by §7.4.B.1, not a `kind`-level
capability here: extraction fidelity is per-format and two engines (`docling`,
`pandoc`) have no §8.1.1 provider profile. Where an extraction engine *is* an §8
surface, it binds through the corresponding capability — the `mistral`
extraction engine is the active `ocr` provider (selected per §8.1.3), and the
audio extraction path binds `stt` (§7.4.C, §8.2). No `extract`/`CapExtract`
capability cell is added.

#### 8.1.3 Provider selection

For each capability, with `<cap>.provider`:

1. **Set** → use that profile, validated against 8.1.2. If it is required and the profile is not eligible (no credential present **and** not credential-less) → `CONFIG_INVALID` with remediation.
2. **Unset (auto)** → select the first profile, by a fixed deterministic precedence, that both (a) is **eligible** — a credential is present, or the profile is credential-less (e.g. a local endpoint) — and (b) can serve the capability. This generalizes the capability-driven activation rule already used by rerank (8.4) and STT (8.2).
3. **None qualify** → a *required* capability (`embed`) fails the startup preflight (§2.5); an *optional* one (`rerank`) stays off silently.

#### 8.1.4 Embeddings are a corpus-lifetime invariant

Vectors from different embed providers/models — **or from the same provider/model served at a different endpoint** — are not comparable. The embed **identity** — provider, **the normalized embed endpoint `base_url` (8.1.1)**, per-axis model, **and the requested output dimension** (8.1.6, recorded as `embed_text_dim`/`embed_code_dim`, §5.5) — is bound to the index at first build and recorded in the config snapshot. On load, if the configured embed identity differs from the index's, the server MUST refuse to mix vector spaces — either erroring (`CONFIG_INVALID`) or triggering a full reindex. `embed.provider`/**the normalized `base_url`**/`embed.text_model`/`embed.code_model`/`embed.text_dim`/`embed.code_dim` — **and the multimodal mode (8.1.7)** — are therefore deploy-time, reindex-bound choices; `chat`/`ocr`/`stt`/`rerank` providers are runtime-swappable. The input role (8.1.5) is **not** part of this identity.

**Why `base_url` is part of the identity.** Two profiles with the same `kind` and model name pointed at **different** endpoints (e.g. two `kind: openai` self-hosted vLLM/Ollama deployments, or a proxy vs. the hosted API) serve **different** vector spaces. Without `base_url` in the identity they collapse to one identity and their vectors can silently mix in a single index — a violation of the "MUST refuse to mix vector spaces" rule above. Including the endpoint closes that gap.

**`base_url` normalization (normative).** `base_url` enters the identity in **canonical, normalized** form so that trivially-different-but-equivalent URLs do not fragment the identity and force needless re-embeds. The recorded value is computed as follows:
1. **Not-meaningful → empty.** For a `kind` whose embed endpoint is a single canonical provider surface that does not select an alternate model space (native `gemini`, `cohere`), the normalized `base_url` is the **empty string** `""` — `base_url` does not participate in the identity for that provider.
2. **Canonical/default → empty.** If the effective `base_url` is unset, or equals the built-in profile's shipped canonical `base_url` for that provider (e.g. `kind: openai` at `https://api.openai.com/v1`, the default `mistral` profile at `https://api.mistral.ai/v1`), it normalizes to `""`. Only an operator-**overridden**, non-canonical endpoint (the exact mis-bind case) yields a non-empty component.
3. **URL canonicalization** (applied before comparison, for the non-empty case): lowercase the scheme and host; remove the default port (`80` for `http`, `443` for `https`); strip trailing slash(es) and collapse duplicate slashes in the path; **preserve** the remaining path (e.g. `/v1`, which can select a different API mount); drop any userinfo, query, and fragment; apply canonical percent-/IDN-encoding. The result is compared exactly (path remains case-sensitive after host lowercasing).

**`""` is a valid identity component.** The empty string is a first-class, legitimate value of the `base_url` component, not a sentinel for "unknown". Consequently an index built **before** this rule — which recorded no `base_url` — is treated as having `base_url == ""` and remains **valid** on reload against any provider whose normalized `base_url` is also `""` (all built-in/hosted-default deployments, per rules 1–2). Only a corpus whose embed endpoint is a **non-canonical / custom** `base_url` sees a one-time `CONFIG_INVALID`/reindex on first reload after this change — the correct, bounded safety action, since those are exactly the corpora previously at risk of silent cross-endpoint mixing.

#### 8.1.5 Asymmetric embeddings (input role)

Some embedding providers (notably **Cohere** via `input_type`, **Gemini** via `taskType`, and Voyage) are **asymmetric**: documents and queries MUST be embedded with a distinct input role to achieve their stated retrieval quality. Therefore:

* Every embedding call carries an **input role** ∈ {`document`, `query`}: corpus/index-time embeddings use `document`; search-time query embeddings use `query`. The role is determined by the call site, not by configuration.
* Adapters for asymmetric providers MUST map the role to the provider's mechanism. Adapters for symmetric providers (OpenAI, Mistral) MUST accept the role and MAY ignore it; behavior MUST NOT differ for symmetric providers.
  * **Cohere**: `input_type=search_document` (role `document`) / `search_query` (role `query`).
  * **Gemini** (native embed surface): `taskType` MUST be sent on every call. Role `document` → `RETRIEVAL_DOCUMENT`; role `query` → `RETRIEVAL_QUERY`. **Code-aware refinement:** when the call uses the configured **code** model (`embed.code_model`), role `query` maps to `CODE_RETRIEVAL_QUERY` (code documents still embed as `RETRIEVAL_DOCUMENT`, since Gemini has no code-specific document task). A `gemini` profile configured as `kind: openai` (OpenAI-compatible endpoint) cannot send `taskType` and is therefore treated as symmetric.
* The input role is **not** a configuration knob and does not affect the corpus-lifetime invariant (8.1.4): the recorded embed identity is provider + model + requested dimension (8.1.6), independent of role.
* The reference `Embedder` interface gains the role parameter (a clean, internal, pre-1.0 break — no compatibility users); see [Design 0001 §5.6](design/0001-multi-provider.md).

#### 8.1.6 Configurable embedding dimensionality (Matryoshka / MRL)

Some embedding models (notably **Gemini** `gemini-embedding-001`) are trained with Matryoshka Representation Learning: a single model emits a high-dimensional vector (Gemini native **3072**) whose leading prefix MAY be truncated to a smaller dimension (e.g. **1536**, **768**) with graceful quality degradation. Therefore:

* `model.embed.text_dim` / `model.embed.code_dim` are **optional** config knobs requesting a specific output dimensionality per axis. Omitted ⇒ the model's native dimension. The default for `gemini-embedding-001` is its native **3072**.
* When a non-native dimension is requested, the adapter MUST (a) request it from the provider where supported (e.g. Gemini `outputDimensionality`) and (b) **re-normalize** the returned vector to unit L2 length — MRL-truncated vectors below the native dimension are not pre-normalized, and the index's cosine/IP scoring assumes unit vectors.
* The requested dimension is part of the corpus-lifetime embed identity (8.1.4): it is recorded as `embed_text_dim`/`embed_code_dim` (§5.5), and changing it forces a reindex / `CONFIG_INVALID` on mismatched reload, exactly like a model change.
* A provider/model that does not support a requested dimension (no MRL, or a value its model cannot serve) MUST fail with `CONFIG_INVALID` rather than silently ignoring the knob, so an operator never believes a dimension is in effect when it is not.

#### 8.1.7 Multimodal embeddings (optional)

Some embedding models are **natively multimodal** — they map text and media
(images, audio, video, PDFs) into one **shared** vector space, so a text
query can retrieve a media chunk and vice versa. The reference multimodal
model is Google **`gemini-embedding-2`** (native surface
`models/{model}:embedContent` / `:batchEmbedContents`). Design rationale and
phasing: [Design 0003](design/0003-multimodal-embeddings.md).

> **Preview caveat.** `gemini-embedding-2` is Public Preview; the limits and
> formats below are from preview docs and MUST be re-verified against the
> current provider docs before any implementation releases against them.

Per-request limits (preview): text ≤ 8192 tokens; images ≤ 6 (PNG, JPEG,
WebP, BMP, HEIC, HEIF, AVIF); video ≤ 120 s (MP4, MOV); audio ≤ 180 s (MP3,
WAV); PDF 1 file ≤ 6 pages. All modalities share one **unified 8192-token
window**, so chunking MUST budget the *combined* request, not just the
per-modality caps. Output is 3072-dim with Matryoshka truncation (8.1.6);
`taskType` (8.1.5) applies across all modalities.

**Media chunking (windowing).** A media file is chunked into one or more media
chunks before embedding, each chunk sized to fit one embed request:

* A standalone **image** is one chunk (`page` 1). A **PDF** is one chunk per
  page (`page` span); one page per request stays within the per-modality page
  cap (≤ 6 pages). Per-page token cost still counts against the unified
  8192-token budget like any other modality.
* **Audio** and **video** are chunked into **non-overlapping, contiguous time
  windows** covering the whole file; each window is one media chunk with a
  `time` span (`start_ms`/`end_ms`, §5.4). Each window MUST respect **both**
  the per-modality duration cap (audio ≤ 180 s, video ≤ 120 s) **and** the
  unified 8192-token budget; implementations SHOULD use conservative default
  window lengths at or below the caps and MAY make them configurable. A file
  of duration *D* with window length *W* yields ⌈*D*/*W*⌉ windows, the last
  being the remainder.
* **Video has no default text representation** (there is no video→text path
  analogous to audio STT, §7.4.C). It is therefore searchable **only** via its
  media windows: under `off` a video produces no chunks; under `augment` and
  `replace` it is represented solely by its `time`-windowed media chunks. Audio
  retains its transcript path (§7.4.C) in `off`/`augment` as before.
* Windowing MUST be **deterministic** — the same file produces the same window
  boundaries on every (re)index — so `time`-span citations are stable.
* The ingester determines media duration. A file whose duration cannot be
  determined is **not** directly embedded; the condition is treated as a
  non-fatal per-document error (§7.7) and a warning SHOULD be emitted. For
  modalities that have a text path (image/PDF OCR, audio transcript), that
  text representation is retained **even under `replace`**, so the file stays
  searchable; a video, which has no text path, is left unindexed. (This same
  text-path-retained fallback applies when a PDF's page count cannot be read.)

* **`model.embed.multimodal`** is a tri-state per-corpus knob:
  * `off` (default) — text-only; current behavior; **any** embed provider.
  * `augment` — keep text extraction + text embeddings **and** additionally
    embed media files directly; both are indexed.
  * `replace` — embed media files directly **instead of** OCR/STT→text; text
    files are unchanged.
* **Single shared space (per 8.1.4).** When `multimodal` is `augment` or
  `replace`, the **entire** embed binding MUST resolve to the multimodal
  model on `gemini`: `embed.provider: gemini` **and both** `embed.text_model`
  **and** `embed.code_model` set to `gemini-embedding-2` (the code axis is
  not exempt — leaving it on a different model would mix incomparable vectors
  in one index). Any other binding is `CONFIG_INVALID`. `off` keeps full
  provider freedom.
* **Reindex-bound.** The multimodal mode is part of the embed identity
  (8.1.4); switching `off`↔`augment`↔`replace` requires a reindex.
* **Provenance.** A media chunk is a representation (§7.4.B) whose persisted
  span reuses the existing `span_kind ∈ {lines, page, time, region}` (§5.4)
  — **no new persisted kind**: a standalone image → `page` 1, audio/video
  windows → `time`, PDF pages → `page`/`region`. (`document`, §15.1.1,
  remains a client-facing `open_file`-only variant, not persisted.)
* **Retrieval.** A text query embeds via the model's text path and retrieves
  any chunk in the shared space, including media. In `augment`, a PDF page
  may carry several docling text/region chunks (§7.5) **and** one coarse
  page-image chunk; to avoid double-counting, retrieval MUST drop a
  page-image candidate for `(rel_path, page)` only when a text/region
  candidate for that same page survives, **before** truncation/rerank —
  distinct text/region chunks are never collapsed into each other.
* **`ask` over media.** Generation grounds on available text: in `augment`
  the media hit's OCR/transcript text grounds the answer; a `replace`-mode
  media-only hit (no text) is cited without quoted context. (Multimodal
  answer grounding is a later concern.)
* **Inspection.** `open_file` returns text only (§15.4); a `replace`-mode
  media-only chunk has no text representation, a **permanent** condition, so
  `open_file` MUST return the non-retryable `MEDIA_NO_TEXT` (§14.2) — never
  raw binary and never the retryable `OCR_NOT_READY`.

The §8.1.2 capability matrix is unchanged: multimodality is a property of
the chosen embed model, not a new capability cell.

### 8.2 STT providers

* STT provider is selected per 8.1.3 among STT-capable profiles (8.1.2): **Mistral** (Voxtral), **ElevenLabs** (Scribe), **OpenAI** (Whisper / `gpt-4o-transcribe`), **Gemini**. Default profile: **Mistral**.
* Outputs MUST be normalized to the same `transcript` representation format regardless of provider.
* **Gemini** transcribes via the native `models/{model}:generateContent` surface: the audio is sent as an inline-data part (base64, with its MIME type) alongside a transcription instruction, and the model's text output is the transcript. Gemini's OpenAI-compatible layer exposes no `/v1/audio/transcriptions`, so the native surface is required (a `kind: openai` Gemini profile is therefore not STT-capable). The `stt_model` (default `gemini-2.5-flash`) and optional `stt_language` apply as for other providers.

#### 8.2.1 Language-coverage-aware selection (honest coverage)

A single configured STT model does not cover every language equally. Some
languages are **absent from or weak in** a model's training (e.g. Whisper on a
low-resource language): rather than erroring, such a model typically **detects
the nearest in-vocabulary language and emits wrong-language, repetition-prone
output**, so coverage collapses **silently** — the §8.6.6 quality gate then drops
much of it, hiding the cause. STT selection MAY therefore be
**language-coverage-aware**. This section is **domain-general**: dir2mcp ships **no
built-in model→language table** (such tables are provider-specific and go stale);
coverage is **operator-declared** and defaults to *open/unknown* (today's
behavior — attempt any language).

* **Source-language detection.** The source language is resolved before
  transcription from, in precedence order: an explicit `media.language` /
  `stt_language` pin, else automatic detection (`internal/langdetect`). The
  resolved language is recorded on the `transcript` representation's `meta_json`
  (§5.2) for observability regardless of the rules below.
* **Declared coverage (optional).** An STT-capable provider profile MAY declare
  `stt_languages` — a **non-empty** set of BCP-47 tags it is known to transcribe
  well. **Unset _or_ empty (`[]`) = open/unknown**: no coverage assertion is made
  and selection proceeds as today — an empty list is treated as "no declaration",
  **never** as "covers zero languages" (which would pointlessly floor every
  language). A declared (non-empty) set is a *positive* assertion only (a language
  in the set is covered); it never expands to imply exclusion beyond the
  honest-coverage rule below.
* **Per-language routing (optional).** `media.stt.language_providers` maps a
  BCP-47 tag → the **name of an STT-capable provider profile** (§8.1.2/§8.1.3).
  When the resolved source language matches a key, that profile transcribes the
  item, overriding the default STT provider for that item only — so an operator
  can route a language their default model handles poorly to a model that covers
  it. A mapped name that is absent or not STT-capable is `CONFIG_INVALID`
  (static validation). An empty/unset map is today's single-provider behavior.
* **Honest-coverage floor.** When the resolved source language is **outside the
  selected model's declared `stt_languages`** (a **non-empty** set is declared and
  the language is not in it) **and** no `language_providers` route covers it, the
  daemon MUST
  NOT silently emit degraded output as if it were fine. It MUST **proceed but
  record honest coverage**: emit a warning and note the (language, model,
  `covered=false`) fact on the transcript `meta_json`, so an operator can see
  "transcribed in a language this model does not declare" instead of only a
  downstream quality-gate drop. This is **fail-open** (transcription still runs;
  the §8.6.6 quality gate remains the backstop for degenerate output) — a strict
  *skip-instead-of-transcribe* mode and a `dir2mcp_stats` coverage aggregate are
  a planned additive extension, not required here. When coverage is **unknown**
  (no `stt_languages`, or an empty one) the floor does not apply — absence of a
  declared set is not evidence of non-coverage.

### 8.3 Note on TTS

* TTS is optional and not required for core retrieval/inspection functionality.
* When used, the TTS provider is selected per 8.1.3 among TTS-capable profiles (8.1.2): **ElevenLabs**, **OpenAI** (`/v1/audio/speech`), **Gemini**.
* It must remain additive and must not break non-TTS workflows; a TTS provider error fails open (the workflow proceeds without audio).
* **Gemini** synthesizes via the native `models/{model}:generateContent` surface with `generationConfig.responseModalities: ["AUDIO"]` and a `speechConfig` voice (`tts_voice`, default `Kore`); the TTS model is `tts_model` (default `gemini-2.5-flash-preview-tts`). Gemini returns raw single-channel PCM (signed 16-bit little-endian, 24 kHz) as inline data; the adapter MUST wrap it in a self-describing container (WAV) so the bytes are directly playable, matching the ready-to-play audio the ElevenLabs/OpenAI adapters return. Gemini's OpenAI-compatible layer exposes no `/v1/audio/speech`, so the native surface is required.

### 8.4 Rerank providers (optional)

* Reranking is **optional** and **capability-driven**: it activates automatically when a rerank provider credential is present and is disabled otherwise. No explicit enable flag is required (this mirrors how embedding/OCR providers activate on credential presence under the server-first preflight model).
* `rerank.enabled` is an **optional override**, not the activation switch:
  * unset → auto (reranking on **iff** a credential is present);
  * `false` → force reranking **off** even when a credential is present;
  * `true` → require reranking — if no credential is present the server MUST fall back (fail-open) and SHOULD emit a warning.
* Optional rerank provider: **Cohere** (`POST /v2/rerank`, default model `rerank-v3.5`).
* When active, the reranker re-scores the fused candidate pool before truncation to `k` (see 9.1.1).
* Reranking MUST be **fail-open**: any provider error (missing key, network failure, non-2xx) falls back to the pre-rerank fused order and MUST NOT fail the query.
* The rerank API key follows the same secret-source rules as other provider credentials (16.1.1) and MUST NOT be persisted to the config snapshot.

### 8.5 Self-hosted / OpenAI-compatible provider endpoints

A **self-hosted model server** is a **first-class provider** when it conforms to
the OpenAI-compatible contract: it is declared as a `kind: openai` profile
(§8.1.1) whose `base_url` points at the self-hosted endpoint. **No new `kind` is
introduced** — a self-hosted server is just an `openai`-kind profile on a
non-OpenAI `base_url`, exactly like Ollama/vLLM/LM Studio already are (§8.1.1).

* **Credential-less by default.** A self-hosted endpoint on a trusted network MAY
  have **no `api_key`** and is therefore credential-less (§8.1.1). Credential-less
  self-hosted profiles are still **eligible** for selection and auto-selection
  (§8.1.3) and pass preflight (§2.5) — they are not second-class.
* **Capability mapping** (which OpenAI-compatible route serves each capability):
  * **embed** → `POST /v1/embeddings` (e.g. Hugging Face TEI, vLLM, Infinity).
  * **chat** → `POST /v1/chat/completions`.
  * **stt** → `POST /v1/audio/transcriptions` (e.g. a faster-whisper or
    whisper.cpp server). As with any `kind: openai` audio route, STT here is
    **endpoint-dependent** and **validated at first use** (§8.1.2 footnote ³), not
    statically rejected — an arbitrary self-hosted `base_url` may or may not
    expose `/v1/audio/transcriptions`, and that can only be known when it is
    called.
  * **ocr** has **no OpenAI analog** — OCR is a bespoke surface (§8.1.2 shows
    `ocr` as `❌` for `kind: openai`); a self-hosted OCR server is not reachable
    through this contract.
* **Embed identity.** A self-hosted **embed** endpoint is bound by the
  corpus-lifetime embed identity (§8.1.4) like any other embed provider: changing
  the self-hosted embed model (or its endpoint such that the model changes) forces
  a reindex / `CONFIG_INVALID` on mismatch.
* **STT normalization.** A self-hosted STT response is normalized to the
  `transcript` representation as defined in **§8.6** (transcript representation,
  timestamps, language) — this section does not re-define it; see §8.6.1.
* **No shipped defaults.** dir2mcp ships **no per-deployment default and no
  built-in self-hosted profile** — there is no canonical self-hosted `base_url` to
  guess. The operator MUST declare the self-hosted profile explicitly in config
  (§16.2).

### 8.6 Media transcription, translation, and subtitle surface

> **Status: Planned.** This section defines the normative contract for the media
> surface that absorbs the retired `livevtt archive_transcriber` (dir2mcp #251).
> The contract is **domain-general**: it carries **no** language- or
> broadcaster-specific behavior (no built-in language list, no default target
> language, no station-specific rules). Implementation lands in follow-up dir2mcp
> code PRs once this spec change is merged.

This section refines the audio/STT path (§7.4.C) and adds translation and
subtitle surfaces. All behavior is deterministic so citations and exports are
stable across re-indexing.

#### 8.6.1 Transcript representation and timing

* A transcript is a `transcript` representation (§5.2), `index_kind=text`,
  organized into **time-spanned segments** (`time` spans, `start_ms`/`end_ms`,
  §5.4).
* **Per-segment timestamps MUST** be stored when the provider returns them.
* **Per-word timestamps MAY** be stored when available, in the segment span's
  `extra_json` as a `words` array of `{t, d, w}` (`t` = start ms, `d` = duration
  ms, `w` = word). Word timing is **metadata only**: it MUST NOT create extra
  chunks and MUST NOT change the chunk `text`. Provider-response normalization
  into this shape and the optional surfacing of word granularity in spans and
  citations are defined in §8.6.9; word-level timing is always optional and
  graceful-degrade.
* **No-timestamp fallback.** When a provider returns no timing, the transcript
  falls back to text-size chunking (§7.4.C) and the segments MUST be flagged
  `timing: "estimated"` (in `meta_json` and/or span `extra_json`), so consumers
  know the spans are not provider-authoritative.
* **Deterministic windowing.** Segment/window boundaries MUST be deterministic so
  `time`-span citations are stable across re-indexing (consistent with §8.1.7
  windowing).

#### 8.6.2 Language: detection and optional translation

* **Source language is AUTO-DETECTED by default**; an operator MAY pin it
  (`media.language` / per-provider `stt_language`, §16.2).
* **Translation is OPT-IN and off by default** (`media.translate.enabled: false`).
* **Target language(s)** are configurable (`media.translate.target_langs`) with
  **NO default**. Enabling translation with an **empty** target list is
  `CONFIG_INVALID`.
* **Transcripts are keyed per language.** A transcript representation is
  identified per language using a **`TranscriptLangSuffix`** convention (the
  source-language transcript and each translated transcript are distinct
  representations of the same document). A translated transcript MUST record its
  `source_language` plus the **translation provider/model** that produced it
  (§5.2, §8.6.7).
* **Terminology guidance (optional).** When translation runs on a chat provider
  (`media.translate.engine: chat`), the operator MAY supply a
  **`media.translate.glossary`** — a mapping of source term/phrase → preferred
  rendering — that is injected into the translate prompt as guidance so proper
  nouns, domain terms, and preferred spellings render **consistently**. Because it
  guides the model rather than post-processing output, it handles the target
  language's **morphology** (a term declines/inflects correctly across surface
  forms), which a find/replace cannot. It is **keyed per target language**
  (`glossary: { <bcp47-target>: { "<source term>": "<rendering>" } }`) since a
  rendering is target-language-specific; only the entries for the current target
  language are injected. It applies to **both** the per-line and windowed
  (cross-line-context) translate prompts. Empty/unset = no guidance (today's
  behavior). This is **guidance, not a hard constraint** — the model SHOULD honor
  it but MAY deviate where grammar requires. It is **distinct from
  `media.subtitles.glossary`** (§8.6.3), which is a deterministic, export-time
  find/replace on already-rendered cue text; the two MAY be used together (prompt
  guidance during translation, a regex safety-net at export). It is **domain-general**:
  no built-in terms ship; the map is entirely operator-provided.

#### 8.6.3 Subtitle export

* **VTT and SRT MUST always be available** for any transcribed media: they are
  **derived from the transcript segment spans** (no re-transcription required).
* **TTML and SMIL are OPTIONAL and off by default**
  (`media.subtitles.ttml.enabled: false`). Producing them MAY require additional
  codec/track metadata (e.g. via `ffprobe`); when that metadata is absent the
  export MUST **fail open** (omit TTML/SMIL, do not fail the request). The
  **bilingual** TTML/SMIL packaging contract (cross-language cue alignment, SMIL
  track metadata) is defined in §8.6.10.
* The **exported language is selectable** (any language for which a transcript
  exists, §8.6.2). Requesting an export for a language with no transcript is
  `INVALID_FIELD`.

#### 8.6.4 Sidecar ingestion

* A subtitle **sidecar** (`.vtt`, `.srt`, `.ttml`) sitting next to a media file
  MUST be ingested **as the transcript** for that media **instead of** running STT
  — an authored transcript is authoritative over a machine transcription.
* Sidecar ingestion is **mtime-gated** (§7.6): a sidecar newer than the cached
  transcript triggers re-ingest; `--force` overrides the gate.
* **Multiple sidecars** for one media file (e.g. `clip.en.vtt`, `clip.fr.vtt`)
  produce **per-language transcripts** (§8.6.2 keying).

#### 8.6.5 Variant / multi-rendition selection

* When a corpus contains multiple **renditions of the same media** (e.g. several
  bitrates/resolutions of one recording), they MUST be **grouped by normalized
  name** (`media.variants.group: true`).
* The pipeline transcribes the **canonical/best** rendition **once**
  (`media.variants.select: best`), **deterministically**, and MUST NOT duplicate
  chunks or embeddings across renditions of the same logical media.
* This is the media-specific special case of **cross-file canonicalization**
  (§7.9); `media.variants` and `dedup` share the `best|first` selection vocabulary.

#### 8.6.6 Output quality gates

* STT, OCR, and translation output MUST pass **degenerate-output checks** before
  being indexed. Minimum checks:
  * **empty** output;
  * **repetition / looping** (the classic STT failure mode);
  * **low density vs. duration** (far too little text for the media length).
  * Implementations **SHOULD** additionally flag a **detected language ≠ pinned
    language** mismatch.
* A failed gate is a **non-fatal per-document error** (§7.7): the document is
  marked `status=error` with the appropriate code — `TRANSCRIBE_FAILED`,
  `OCR_FAILED`, or the new `TRANSLATE_FAILED` (§14.4) — and indexing continues.
* The checks MUST be **deterministic** (the same output is judged the same way
  every run).

#### 8.6.7 Representation provenance and re-derivation

* Every **derived** representation (extracted markdown, transcript, translated
  transcript, annotation) MUST record the **provider + model (+ model version)**
  that produced it (§5.2).
* A representation's **derivation identity** is
  `{capability, provider, model, version, language}`. On load, if the configured
  derivation identity for a capability differs from the one recorded on a
  representation, that representation is **stale** and MUST be **re-derived,
  re-chunked, and re-embedded**. This is the runtime analogue of the
  embed-identity → reindex rule (§8.1.4), but **scoped to a single
  representation** rather than the whole index.
* **Sidecar-sourced transcripts are NOT model-derived** (§8.6.4): they have no
  STT provider/model derivation identity and MUST NOT be invalidated by an STT
  model change. (A change to the sidecar file itself still re-ingests via the
  mtime gate, §8.6.4.)

#### 8.6.8 Speaker diarization (optional)

> **Status: Planned.** This subsection defines an **optional** contract for
> **speaker-attributed transcripts** (dir2mcp #266). It is **OFF by default** and
> **provider-dependent**: speaker attribution requires a **diarization-capable
> STT backend** (e.g. a self-hosted WhisperX / pyannote-backed endpoint, §8.5).
> The contract is **domain-general** — no built-in speaker roster, no
> language- or broadcaster-specific behavior. Implementation lands in a follow-up
> dir2mcp code PR once this spec change is merged.

Diarization attributes each transcript segment to a **speaker**. It refines the
transcript representation (§8.6.1) **without changing chunk `text`** — speaker
attribution is **metadata only**: it never edits, reorders, or re-times
transcript content. An implementation MAY, however, introduce a **chunk boundary
at a speaker change** so that every emitted chunk carries a single `speaker` (the
one-`speaker`-per-span model of §5.4/§9.3); this speaker-aligned split is the
only boundary effect diarization may have, and it applies **only when diarization
is active**. A transcript with no speaker attribution MUST chunk **identically**
to the non-diarized path.

* **Off by default; opt-in.** Diarization is enabled via
  `media.diarize.enabled: true` (§16.2). When disabled (the default), transcripts
  carry no speaker attribution and behave exactly as today.
* **Provider-dependent (capability-gated).** Diarization requires a
  diarization-capable STT backend. If `media.diarize.enabled: true` but no
  configured STT provider advertises the diarization capability, startup MUST fail
  `CONFIG_INVALID` with remediation (no silent partial behavior). Consistent with
  capability-driven activation, an implementation MAY additionally **auto-enable**
  diarization when the active STT backend advertises the capability *and* the
  operator has not set `media.diarize.enabled: false`; the tri-state opt-out
  (`false`) always forces it off.
* **Storage.** Per-segment speaker attribution is stored on the segment `time`
  span's `extra_json.speaker` (a stable per-transcript identifier, e.g. `"S1"`),
  with an optional human-readable `extra_json.speaker_label` (§5.4). The
  transcript representation records `diarized: true`, the
  `diarize_provider`/`diarize_model`, and the distinct `speakers` set in its
  `meta_json` (§5.2).
* **Stable, deterministic identifiers.** Speaker identifiers MUST be **stable and
  deterministic across re-indexing** of the same media with the same diarization
  identity, so `speaker`-scoped citations and filters are reproducible. Mapping a
  raw diarization label to a friendly name (`speaker_label`) is optional and MUST
  NOT change the underlying `speaker` identifier.
* **Sidecar speakers.** A subtitle sidecar (§8.6.4) that carries voice/speaker
  markup (e.g. WebVTT `<v Speaker>` cues) MAY populate `speaker`/`speaker_label`
  directly; such a transcript is **not** model-derived for diarization (no
  `diarize_provider`/`diarize_model`), mirroring §8.6.7.
* **Derivation identity.** When diarization is active, the diarization
  provider/model is part of the transcript's derivation identity (§8.6.7): a
  change to the diarization backend invalidates and re-derives the transcript like
  any other capability change.
* **Retrieval and citation surface.** Speaker is **additive** at retrieval time:
  * `dir2mcp_search` MAY accept an optional `speaker` filter (§15.2) that
    restricts time-spanned transcript hits to segments attributed to that speaker;
    a corpus without diarized transcripts simply returns no speaker-filtered hits.
  * A hit `span` of kind `time` MAY surface `speaker`/`speaker_label` (§9.2), and
    human-readable transcript citations MAY append the speaker, e.g.
    `[interview.mp4@t=02:13-02:41 › S2]` (§9.3). The base citation form is
    unchanged when no speaker is present.
* **Export.** Subtitle export (§8.6.3) MAY carry speaker as voice markup when the
  target format supports it (WebVTT `<v>`, TTML voice); formats that cannot
  represent it omit it (fail open, never fail the export).
* **Degenerate output.** Diarization that yields a single speaker for clearly
  multi-speaker audio, or an implausible speaker count, MAY be flagged by the
  output quality gate (§8.6.6) but MUST NOT fail the transcript: a
  diarization-quality concern degrades to a flat (un-attributed) transcript rather
  than `TRANSCRIBE_FAILED`.

#### 8.6.9 Word-level timing: capture, normalization, and surfacing

> **Status: Planned.** This subsection refines §8.6.1's per-word timing rule
> (dir2mcp #252) by defining (a) how a provider's word-level response is
> normalized into the `words` array and (b) how word granularity is **optionally
> surfaced** in spans and citations. Word-level timing is **always optional and
> graceful-degrade**: a transcript with only segment timing remains fully
> conformant. Implementation lands in a follow-up dir2mcp code PR.

Per-segment timing is the conformance baseline (§8.6.1); per-word timing is a
finer, **optional** refinement layered on top of it.

* **Granularity is recorded, not assumed.** A transcript declares its finest
  available granularity in `meta_json` via the `words` flag (§5.2): `words: true`
  iff at least one segment carries a populated `extra_json.words` array. Consumers
  MUST treat absent/`false` as "segment granularity only" and degrade gracefully
  — never error because word timing is missing.
* **Provider normalization (OpenAI-compatible / verbose-JSON).** When an STT
  backend returns word-level timing — e.g. a self-hosted faster-whisper /
  whisper.cpp `/v1/audio/transcriptions` endpoint (§8.5) responding in the
  OpenAI `verbose_json` shape with a `words` array of `{word, start, end}`
  (seconds) — the implementation MUST normalize it into the §8.6.1 `words` shape
  `{t, d, w}` (`t` = start **ms**, `d` = duration **ms**, `w` = word) on the
  owning segment span's `extra_json`. Seconds-to-ms conversion MUST be
  deterministic (round half-up). A response that carries only segment timing
  normalizes to segment spans with no `words` arrays (the existing path,
  unchanged). This parser is a **sibling** to the existing provider segment
  parser, not a replacement: the Mistral `[mm:ss]` segment path is unaffected.
* **Word arrays do not change chunking.** Reaffirming §8.6.1: `words` is metadata
  on the segment span only. It MUST NOT create extra chunks, MUST NOT alter chunk
  `text`, and MUST NOT change segment boundaries. The chunker behaves identically
  whether or not word timing is present, so a transcript chunks the **same** with
  word timing added or removed (deterministic, citation-stable).
* **Optional word-level span surfacing.** A `time` span (§5.4, §9.2) MAY OPTIONALLY
  narrow its `start_ms`/`end_ms` to **word boundaries** drawn from the segment's
  `words` array (for tighter highlighting/deep-linking), provided the narrowed
  span stays **within** the owning segment's bounds. When word timing is absent,
  the span uses segment bounds (the default). This narrowing is a presentation
  refinement: it MUST NOT add or drop hits and MUST NOT change which chunk a span
  belongs to. It is consistent with the word-level deep-linking already permitted
  for `dir2mcp_open_media_clip` (§15.11).
* **Citation form is unchanged.** Word-level surfacing reuses the transcript
  citation form `[path@t=<start>-<end>]` (§9.3); the only difference is that
  `<start>`/`<end>` MAY be word-snapped. No new citation syntax is introduced, and
  a consumer that ignores word timing renders the segment-level citation
  identically.

#### 8.6.10 Bilingual subtitle export (TTML + SMIL)

> **Status: Planned.** This subsection refines §8.6.3's optional TTML/SMIL surface
> (dir2mcp #255) to define **bilingual** packaging precisely. It is **OPTIONAL and
> OFF by default** (`media.subtitles.ttml.enabled: false`), broadcaster-neutral
> (no station-specific rules), and requires the translation surface (§8.6.2) only
> for the bilingual case. Implementation lands in a follow-up dir2mcp code PR.

VTT and SRT are always available and monolingual per export (§8.6.3). TTML and
SMIL are the **optional broadcast-packaging** surface and MAY carry **two
languages in one document**.

* **Off by default.** With `media.subtitles.ttml.enabled: false` (the default),
  no TTML/SMIL is produced and behavior is exactly as in §8.6.3. Enabling the
  surface is purely additive — VTT/SRT remain unaffected.
* **Bilingual TTML.** When enabled with a primary and a secondary language (each
  identified per §8.6.2 keying), TTML export MUST emit, per cue, the primary- and
  secondary-language text aligned to the **same** time region, with each text run
  language-tagged (`xml:lang`). Both languages MUST map back to the **same
  transcript segment span** (§8.6.1) so a TTML cue is traceable to its
  `start_ms`/`end_ms` and chunk. Monolingual TTML (one configured language) is
  also valid.
* **Cross-language cue alignment.** Source and translated transcripts (§8.6.2) are
  distinct representations whose segment boundaries MAY differ slightly. The
  exporter MUST align the secondary language to the primary segment's time region
  within a **configurable tolerance** (`media.subtitles.ttml.align_tolerance_ms`,
  default `2500`); a secondary segment whose start is within tolerance of a
  primary cue is merged into that cue. Alignment MUST be **deterministic** (same
  inputs → identical cues). A secondary segment with no primary cue within
  tolerance is emitted as its own secondary-only cue rather than dropped.
* **SMIL packaging.** When SMIL is produced it describes the media presentation:
  the media reference plus probed track metadata — container/codec, bitrate, and
  video **width/height** when applicable — and references the companion subtitle
  document(s). Track metadata is obtained via `ffprobe` (reusing `internal/avutil`,
  §1) and MAY be cached. SMIL is emitted alongside TTML under the same enable flag.
* **Fail-open on missing metadata.** Consistent with §8.6.3, when required codec/
  track metadata is absent or `ffprobe` is unavailable, the export MUST **fail
  open**: omit SMIL (and any metadata-dependent TTML attributes) and still emit the
  text-bearing subtitle output, never failing the request.
* **Language selection.** The exported primary/secondary languages are selectable
  among languages for which a transcript exists (§8.6.2). Requesting an export for
  a language with no transcript is `INVALID_FIELD` (§8.6.3). The bilingual case
  requires translation (§8.6.2) to be enabled for the secondary language; if it is
  not, the export degrades to monolingual rather than failing.
* **Speaker markup.** Per §8.6.8, TTML export MAY carry speaker as voice markup
  when present; formats/cues that cannot represent it omit it (fail open).

#### 8.6.11 Two-phase batch transcription, progress, and run manifest

> **Status: Planned.** This subsection defines the **batch ergonomics** contract
> for large-archive ingests (dir2mcp #260): an optional two-phase pass split,
> progress reporting, and a resumable run manifest. It is **implementation-agnostic
> but precise**, additive, and changes no per-document representation, chunk, or
> citation. Worker-pool / multi-GPU distribution is explicitly **out of scope**
> (covered separately). Implementation lands in a follow-up dir2mcp code PR.

* **Optional two-phase ingest.** An implementation MAY run media ingest as **two
  ordered passes** over the corpus — a **transcription pass** (STT/sidecar →
  transcript, §8.6.1/§8.6.4) followed by a **derivation pass** (translation §8.6.2
  and subtitle export §8.6.3/§8.6.10). Two-phase mode is **opt-in**; the default
  single-pass per-document pipeline is unchanged. The two-phase split MUST be
  **observably equivalent** to single-pass for the resulting representations,
  chunks, embeddings, and citations — it changes **ordering and reporting only**,
  never output. Either pass MUST be independently **resumable** (a pass picks up
  where it left off using existing identity/cache state, §7.6/§8.6.7), so an
  interrupted transcription pass does not force re-transcription of completed
  assets.
* **Progress semantics.** Progress reporting is **optional and side-channel**: it
  MUST NOT alter representations, chunks, embeddings, citations, ordering of
  results, or error semantics. Progress is reported against a **total unit count**
  established at pass start (e.g. assets, or asset-seconds of media) and is
  **monotonic** within a pass — completed/failed/skipped units only increase. A
  unit resolved from cache (no work performed) counts as **completed** so a resumed
  run reports faithful totals. Progress output is for human/operator consumption
  and is not part of the MCP wire contract.
* **Run manifest (JSONL).** When enabled, a batch run MUST write a **manifest** as
  newline-delimited JSON (one record per asset) for auditability and resume. Each
  record MUST be **self-describing and deterministic** in field set, and MUST
  record at least:
  * **asset identity** — the corpus-relative path (`rel_path`, stable across source
    schemes per §7.8) and the resolved `content_hash` (§7.6);
  * **outcome** — a terminal `status` (`completed` | `skipped` | `error`), and for
    `error` the canonical code (§14.4, e.g. `TRANSCRIBE_FAILED` / `TRANSLATE_FAILED`
    / `OCR_FAILED`) so a manifest is a faithful record of §7.7 per-document outcomes;
  * **media duration** (`duration_ms`, when known) and **processing time** for the
    asset;
  * **outputs produced** — the derived representations and any export artifacts
    (e.g. transcript language(s), translated language(s), subtitle formats emitted).
* **Manifest as resume index.** A manifest MAY be consumed by a subsequent run to
  **skip** assets already terminal in a compatible derivation identity (§8.6.7) and
  to re-attempt `error` assets. The manifest is **advisory for resume** — it MUST
  NOT override the authoritative identity/cache and mtime gates (§7.6, §8.6.4,
  §8.6.7); when the manifest and the live state disagree, the live state wins (the
  manifest can only avoid redundant work, never suppress required re-derivation).
* **Determinism.** Asset processing order within a pass MUST be deterministic so
  manifests and progress are reproducible across runs of an unchanged corpus.

### 8.7 Distributed embedding (coordinator + workers)

> **Status: Planned.** This subsection defines the **optional** contract for
> embedding a corpus with **multiple workers on separate machines** (e.g. a pool
> of GPU hosts) instead of the single in-process embedding loop. It is
> **off by default** and **additive**: a conforming deployment still runs the
> whole pipeline in **one binary on one machine** with no broker (§1.2,
> local-first single-binary default). It is **implementation-agnostic** — it names
> a job-queue *contract*, not a specific broker. Implementation lands in follow-up
> dir2mcp code PRs (dir2mcp #248 distributed workers, dir2mcp #249 standalone
> embed-worker mode) once this spec change is merged.

By default, embedding runs **in-process**: the same binary that discovers,
chunks, stores, and serves also embeds pending chunks (the chunk-level
`embedding_status` machinery of §5.3/§7.6). The distributed mode **separates the
control plane from the embedding compute** so embedding can scale across hosts; it
changes **where embedding happens**, not **what is persisted** — the store shape
(§5), embed identity (§8.1.4), and retrieval contract (§9) are unchanged.

#### 8.7.1 Roles

* **Coordinator (control plane).** Exactly one logical coordinator per corpus does
  discovery (§7.1, §7.8), representation generation (§7.4), chunking (§7.5), store
  ownership (§5), MCP serving (§10), and retrieval (§9). It **enqueues** embedding
  jobs for chunks whose `embedding_status` is `pending` (§5.3) and records results
  written back to the store. The coordinator owns the **local** state directory
  (§1.2) — SQLite metadata and, for the embedded tiers, the vector index.
* **Embed-worker (compute plane).** Zero or more stateless workers (e.g. on GPU
  hosts) **pull** jobs, read the referenced corpus bytes, call the configured
  **embed** provider (§8.1, typically a co-located self-hosted endpoint, §8.5),
  and **write the resulting vectors and chunk status back to the shared store**. A
  worker does **no** discovery, chunking, MCP serving, or retrieval. A standalone
  worker run mode (dir2mcp #249) is exactly this role with no serving
  responsibilities.

The single-binary default is the **degenerate case** of this contract: one process
plays both roles with an in-process queue and no external broker. Enabling the
distributed mode MUST NOT change results versus the in-process default for the
same corpus and embed identity.

#### 8.7.2 Job description

An embedding job MUST identify its work precisely enough that any worker can
execute it without coordinator-relayed payload bytes:

* a **corpus reference** — which corpus/`corpus_id` (§5.5) and the `source`
  binding (§7.8) needed to read bytes via CorpusFS (§7.10);
* a **chunk identity** — the `chunk_id` (§5.3, the ANN label) and the
  `index_kind` (`text|code`, §6.1) so the worker writes to the correct axis;
* a **payload identity** — the chunk's `text_hash` (§5.3) for text chunks, or, for
  a media chunk (§8.1.7), the `rel_path`/media ref plus the chunk's span
  (§5.4 `page`/`time`/`region`) so the worker can fetch and window the exact media
  bytes via CorpusFS range reads (§7.10);
* the **embed identity** (§8.1.4) the job was enqueued under
  (`provider | text_model | code_model | text_dim | code_dim | multimodal`), so a
  worker can **reject** a job whose embed identity does not match its configured
  provider rather than silently writing vectors from the wrong space.

A worker reads corpus bytes **directly from the source** via CorpusFS (§7.10) —
never relayed through the coordinator — so a remote (`s3`/`nfs`) corpus and a
worker pool can share the same bytes without the coordinator becoming a data-plane
bottleneck.

#### 8.7.3 Idempotency, ordering, and identity

* **Idempotent writes.** A job MUST be safe to execute **more than once** (at-least-
  once delivery is assumed). Writing a vector is keyed by `chunk_id` (§6.1), so a
  re-delivered or duplicated job overwrites the same vector and sets the same
  terminal `embedding_status` — re-running a completed job is a no-op, never a
  duplicate vector. A worker MUST NOT assume exactly-once delivery.
* **No global ordering requirement.** Embedding jobs are **independent**; workers
  MAY drain them in any order and in parallel. Retrieval already operates on a
  partial index (§1.2), so chunks becoming searchable in arbitrary order is
  acceptable. The only ordering constraint is causal: a chunk MUST exist in the
  store (enqueued by the coordinator) before a job for it can be claimed.
* **Embed identity is enforced per job.** The embed identity (§8.1.4) is part of
  the job (§8.7.2). A worker whose configured embed provider/model/dim/multimodal
  does not match the job's embed identity MUST fail the job (returning it for
  redelivery or dead-lettering) rather than write a vector — this preserves the
  corpus-lifetime single-space invariant (§6.4, §8.1.4) across a heterogeneous
  worker pool.
* **Failure handling.** A job failure is **non-fatal** to the corpus: the chunk's
  `embedding_status` records `error` (§5.3) and the job MAY be retried (broker
  redelivery) up to an implementation-defined limit, after which it is dead-
  lettered and surfaced as a per-document/per-chunk error (§7.7), exactly as an
  in-process embedding failure is today. A stuck/abandoned in-flight job MUST
  become re-claimable (visibility timeout / lease expiry) so a crashed worker does
  not strand a chunk in `pending` forever.
* **Tombstone safety.** A job for a `chunk_id` that has since been tombstoned
  (`deleted=1`, §6.6) MUST NOT resurrect it: the write either is skipped or is
  harmless because retrieval honors the tombstone (§6.6) regardless of vector
  presence.

#### 8.7.4 Shared store and broker

* **Shared vector store.** Workers and coordinator MUST write to a **shared**
  vector home. The embedded tiers (Tier A/B, §6.2) are **single-node** and are
  therefore **not** a shared store across machines; a distributed worker pool
  REQUIRES an external store reachable by all participants — a **Tier C** backend
  (`qdrant`/`pgvector`, §6.2/§6.3) addressed by the `corpus_id`-derived
  collection/namespace. This is the one configuration where Tier C stops being
  merely optional and becomes a **prerequisite of the distributed mode** — the
  embedded default remains correct for the single-machine case (§1.2). Chunk
  metadata/status (§5.3) likewise lives in a store reachable by all workers.
* **Broker is implementation-defined.** The transport that carries jobs
  (coordinator → workers) is **not** specified here — any queue/broker providing
  at-least-once delivery, a redelivery/visibility mechanism, and a dead-letter
  path satisfies §8.7.3 (e.g. NATS, Redis, SQS). The in-process default needs no
  broker. Broker connection parameters and credentials follow §16.1.1 (resolved
  from a secret source, **never persisted** to the config snapshot), consistent
  with every other provider/store credential.
* **Capability-driven, off by default.** The distributed mode activates only when
  a broker/worker topology is configured; with no such config, the pipeline runs
  the in-process embedding loop unchanged (§1.2). The standalone embed-worker run
  mode (dir2mcp #249) is the worker role packaged without serving — it joins the
  pool, pulls jobs, reads corpus bytes via CorpusFS (§7.10), embeds via its
  configured provider (§8.5), and writes back; it never serves MCP or runs
  discovery.

### 8.8 Detected-language resolution (representation language)

A representation's recorded language (§5.2 `language`, `language_source`,
`language_confidence`) enables multilingual-corpus filtering and per-language
retrieval (§9.5). Recording it is **optional, additive, and best-effort**; it
MUST NOT make ingestion fail.

* **Auto-detect by default; pin optional.** Language detection is **on by default
  and best-effort**: an implementation SHOULD record a representation's language
  when it can determine one (a `transcript` already does, §8.6.2; OCR and plain
  text MAY add it). An operator MAY pin the language (`media.language` /
  per-provider `stt_language`, §16.2; an analogous pin for non-media text is
  implementation-defined and optional). No fixed or default language is assumed —
  the surface is general-purpose and language-agnostic.
* **Resolution precedence.** When more than one signal is available, the recorded
  effective `language` MUST be resolved deterministically with this precedence,
  and `language_source` MUST record which signal won:
  1. **`configured`** — an explicit operator pin always wins (§16.2).
  2. **`declared`** — a language asserted by the source itself (sidecar suffix
     §8.6.4, document/track language tag, OCR-provider-reported language).
  3. **`detected`** — an auto-detector's best-effort result.
  A translated transcript's effective `language` is its **target** language
  (§8.6.2), recorded independently of the above.
* **Graceful degradation (absent ⇒ unknown, never an error).** When no signal is
  available — no pin, no declaration, and detection is unavailable, fails, or
  returns below a configured confidence floor — the representation records **no**
  `language` and is treated as **unknown language**. Unknown is a first-class,
  non-error state: ingestion, indexing, retrieval, and citation all proceed
  exactly as today; only per-language filtering (§9.5) is affected.
* **Confidence floor (optional).** An implementation MAY apply a configured
  minimum confidence at detection time and decline to record a low-confidence
  `detected` language (leaving it unknown). Once a `language` value is written it
  is authoritative for retrieval matching (§9.5); `language_confidence` is
  informational and MUST NOT be re-applied as a filter at query time.
* **Stability & re-derivation.** Detection MUST be deterministic for identical
  input + detector so the recorded language is stable across re-indexing. The
  detector/pin is **not** part of a representation's derivation identity (§8.6.7)
  unless an implementation chooses to make a *pin change* trigger re-derivation;
  a pure detector change MAY refresh `language` opportunistically without forcing
  re-embedding (language metadata does not change chunk `text`).

---

## 9) Retrieval and answer generation

### 9.1 Search routing

At query time:

* `index=auto`:

  * default to `text`
  * choose `code` if query is code-oriented (heuristic) or filters target code
* `index=both`:

  * query both indices and fuse results
  * normalization: per-index score normalization then merge

### 9.1.1 Optional reranking

Reranking is optional; it is a retrieval-quality optimization, not a hard dependency. It is **auto-enabled when a rerank provider credential is present** (e.g. `COHERE_API_KEY`) and disabled otherwise. `rerank.enabled` is an optional override (see 8.4): `false` forces it off even with a credential present; an explicit `true` without a credential MUST fall back (fail-open) and SHOULD warn.

When active, after candidate generation/fusion and **before** truncation to `k`:

* the top `rerank.candidate_pool` (default 50) fused candidates are re-scored by the configured rerank provider (8.4) using the query text and each candidate's `snippet`;
* those candidates are reordered by the provider's relevance score; when `rerank.candidate_pool < k`, the remaining (un-reranked) fused candidates MUST be appended **after** the reranked ones in their original deterministic fused order;
* the combined list is then truncated to `k`.

Rules:

* For `index=both`, reranking is applied **once to the merged candidate pool** (after per-index normalization and merge), not per-index.
* **Fail-open**: any provider error falls back to the pre-rerank fused order, truncated to `k`. A query MUST NOT fail because reranking failed.
* **No result loss**: reranking MUST NOT reduce the result count below what the pre-rerank fused order would return for the same `k`. When `rerank.candidate_pool < k`, the un-reranked fused tail is appended (in fused order) before truncation, so reranking only reorders and never drops results.
* Reranking only reorders results and MAY overwrite `score` with the provider's relevance score; it MUST NOT change the result structure (9.2) or add/remove fields.
* **Determinism**: ties in rerank score MUST be broken deterministically (e.g. by `chunk_id`).

### 9.2 Result structure and provenance

Each hit includes:

* `chunk_id`, `rel_path`, `rep_type`, `score`, `snippet`
* `span` with one of:

  * `lines` (start_line/end_line)
  * `page` (page)
  * `time` (start_ms/end_ms; on a diarized transcript MAY also carry
    `speaker`/`speaker_label`, §8.6.8)

**Cross-file de-duplication.** When `dedup.retrieval: true`, search MUST collapse
candidate hits whose source documents belong to the same duplicate group (§7.9)
to a **single** hit — the best-ranked survivor — keeping the canonical document's
`rel_path` in the surviving hit. This applies whether or not ingest-time
canonicalization (§7.9) is enabled, so a corpus indexed before dedup was turned on
still de-duplicates at query time.

* **Ordering.** De-duplication runs after candidate generation/fusion and
  **before** reranking (§9.1.1) and truncation to `k`, so the *candidate pool*
  shrinks, not the rerank output. This preserves the §9.1.1 **no-result-loss**
  guarantee, which is defined relative to the (now de-duplicated) candidate pool:
  reranking still only reorders and never drops results. Because dedup reduces the
  pool, a query MAY legitimately return fewer than `k` hits when the corpus does
  not contain `k` distinct (non-duplicate) results.
* **Determinism & order preservation.** Collapsing MUST keep the first (best
  pre-rerank) survivor per group and preserve the relative order of survivors.
* **Citations.** Citations (§9.3) reference the surviving (canonical) `rel_path`,
  so an answer never cites two byte-identical sources for the same fact.
* **Default off.** When `dedup.retrieval` is false (default), search returns the
  pre-dedup candidate set exactly as before.

### 9.3 Citation formatting (human-readable)

Within answers, citations must be rendered as:

* code/text: `[path:L<start>-L<end>]`
* pdf OCR: `[path#p=<page>]`
* pdf structured (region): render the primary page (`bbox.page`) as
  `[path#p=<page>]`; when the span covers multiple pages
  (`start_page != end_page`) render the range `[path#p=<start_page>-<end_page>]`.
  Optionally suffix with the section breadcrumb when present, e.g.
  `[report.pdf#p=3 › Results › 3.1 Revenue]`
* transcript: `[path@t=<start>-<end>]` where `<start>/<end>` are `mm:ss` or `ms`.
  `<start>`/`<end>` MAY be word-snapped when the transcript carries per-word timing
  (§8.6.9); the citation **syntax is unchanged** and a consumer that ignores word
  timing renders the segment-level bounds identically. On a diarized transcript
  (§8.6.8) the speaker MAY be appended, e.g.
  `[interview.mp4@t=02:13-02:41 › S2]`; the base form is used when no speaker is
  present.

### 9.4 RAG generation

If enabled:

* build a prompt with:

  * system prompt
  * question
  * retrieved contexts + citations
* return answer text + citations list + underlying hits (structured output)

If disabled or `mode=search_only`:

* return hits only.

### 9.5 Per-language retrieval filter (optional)

`dir2mcp_search` (§15.2) and `dir2mcp_ask` (§15.3) MAY accept an **optional**
`languages` filter that restricts results to representations recorded in one or
more languages (§5.2, §8.8). The filter is **additive and off by default**:
absent or empty ⇒ **no language filtering** and search/ask behave exactly as
today (unchanged results).

* **Argument shape.** `languages` is an array of BCP-47 language tags (e.g.
  `["en"]`, `["pt-BR", "es"]`). An empty array is equivalent to omitting it (no
  filter). The argument is OPTIONAL; existing callers that never send it observe
  no behavior change. An OPTIONAL companion argument `language_match` selects the
  matching mode for the whole array: `"primary"` (the DEFAULT — primary-subtag
  matching, below) or `"strict"` (opt-in region/script narrowing, below). Absent
  or empty ⇒ `"primary"`; existing callers that never send it observe no behavior
  change. An unrecognized `language_match` value is `INVALID_FIELD` (§14).
* **Matching semantics (default — `language_match: "primary"`).** A hit matches
  when its source representation's recorded `language` (§5.2) matches **any**
  requested tag (logical OR across the array). Matching is performed on the
  **BCP-47 primary subtag**, **case-insensitively**: a request for `en` matches a
  representation recorded as `en`, `EN`, or `en-US`, and a request for `pt-BR`
  matches `pt` (primary-subtag match). Region, script, and other subtags MUST NOT
  cause a match to be missed when the primary subtags agree. Implementations MAY
  additionally honor an exact full-tag match but MUST AT LEAST honor
  primary-subtag matching. This is the DEFAULT and is unchanged from prior
  versions; callers that omit `language_match` (or send `"primary"`) observe
  exactly this behavior.
* **Region/script narrowing (opt-in — `language_match: "strict"`).** When the
  caller sets `language_match` to `"strict"`, matching uses **BCP-47 Basic
  Filtering** (RFC 4647 §3.3.1) instead of primary-subtag matching: a requested
  tag matches a recorded `language` **iff** the recorded value equals the
  requested tag or extends it with additional subtags (the recorded tag begins
  with the requested tag followed by a `-` separator), compared
  **case-insensitively** on canonicalized subtags. Under `"strict"`, region,
  script, and variant subtags in the request DO narrow the match: `pt-BR` matches
  representations recorded as `pt-BR` (and `pt-BR-…`) but **not** bare `pt` or
  `pt-PT`; `zh-Hans` matches `zh-Hans`/`zh-Hans-CN` but **not** `zh-Hant` or bare
  `zh`. A request that carries only a primary subtag (e.g. `pt`) still matches
  that primary subtag and all its region/script extensions (`pt`, `pt-BR`,
  `pt-PT`), so `"strict"` narrows **only** to the precision the caller actually
  supplies. The default `"primary"` guarantee that region/script MUST NOT cause a
  miss is **unaffected**: narrowing occurs only when the caller explicitly opts in
  via `language_match: "strict"`.
* **Unknown / absent language.** A representation with **no** recorded language
  (unknown, §8.8) **never** matches a specific language filter — it is excluded
  whenever `languages` is non-empty. When `languages` is absent/empty, unknown
  representations are **unaffected** (returned exactly as today). Implementations
  MAY offer an explicit opt-in sentinel for unknown (e.g. `"und"`, the BCP-47
  "undetermined" tag) to *include* unknown-language hits alongside a filter; this
  is OPTIONAL and, when unsupported, an unrecognized tag simply matches nothing.
* **Translated representations.** A translated transcript (§8.6.2) is recorded
  under its **target** language (§5.2, §8.8) and matches that target; its
  `source_language` is not the matched value. Filtering for a language thus
  returns both source-language representations in that language and translations
  *into* that language, which is the intended multilingual-corpus behavior.
* **Pipeline placement & guarantees.** The language filter is applied at
  **candidate selection** (alongside `path_prefix` / `file_glob` / `doc_types`),
  **before** cross-file de-duplication (§9.2), reranking (§9.1.1), and truncation
  to `k`. It only **removes** non-matching candidates; it MUST NOT reorder, add
  fields, or change the result structure (§9.2) or citation format (§9.3). As
  with any selective filter, a filtered query MAY return fewer than `k` hits.
* **No match is not an error.** A `languages` filter that excludes every
  candidate returns an empty `hits` list (and, for `ask`, an answer grounded in
  no contexts per §9.4) — never an error. An unrecognized or malformed tag value
  (not a syntactically valid BCP-47 tag) is `INVALID_FIELD` (§14); a
  syntactically valid tag that simply matches nothing in the corpus is **not** an
  error.

The filter matches the same recorded representation `language` that ingestion
writes (§8.8), so a corpus indexed before any language was recorded simply has
unknown-language representations that no specific filter matches — there is no
migration and no breaking change.

### 9.6 Date/time-range filter (optional)

`dir2mcp_search` and `dir2mcp_ask` MAY accept optional `date_from` and `date_to`
arguments that restrict results to a **document date window**. The filter is
**additive and off by default**: an absent bound is an open range on that side,
and existing callers that never send them observe no change (dir2mcp #326).

* **Value.** Each bound is an **RFC 3339** timestamp (e.g.
  `2026-04-01T00:00:00Z`) or a bare calendar date `YYYY-MM-DD`. A bare `date_from`
  date is interpreted as the **start** of that day (`00:00:00Z`); a bare `date_to`
  date as the **end** of that day (`23:59:59.999Z`), so a single day is expressed
  as `date_from = date_to = YYYY-MM-DD`. A value that parses as neither, or a
  `date_from` strictly later than `date_to`, is an `INVALID_FIELD` error (§14).
* **What "document date" means.** The bound is compared against the document's
  recorded modification time (`documents.mtime_unix`, §5.1) — the calendar anchor
  every document carries, independent of representation type. (A future
  content-derived "recorded date" MAY refine this per representation; until then
  `mtime` is the anchor.) Media **time** spans (§5.4) are *intra-document offsets*,
  not calendar dates, and are NOT used here.
* **Semantics.** The range is **inclusive** on both provided bounds: a hit is kept
  iff its document date is `>= date_from` (when given) **and** `<= date_to` (when
  given). It composes with the other filters (`path_prefix` / `file_glob` /
  `doc_types` / `languages`, §9.5) by **conjunction**.
* **Pipeline placement.** Applied as a **candidate-selection** filter alongside
  `path_prefix` / `file_glob` / `doc_types` (§9.1), before ranking — so `k` counts
  only in-window hits, and a filtered query MAY return fewer than `k`.
* **No match is not an error.** A window that excludes every hit returns an empty
  result set, not an error, exactly as the language filter (§9.5).

---

## 10) MCP server: Streamable HTTP (2025-11-25)

### 10.1 Endpoint

* Default MCP path: `/mcp`
* POST accepts JSON-RPC messages (single object; batch arrays may be accepted optionally).

### 10.2 Required headers

Clients MUST send:

* `MCP-Protocol-Version: 2025-11-25` (after initialization)
* `Authorization: Bearer <token>` (unless auth disabled)
* `Accept: application/json, text/event-stream` (recommended)

Server returns:

* `MCP-Session-Id: <id>` on initialize response.

### 10.3 Sessions

* On initialize success, server assigns a session id and returns it in `MCP-Session-Id`.
* Client must include `MCP-Session-Id` on subsequent requests.
* Sessions are stateful resources with a defined lifecycle:

  * **Inactivity timeout:** a session SHOULD expire if the server has not seen any requests using that `MCP-Session-Id` for a configurable period. The reference implementation defaults to 24 hours of inactivity (matching the previous hardcoded `sessionTTL`), though some deployments may prefer shorter windows such as 30 minutes. Servers SHOULD expose a configuration parameter (e.g. `session_inactivity_timeout` as a YAML duration) so operators can adjust the value.
  * **Absolute lifetime (optional):** servers MAY enforce a maximum absolute duration (e.g. 24 hours) after which the session expires regardless of activity. In the reference implementation this is governed by `session_max_lifetime` (YAML duration); a zero value disables the limit.
  * **Cleanup/eviction:** expired sessions MUST be evicted or garbage‑collected from the server’s in‑memory or persisted session store. Cleanup can run lazily on access or via a periodic background task; the key requirement is that an expired `MCP-Session-Id` is treated as unknown.
  * **Logging & visibility:** servers SHOULD log session expiration events, including the reason (inactivity vs. lifetime) and the session id. Responses may include a diagnostic header such as `X-MCP-Session-Expired: inactivity|max-lifetime`.

* Unknown or expired session id:

  * server returns HTTP 404. This is the same status used for any non‑existent session; clients SHOULD treat both cases identically even if a diagnostic header is present.
  * client MUST re‑initialize by issuing a fresh `initialize` request. The previous id is discarded and a new `MCP-Session-Id` will be returned. Clients SHOULD treat a 404 as indicating that they should restart the flow rather than retrying.

* **Production guidance:**

  1. Choose default timeout values appropriate for your workload and security requirements. Public‑facing servers often use shorter inactivity timeouts to conserve resources.
  2. Expose configuration knobs for both inactivity and absolute lifetime. Document defaults in your service README.
  3. Surface expiration reasons in logs and, optionally, response headers to assist operators and clients.
  4. Implement robust cleanup to avoid unbounded session growth; periodic eviction or TTL caches are recommended.

### 10.4 Notifications

If a POST is a JSON-RPC notification (no id), and accepted:

* server returns HTTP `202 Accepted` and no body.

### 10.5 Origin checks (DNS rebinding mitigation)

If `Origin` header is present:

* must match allowlist
* otherwise return HTTP 403

### 10.6 Auth

* Bearer token required by default.
* Token storage: `.dir2mcp/secret.token`
* If `--auth file:<path>` is set, the token is loaded from that path, `connection.data.token_source` MUST be `file`, and `connection.data` SHOULD include `token_file` (or `token_source_details.path`).
* Tokens must not be embedded in URLs by default (avoid `?token=` in docs/outputs).

---

## 11) MCP lifecycle (wire-level)

All JSON-RPC messages are POSTed to the MCP endpoint.

### 11.1 `initialize` request (example)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": { "tools": { } },
    "clientInfo": { "name": "example-client", "version": "0.0.1" }
  }
}
```

### 11.2 `initialize` response (example)

HTTP response headers include:

* `MCP-Session-Id: sess_...`

`serverInfo.name` is per-instance: by default it is auto-derived as
`dir2mcp-<slug>-<6-hex>` from the absolute path of the indexed directory
so that operators running many `dir2mcp` instances can distinguish them
in their MCP client list. Builds whose embedded version is recognized
as a dev version (specifically `0.0.0-dev` or `dev-<sha>[+dirty]`) use
a `dir2mcp-dev-<slug>-<6-hex>` prefix so local dev binaries can coexist
with brew-installed releases without identity collision. Other non-release
builds, including `go install` snapshots or pseudo-versions, still use
the normal `dir2mcp-<slug>-<6-hex>` prefix. It can be overridden via the
`server.name` YAML key or the `DIR2MCP_SERVER_NAME` env variable;
overrides apply verbatim regardless of build type.

Body:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "tools": { "listChanged": false }
    },
    "serverInfo": {
      "name": "dir2mcp-stas-legal-a1b2c3",
      "title": "dir2mcp: Directory RAG MCP Server",
      "version": "0.7.0"
    },
    "instructions": "Use tools/list then tools/call. Results include citations."
  }
}
```

The server **MUST** advertise this spec's pinned `protocolVersion` (`2025-11-25`,
§1.3) in the response rather than echoing back whatever value the client sent in
its `initialize` request; a client that requested a different version thereby
learns the server's actual protocol generation instead of having its own guess
reflected (closes #404).

### 11.3 `notifications/initialized` (example)

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

Server returns: HTTP 202.

---

## 12) MCP tools: list and call

### 12.1 Tool naming

All tools are prefixed with `dir2mcp_`. The historical dotted form `dir2mcp.<tool>` is **superseded** as of spec `0.5.0` (see `spec/versioning.md`).

### 12.2 Tool discovery: `tools/list`

Request:

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {} }
```

Response contains an array of tools; each tool MUST include:

* `name`
* `description`
* `inputSchema` (valid JSON Schema object)
* (recommended) `outputSchema` (valid JSON Schema object)

### 12.3 Tool invocation: `tools/call`

Request:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": { "name": "dir2mcp_search", "arguments": { "query": "..." } }
}
```

### 12.4 Tool result contract (MCP-native)

Tool call responses MUST return:

* `result.content[]` (at least one item)
* `result.structuredContent` when supported by negotiated version
* `result.isError` true for tool execution failures (not JSON-RPC error)

Example success:

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

Example tool execution error:

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

---

## 13) Tool set (core + recommended + optional)

### 13.1 Core tool set

* `dir2mcp_search`
* `dir2mcp_ask`
* `dir2mcp_open_file`
* `dir2mcp_list_files`
* `dir2mcp_stats`

### 13.2 Recommended extended tools

* `dir2mcp_transcribe` (audio → transcript, uses configured provider)
* `dir2mcp_annotate` (document → structured JSON + flattened text)
* `dir2mcp_transcribe_and_ask` (audio → transcript → ask)
* `dir2mcp_open_media_clip` (media hit → extracted audio/video snippet for a time span; §15.11)

### 13.3 Optional extension

* `dir2mcp_ask_audio` (answer → audio via ElevenLabs TTS)

---

## 14) Error taxonomy (canonical codes)

### 14.1 Auth/transport

* `UNAUTHORIZED` (missing/invalid token)
* `FORBIDDEN_ORIGIN` (Origin not allowed)
* `SESSION_NOT_FOUND` (invalid MCP-Session-Id)
* `BIND_FAILED` (cannot bind host/port)
* `TLS_CONFIG_INVALID`

### 14.2 Input validation

* `MISSING_FIELD`
* `INVALID_FIELD`
* `INVALID_RANGE`
* `CLIP_TOO_LARGE` (returned by `dir2mcp_open_media_clip` when the requested time
  span exceeds the configured maximum clip duration/size bound (§15.11);
  **non-retryable** — the caller must request a shorter span)
* `FORBIDDEN` (path/content blocked by policy)
* `PATH_OUTSIDE_ROOT`
* `FILE_NOT_FOUND`
* `DOC_TYPE_UNSUPPORTED`
* `OCR_NOT_READY` (returned by `dir2mcp_open_file` for binary doc types — PDF, audio — when no OCR/transcript representation is cached yet; retryable once ingestion completes)
* `MEDIA_NO_TEXT` (returned by `dir2mcp_open_file` for a `replace`-mode multimodal media chunk (8.1.7) that has **no** text representation; **non-retryable** — the gap is permanent, unlike `OCR_NOT_READY`; the hit can still be cited)

### 14.3 Index/state

* `CONFIG_INVALID`
* `STORE_CORRUPT`
* `INDEX_VERSION_MISMATCH`
* `INDEX_NOT_READY` (should be rare; prefer partial results)

### 14.4 Ingestion/extraction

* `EXTRACT_FAILED`
* `OCR_FAILED` — also covers an OCR output **rejected by the degenerate-output
  quality gate** (§8.6.6), not only a provider/transport failure.
* `TRANSCRIBE_FAILED` — also covers a transcript output **rejected by the
  degenerate-output quality gate** (§8.6.6) (empty / repetition / low density),
  not only a provider/transport failure.
* `TRANSLATE_FAILED` — translation failed, including a translation output
  rejected by the degenerate-output quality gate (§8.6.6).
* `MEDIA_CLIP_FAILED` — clip extraction failed (returned by
  `dir2mcp_open_media_clip`, §15.11): the underlying media is unreadable, the
  extraction tool (e.g. `ffmpeg`) is unavailable, or the segment extraction
  errored. Distinct from `CLIP_TOO_LARGE` (a bounds rejection) and
  `MEDIA_NO_TEXT` (a missing-text condition on `open_file`).
* `ANNOTATE_FAILED`
* `FILE_TOO_LARGE`
* `BINARY_SKIPPED`

### 14.5 Provider/API

* `MISTRAL_AUTH`
* `MISTRAL_RATE_LIMIT`
* `MISTRAL_FAILED`
* `ELEVENLABS_AUTH`
* `ELEVENLABS_RATE_LIMIT`
* `ELEVENLABS_FAILED`

Each tool error returns:

* `code`, `message`, `retryable` boolean.

---

## 15) Tool specifications (full schemas)

All schemas are JSON Schema (draft-agnostic, compatible with common validators).

### 15.1 Shared types

#### 15.1.1 `Span`

```json
{
  "type": "object",
  "oneOf": [
    {
      "additionalProperties": false,
      "properties": { "kind": { "const": "lines" }, "start_line": { "type": "integer" }, "end_line": { "type": "integer" } },
      "required": ["kind", "start_line", "end_line"]
    },
    {
      "additionalProperties": false,
      "properties": { "kind": { "const": "page" }, "page": { "type": "integer" } },
      "required": ["kind", "page"]
    },
    {
      "additionalProperties": false,
      "properties": {
        "kind": { "const": "time" },
        "start_ms": { "type": "integer" },
        "end_ms": { "type": "integer" },
        "speaker": { "type": "string", "description": "Optional (§8.6.8): stable per-transcript speaker id on a diarized transcript." },
        "speaker_label": { "type": "string", "description": "Optional human-readable speaker name (§8.6.8)." }
      },
      "required": ["kind", "start_ms", "end_ms"]
    },
    {
      "additionalProperties": false,
      "properties": {
        "kind": { "const": "region" },
        "start_page": { "type": "integer" },
        "end_page": { "type": "integer" },
        "bbox": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "page": { "type": "integer" },
            "l": { "type": "number" }, "t": { "type": "number" },
            "r": { "type": "number" }, "b": { "type": "number" },
            "coord_origin": { "enum": ["TOPLEFT", "BOTTOMLEFT"] }
          },
          "required": ["page", "l", "t", "r", "b", "coord_origin"]
        },
        "section": { "type": "array", "items": { "type": "string" } }
      },
      "required": ["kind", "start_page", "end_page", "bbox"]
    },
    {
      "additionalProperties": false,
      "properties": { "kind": { "const": "document" } },
      "required": ["kind"]
    }
  ]
}
```

The `region` variant is emitted by structured document extraction (§7.4.B). It
localizes a chunk to a page range (`start_page`/`end_page`, equal when
single-page) and always carries a bounding box (`bbox`); an element without
provenance is recorded as a `page` span instead, never a `region` span with a
missing `bbox` (§7.4.B). The section breadcrumb (`section`) is optional (`[]`
when none). The `region` kind and its `section` field are additive: clients that
do not recognize the `region` kind, or that ignore `section`, MUST degrade
gracefully (treat as a page-level citation on `start_page`).

The `document` variant is emitted by `dir2mcp_open_file` when the requested
`rel_path` is a binary doc type (PDF, audio) and the caller did not supply
`page`, `start_ms/end_ms`, or `start_line/end_line`. It signals that
`content` is the full OCR / transcript representation rather than a paged or
timed slice.

#### 15.1.2 `Hit`

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "chunk_id": { "type": "integer" },
    "rel_path": { "type": "string" },
    "doc_type": { "type": "string" },
    "rep_type": { "type": "string" },
    "score": { "type": "number" },
    "snippet": { "type": "string" },
    "span": { "$ref": "#/definitions/Span" }
  },
  "required": ["chunk_id", "rel_path", "score", "snippet", "span"]
}
```

> Note: In `tools/list`, you will inline these definitions or include them in each tool’s `outputSchema` as `definitions`.

---

### 15.2 `dir2mcp_search`

**Description:** semantic retrieval across indexed content.

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "query": { "type": "string", "minLength": 1 },
    "k": { "type": "integer", "minimum": 1, "maximum": 50, "default": 15 },
    "index": { "type": "string", "enum": ["auto", "text", "code", "both"], "default": "auto" },
    "path_prefix": { "type": "string" },
    "file_glob": { "type": "string" },
    "doc_types": { "type": "array", "items": { "type": "string" } },
    "speaker": { "type": "string", "description": "Optional (§8.6.8): restrict time-spanned transcript hits to this speaker id. A corpus without diarized transcripts returns no speaker-filtered hits." },
    "languages": { "type": "array", "items": { "type": "string" }, "description": "Optional (§9.5): restrict hits to representations recorded in any of these BCP-47 languages (case-insensitive primary-subtag match). Absent/empty = no filtering. Unknown-language representations never match a specific filter." },
    "date_from": { "type": "string", "description": "Optional (§9.6): RFC 3339 timestamp or bare YYYY-MM-DD date; exclude hits from documents dated before this (inclusive; a bare date = start of that day UTC). Absent = open lower bound." },
    "date_to": { "type": "string", "description": "Optional (§9.6): RFC 3339 timestamp or bare YYYY-MM-DD date; exclude hits from documents dated after this (inclusive; a bare date = end of that day UTC). Absent = open upper bound." }
  },
  "required": ["query"]
}
```

**Output schema (structuredContent):**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "query": { "type": "string" },
    "k": { "type": "integer" },
    "index_used": { "type": "string", "enum": ["text", "code", "both"] },
    "hits": {
      "type": "array",
      "items": { "$ref": "#/definitions/Hit" }
    },
    "indexing_complete": { "type": "boolean" }
  },
  "required": ["query", "hits", "indexing_complete"]
}
```

**content[] requirements:**

* At least one `text` item summarizing results (top hits + citations).

---

### 15.3 `dir2mcp_ask`

**Description:** RAG answer with citations; can run search-only.

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "question": { "type": "string", "minLength": 1 },
    "k": { "type": "integer", "minimum": 1, "maximum": 50, "default": 15 },
    "mode": { "type": "string", "enum": ["answer", "search_only"], "default": "answer" },
    "index": { "type": "string", "enum": ["auto", "text", "code", "both"], "default": "auto" },
    "path_prefix": { "type": "string" },
    "file_glob": { "type": "string" },
    "doc_types": { "type": "array", "items": { "type": "string" } },
    "languages": { "type": "array", "items": { "type": "string" }, "description": "Optional (§9.5): restrict retrieved contexts to representations recorded in any of these BCP-47 languages (case-insensitive primary-subtag match). Absent/empty = no filtering. Unknown-language representations never match a specific filter." },
    "date_from": { "type": "string", "description": "Optional (§9.6): RFC 3339 timestamp or bare YYYY-MM-DD date; exclude contexts from documents dated before this (inclusive; a bare date = start of that day UTC). Absent = open lower bound." },
    "date_to": { "type": "string", "description": "Optional (§9.6): RFC 3339 timestamp or bare YYYY-MM-DD date; exclude contexts from documents dated after this (inclusive; a bare date = end of that day UTC). Absent = open upper bound." }
  },
  "required": ["question"]
}
```

**Output schema (structuredContent):**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "question": { "type": "string" },
    "answer": { "type": "string" },
    "citations": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "chunk_id": { "type": "integer" },
          "rel_path": { "type": "string" },
          "span": { "$ref": "#/definitions/Span" }
        },
        "required": ["chunk_id", "rel_path", "span"]
      }
    },
    "hits": { "type": "array", "items": { "$ref": "#/definitions/Hit" } },
    "indexing_complete": { "type": "boolean" }
  },
  "required": ["question", "citations", "hits", "indexing_complete"]
}
```

**content[] requirements:**

* `text` item containing the final answer (if mode=answer and generation enabled) with inline citations.

---

### 15.4 `dir2mcp_open_file`

**Description:** open an exact source slice for verification (lines/page/time).

*Implementation note:* before reading or returning any data, the server MUST run the
requested `rel_path` and any extracted content through the configured exclusion
engine (pattern matcher + path excludes).  If a match occurs the tool **must not**
return the secret content; it should either return an error (e.g. `FORBIDDEN`)
or an empty/plain-text placeholder.  This ensures tool-level bypass of ingestion
filters is impossible.

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string", "minLength": 1 },
    "start_line": { "type": "integer", "minimum": 1 },
    "end_line": { "type": "integer", "minimum": 1 },
    "page": { "type": "integer", "minimum": 1 },
    "start_ms": { "type": "integer", "minimum": 0 },
    "end_ms": { "type": "integer", "minimum": 0 },
    "max_chars": { "type": "integer", "minimum": 200, "maximum": 50000, "default": 20000 }
  },
  "required": ["rel_path"]
}
```

**Selection rules:**

* If `page` provided → return OCR page text (if available; else error `DOC_TYPE_UNSUPPORTED`).
* Else if `start_ms/end_ms` provided → return transcript excerpt (if available).
* Else if `start_line/end_line` provided → return file lines.
* Else default:

  * for text/code/markdown/html: return first `max_chars` of the file with no `span` set,
  * for PDF: return the cached full-document OCR markdown with `span.kind="document"`; if the OCR cache hasn't been populated yet (e.g. ingest is still running) the tool MUST return error `OCR_NOT_READY` rather than the raw bytes,
  * for audio: return the cached full-document transcript with `span.kind="document"`; same `OCR_NOT_READY` semantics as PDF when no transcript exists yet.
  * for a `replace`-mode multimodal media chunk with no text representation (8.1.7): return the **non-retryable** `MEDIA_NO_TEXT` (the absence is permanent — distinct from `OCR_NOT_READY`), never the raw media bytes.

The handler MUST NOT emit raw binary bytes through `content[].text` — that
field is documented as text. PDFs and audio without a span argument resolve
through the OCR / transcript cache, never through a direct file read.

**Output schema (structuredContent):**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string" },
    "doc_type": { "type": "string" },
    "span": { "$ref": "#/definitions/Span" },
    "content": { "type": "string" },
    "truncated": { "type": "boolean" }
  },
  "required": ["rel_path", "doc_type", "content", "truncated"]
}
```

---

### 15.5 `dir2mcp_list_files`

**Description:** list files under root for navigation and filter selection.

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "path_prefix": { "type": "string" },
    "glob": { "type": "string" },
    "limit": { "type": "integer", "minimum": 1, "maximum": 5000, "default": 200 },
    "offset": { "type": "integer", "minimum": 0, "default": 0 }
  }
}
```

**Output schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "limit": { "type": "integer" },
    "offset": { "type": "integer" },
    "total": { "type": "integer" },
    "files": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "rel_path": { "type": "string" },
          "doc_type": { "type": "string" },
          "size_bytes": { "type": "integer" },
          "mtime_unix": { "type": "integer" },
          "status": { "type": "string", "enum": ["ok", "skipped", "error"] },
          "deleted": { "type": "boolean" }
        },
        "required": ["rel_path", "doc_type", "size_bytes", "mtime_unix", "status", "deleted"]
      }
    }
  },
  "required": ["limit", "offset", "total", "files"]
}
```

---

### 15.6 `dir2mcp_stats`

**Description:** status/progress/health for indexing and models.

**Input schema:**

```json
{ "type": "object", "additionalProperties": false }
```

**Output schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "root": { "type": "string" },
    "state_dir": { "type": "string" },
    "protocol_version": { "type": "string" },
    "format_version": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$", "description": "Optional (SHOULD, §1.3): payload-shape semver (df-000). Absent on older producers; a client MUST treat absence as a pre-format_version baseline shape and MUST NOT infer shape from protocol_version. Independent of the spec version." },
    "doc_counts": { "type": "object", "additionalProperties": { "type": "integer" } },
    "total_docs": { "type": "integer" },
    "doc_counts_available": { "type": "boolean" },
    "indexing": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "job_id": { "type": "string" },
        "running": { "type": "boolean" },
        "mode": { "type": "string", "enum": ["incremental", "full"] },
        "scanned": { "type": "integer" },
        "indexed": { "type": "integer" },
        "skipped": { "type": "integer" },
        "deleted": { "type": "integer" },
        "representations": {
          "type": "integer",
          "minimum": -1,
          "description": "Number of representations created/updated. -1 means not derivable (ListFiles-only fallback path); treat as unavailable, not as an error."
        },
        "chunks_total": {
          "type": "integer",
          "minimum": -1,
          "description": "Total chunks known/created. -1 means not derivable (ListFiles-only fallback path); treat as unavailable, not as an error."
        },
        "embedded_ok": {
          "type": "integer",
          "minimum": -1,
          "description": "Chunks embedded successfully. -1 means not derivable (ListFiles-only fallback path); treat as unavailable, not as an error."
        },
        "errors": { "type": "integer" }
      },
      "required": ["job_id", "running", "mode", "scanned", "indexed", "skipped", "deleted", "representations", "chunks_total", "embedded_ok", "errors"]
    },
    "models": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "embed_text": { "type": "string" },
        "embed_code": { "type": "string" },
        "ocr": { "type": "string" },
        "stt_provider": { "type": "string", "description": "STT provider name; not a closed enum (any STT-capable provider per §8.2, e.g. mistral|elevenlabs|openai|gemini|self-hosted)." },
        "stt_model": { "type": "string" },
        "chat": { "type": "string" }
      },
      "required": ["embed_text", "embed_code", "ocr", "stt_provider", "stt_model", "chat"]
    },
    "recent_failures": {
      "type": "array",
      "description": "Optional. Up to recent_failures_limit (default 20) of the most-recent documents with status='error', newest first by mtime_unix. Each entry carries a short, sanitized error_message explaining why ingest failed (extraction crash, representation generation failure). Implementations MAY omit this field when no failures are recorded; clients MUST treat omission as 'no recent failures', not as 'unsupported'. Intended for diagnostic UIs (doctor-style consoles); the per-failure detail also surfaces in dir2mcp support-bundle's list-files.json.",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "rel_path": { "type": "string" },
          "doc_type": { "type": "string" },
          "mtime_unix": { "type": "integer" },
          "error_message": {
            "type": "string",
            "description": "Short, single-line, length-capped (implementations SHOULD cap at 512 bytes on a UTF-8 rune boundary) explanation of why this document failed ingest. Control characters MUST be stripped so the field renders as one line. Never contains secrets or raw file content."
          }
        },
        "required": ["rel_path", "doc_type", "mtime_unix", "error_message"]
      }
    },
    "skip_reasons": {
      "type": "array",
      "description": "Optional honest-coverage breakdown: one entry per distinct reason a document was set to status='skipped' during ingest, with the count of documents skipped for that reason across the current corpus. Aggregated in CorpusStats parallel to recent_failures. Unlike doc_counts (which groups ALL non-deleted documents by doc_type regardless of status — a skipped .odt and an extracted .docx both count as 'document' — and therefore overstates coverage), this field reports what was NOT indexed and why. Clients MUST NOT read doc_counts as an indexed-document count. Implementations MAY omit this field when nothing was skipped; clients MUST treat omission as 'nothing skipped', not as 'unsupported'. Entries whose count would be 0 MUST be omitted (the array carries only non-empty reasons; an empty corpus omits the field entirely). Intended for coverage / 'what wasn't indexed & why' UIs; the same breakdown also surfaces in dir2mcp support-bundle.",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "reason": {
            "type": "string",
            "enum": ["unsupported_format", "binary_ignored", "archive", "ignore_rule", "secret_excluded", "path_excluded", "size_cap"],
            "description": "Stable skip-reason enum. unsupported_format: extension/MIME has no extractor (e.g. .odt, .rtf, encrypted PDF, image outside the OCR allowlist, video with no sidecar). binary_ignored: detected-binary file with no text representation. archive: an archive container itself, or a nested archive member not expanded. ignore_rule: excluded by an .gitignore/.dir2mcpignore-style rule. secret_excluded: withheld because it matched secret-detection. path_excluded: excluded by a configured path/glob exclusion. size_cap: exceeded the configured max file size. This enum is closed for a given spec minor; new reasons are introduced only by a minor version bump (additive), so a client MAY receive a value it does not recognize from a newer server and SHOULD render it verbatim rather than error."
          },
          "count": {
            "type": "integer",
            "minimum": 1,
            "description": "Number of documents skipped for this reason in the current corpus. Always >= 1 (zero-count reasons are omitted)."
          }
        },
        "required": ["reason", "count"]
      }
    }
  },
  "required": ["root", "state_dir", "protocol_version", "doc_counts", "total_docs", "doc_counts_available", "indexing", "models"]
}
```

---

### 15.7 `dir2mcp_transcribe` (recommended)

**Description:** force transcription for an audio file, persist transcript representation, and (optionally) return segments.

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string", "minLength": 1 },
    "language": { "type": "string" },
    "timestamps": { "type": "boolean", "default": true },
    "retranscribe": { "type": "boolean", "default": false }
  },
  "required": ["rel_path"]
}
```

**Output schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string" },
    "stt_provider": { "type": "string", "description": "STT provider name; not a closed enum (any STT-capable provider per §8.2)." },
    "model": { "type": "string" },
    "indexed": { "type": "boolean" },
    "segments": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "start_ms": { "type": "integer" },
          "end_ms": { "type": "integer" },
          "text": { "type": "string" }
        },
        "required": ["start_ms", "end_ms", "text"]
      }
    }
  },
  "required": ["rel_path", "stt_provider", "model", "indexed"]
}
```

---

### 15.8 `dir2mcp_annotate` (recommended)

**Description:** run structured extraction on a document with provided JSON schema; store JSON; optionally index flattened text.

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string", "minLength": 1 },
    "schema_json": { "type": "object" },
    "index_flattened_text": { "type": "boolean", "default": true }
  },
  "required": ["rel_path", "schema_json"]
}
```

**Output schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string" },
    "stored": { "type": "boolean" },
    "flattened_indexed": { "type": "boolean" },
    "annotation_json": { "type": "object" },
    "annotation_text_preview": { "type": "string" }
  },
  "required": ["rel_path", "stored", "flattened_indexed", "annotation_json"]
}
```

---

### 15.9 `dir2mcp_transcribe_and_ask` (recommended)

**Description:** ensure transcript exists (transcribe if missing/stale), then answer a question using transcript (and optionally whole corpus if configured).

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string", "minLength": 1 },
    "question": { "type": "string", "minLength": 1 },
    "k": { "type": "integer", "minimum": 1, "maximum": 50, "default": 15 }
  },
  "required": ["rel_path", "question"]
}
```

**Output schema:** same as `dir2mcp_ask` output schema, plus:

* `stt_provider`, `transcript_model`, and `transcribed` boolean.

---

### 15.10 `dir2mcp_ask_audio` (optional extension)

**Description:** same as `ask` but includes audio output (TTS). Optional and additive. The input schema inherits all fields of `dir2mcp_ask` (`question`, `k`, `mode`, `index`, `path_prefix`, `file_glob`, `doc_types`, `languages`, `date_from`, `date_to`) plus the audio-specific fields shown below.

Input is the same as `dir2mcp_ask`, with additive audio options:
- `voice_id` (optional)
- `format` (optional; `mp3` or `wav`, default `mp3`)

Input schema (audio-specific fields; the rest mirror `dir2mcp_ask`):

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "question": { "type": "string", "minLength": 1 },
    "k": { "type": "integer", "minimum": 1, "maximum": 50, "default": 15 },
    "voice_id": { "type": "string" },
    "format": { "type": "string", "enum": ["mp3", "wav"], "default": "mp3" }
  },
  "required": ["question"]
}
```

Tool result `content[]` must include:

* `text` item for answer
* `audio` item with base64 payload and mimeType

---

### 15.11 `dir2mcp_open_media_clip` (recommended)

> **Status: Planned.** Returns the **actual audio/video snippet** for a media
> search/ask hit (dir2mcp #264), rather than only a `path@t=...` citation. It is
> the time-media analogue of `dir2mcp_open_file`: where `open_file` returns the
> **transcript text** for a `time` span, `open_media_clip` returns the **extracted
> media bytes** for that span. It is **additive** and lands in a follow-up dir2mcp
> code PR.

**Description:** extract and return the media snippet for a transcript/media hit,
identified either by `chunk_id` (resolved to its source media + `time` span) or
by an explicit `rel_path` + `start_ms`/`end_ms` range.

**Relationship to `dir2mcp_open_file`.** `open_file` with `start_ms/end_ms` on an
audio document returns the **transcript excerpt** (text). `open_media_clip`
returns the **media bytes** for the same span. Callers verifying *what was said*
use `open_file`; callers that need a *playable snippet* use `open_media_clip`. The
two share span semantics (`time`, §5.4) so a single hit can be cited, read as
text, and played.

**Selection rules:**

* If `chunk_id` is provided, the server resolves it to its source media
  (`rel_path` / media ref) and the chunk's `time` span. An explicit
  `start_ms`/`end_ms` provided alongside `chunk_id` overrides the chunk's span
  (still bounded to the same source media).
* Else `rel_path` plus `start_ms`/`end_ms` MUST be provided.
* The target document MUST be audio/video; a non-media `rel_path` returns
  `DOC_TYPE_UNSUPPORTED`. A missing source returns `FILE_NOT_FOUND`. A
  `start_ms >= end_ms` (or out-of-bounds) range returns `INVALID_RANGE`.

**Bounds (normative).** Implementations MUST enforce a **maximum clip duration**
(`media.clip.max_duration_ms`, default 120000 = 2 min) and a **maximum clip byte
size** (`media.clip.max_bytes`, default 25 MiB), §16.2. A request whose span
exceeds the duration bound, or whose extraction would exceed the byte bound,
returns the **non-retryable** `CLIP_TOO_LARGE`; the caller must request a shorter
span. Extraction failures (unreadable media, missing `ffmpeg`) return
`MEDIA_CLIP_FAILED` (§14.4).

**Return shape.** The server returns the clip in **one** of two modes selected by
`return` (default `inline`):

* `inline` — the clip is returned **base64-encoded** in the structured output
  (`data` + `mime_type`) and as an `audio`/`video`-typed `content[]` item. Inline
  return is subject to the byte bound above.
* `reference` — the clip is materialized to a short-lived, server-managed location
  and a `uri` (plus `expires_unix`) is returned instead of bytes, for clients that
  fetch out-of-band. Implementations that do not support `reference` MUST fall
  back to `inline` (and SHOULD note it), never error solely because `reference`
  was requested.

The handler MUST NOT emit raw binary bytes through a `text` content item (media
bytes travel only via `data`/`uri`). Exclusion-engine and x402 gating that apply
to `open_file` (§15.4, §17) apply equally to `open_media_clip`.

**Word-level deep-linking (optional refinement).** When the source transcript
carries per-word timing (§8.6.1 `words`), an implementation MAY accept the same
`start_ms`/`end_ms` snapped to word boundaries for tighter clips; this is an
optional refinement and MUST NOT change the bounds or error semantics above.

**Input schema:**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "chunk_id": { "type": "integer" },
    "rel_path": { "type": "string", "minLength": 1 },
    "start_ms": { "type": "integer", "minimum": 0 },
    "end_ms": { "type": "integer", "minimum": 0 },
    "return": { "type": "string", "enum": ["inline", "reference"], "default": "inline" }
  },
  "anyOf": [
    { "required": ["chunk_id"] },
    { "required": ["rel_path", "start_ms", "end_ms"] }
  ]
}
```

**Output schema (structuredContent):**

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rel_path": { "type": "string" },
    "doc_type": { "type": "string" },
    "span": { "$ref": "#/definitions/Span" },
    "mime_type": { "type": "string" },
    "duration_ms": { "type": "integer" },
    "size_bytes": { "type": "integer" },
    "return": { "type": "string", "enum": ["inline", "reference"] },
    "data": { "type": "string", "contentEncoding": "base64", "description": "Present when return=inline: base64 clip bytes." },
    "uri": { "type": "string", "description": "Present when return=reference: short-lived fetch URI." },
    "expires_unix": { "type": "integer", "description": "Present when return=reference: expiry of uri." }
  },
  "required": ["rel_path", "doc_type", "span", "mime_type", "return"]
}
```

Tool result `content[]` MUST include an `audio`- or `video`-typed item carrying
the clip (base64 `data` + `mimeType`) when `return=inline`; for
`return=reference` the `content[]` carries a text item with the `uri` and a
`resource_link` where supported.

---

## 16) Configuration (single file)

### 16.1 Precedence

1. CLI flags
2. env vars
3. `.dir2mcp.yaml`
4. defaults

### 16.1.1 Secret source precedence

For credential material (API keys/tokens), the runtime resolves sources in this order:

1. Explicit env var (for example `MISTRAL_API_KEY`, `ELEVENLABS_API_KEY`, `DIR2MCP_AUTH_TOKEN`)
2. OS keychain entry (when configured and supported)
3. Configured secret file reference (for example `security.auth.token_file`)
4. Interactive session-only value (not persisted; TTY flow only)

The config snapshot (`.dir2mcp.yaml.snapshot`) MUST record secret source metadata (env/keychain/file/session) and MUST NOT contain plaintext secrets.

### 16.2 Minimal config template (dual STT, 2025-11-25)

```yaml
version: 1

# Provider profiles (built-ins exist; declare only what you override).
# `kind` selects the adapter/wire protocol; credentials follow 16.1.1
# and are never persisted to the snapshot.
providers:
  mistral:                                 # chat+embed are OpenAI-shaped
    kind: openai
    base_url: https://api.mistral.ai/v1
    api_key: ${MISTRAL_API_KEY}
  mistral-ocr:                             # native /v1/ocr (non-OpenAI)
    kind: mistral
    api_key: ${MISTRAL_API_KEY}
  openai:
    kind: openai
    api_key: ${OPENAI_API_KEY}
  openrouter:
    kind: openai
    base_url: https://openrouter.ai/api/v1
    api_key: ${OPENROUTER_API_KEY}
  anthropic:
    kind: anthropic
    api_key: ${ANTHROPIC_API_KEY}
  local:
    kind: openai
    base_url: http://localhost:11434/v1    # Ollama / vLLM / LM Studio

# Per-capability bindings. `provider` unset => auto-select the first
# credentialed profile that can serve the capability (8.1.3).
model:
  embed:                                   # reindex-bound (8.1.4)
    provider: mistral
    text_model: mistral-embed
    code_model: codestral-embed
    # Optional output dimensionality for Matryoshka/MRL models (8.1.6),
    # e.g. Gemini gemini-embedding-001 (native 3072, truncatable to
    # 1536/768). Omit to use the model's native dimension. Truncated
    # vectors are re-normalized by the adapter. Reindex-bound (8.1.4).
    # text_dim: 3072
    # code_dim: 3072
    # Optional multimodal embeddings (8.1.7): off (default) | augment |
    # replace. augment/replace require provider: gemini with BOTH
    # text_model AND code_model set to gemini-embedding-2 (all axes — a
    # mixed model is CONFIG_INVALID). Reindex-bound (8.1.4).
    # multimodal: off
  chat:
    provider: mistral
    model: mistral-small-2506
  ocr:
    provider: mistral-ocr
    model: mistral-ocr-latest

# Corpus source (§7.8). Default is a local filesystem path (the --dir root).
# rel_path is stable across schemes so a corpus may relocate local<->nfs<->s3
# without changing identity. The state dir always stays LOCAL.
source:
  kind: local            # local|nfs
  # kind: s3             # objects under a bucket+prefix
  # s3:
  #   bucket: my-corpus-bucket
  #   prefix: docs/
  #   region: us-east-1
  #   endpoint: ""       # optional, for S3-compatible stores
  #   # credentials resolve per §16.1.1 (env/keychain/file); never persisted.

# Vector index backend (§6). Default is the embedded, zero-infra Tier A.
# An external store (qdrant|pgvector, Tier C) is OPTIONAL and never required.
index:
  backend: memory        # memory (Tier A, default) | disk (Tier B) | qdrant | pgvector (Tier C)
  # qdrant:              # required only when backend=qdrant
  #   url: http://127.0.0.1:6333
  #   api_key: ${QDRANT_API_KEY}   # §16.1.1; never persisted
  # pgvector:            # required only when backend=pgvector
  #   dsn: ${PGVECTOR_DSN}         # §16.1.1; never persisted
  # An unreachable Tier C backend fails preflight with CONFIG_INVALID
  # (no silent fallback to an embedded tier, §6.3).

rag:
  generate_answer: true
  k_default: 15
  system_prompt: |
    You are a retrieval-augmented assistant.
    Use citations and never invent sources.
  max_context_chars: 20000
  oversample_factor: 5

ingest:
  gitignore: true
  extractor: auto      # auto|docling|docling-serve|mistral|pandoc|off
  # auto = best-available per format (§7.4.B.1): highest-fidelity ACTIVE engine
  # that supports each format; no format routed to an engine that can't read it,
  # no higher-fidelity engine bypassed. A pinned engine (docling|docling-serve|
  # mistral|pandoc) is honored exactly; formats it can't read degrade per
  # on_unsupported. pandoc (T2, #393) is capability-activated: a working binary
  # (resolved via PATH or pandoc.command) => its docx/odt/rtf/epub/html cells are
  # active; absent or non-functional => inactive.
  on_unsupported: lenient   # lenient|strict (§7.4.B.2). lenient (default) =
    # skip-with-warning: a doc left with NO searchable representation is recorded
    # as a durable status=skipped (unsupported-format skip_reason) + file_skip, so
    # the gap is named in the coverage report (§7.7) and survives a restart; strict
    # = non-fatal per-document UNSUPPORTED_FORMAT error (§7.7). Backward-compatible:
    # lenient keeps the not-indexed outcome, minus the silent part.
  docling:
    # HTTP endpoint of a running docling-serve container. REQUIRED when
    # extractor=docling-serve: an empty or unreachable URL disables that
    # extractor (no silent fallback to the docling CLI). Under extractor=auto
    # an empty value simply means the HTTP transport is not used.
    serve_url: ""      # e.g. http://127.0.0.1:5001
  pandoc:
    # Optional override for the pandoc binary (#393). Empty = resolve `pandoc`
    # from PATH. Capability-activated: a working binary activates the T2 matrix
    # cells; no enable flag. A resolved-but-non-functional binary is unavailable.
    command: ""        # e.g. /opt/homebrew/bin/pandoc
  pdf:
    mode: ocr          # off|ocr|auto
  images:
    mode: ocr_auto     # off|ocr_auto|ocr_on
  audio:
    mode: auto         # off|auto|on
    cache: true
  archives:
    mode: deep         # off|shallow|deep
  follow_symlinks: false
  max_file_mb: 20

chunking:
  max_chars: 2500
  overlap_chars: 250
  min_chars: 200
  code:
    max_lines: 200
    overlap_lines: 30
  transcript:
    segment_ms: 30000
    overlap_ms: 5000

stt:
  provider: mistral        # mistral|elevenlabs
  mistral:
    api_key: ${MISTRAL_API_KEY}
    model: voxtral-mini-latest
    timestamps: true
    # stt_languages: [ru, en]  # optional declared coverage (BCP-47); omit or [] => open/unknown (§8.2.1)
  elevenlabs:
    api_key: ${ELEVENLABS_API_KEY}
    model: scribe_v1
    timestamps: true

# Media transcription/translation/subtitle surface (§8.6; Status: Planned).
# Domain-general: no built-in language list, no default target language.
media:
  # language: ""              # optional pin; omit => auto-detect source language
  stt:                        # language-coverage-aware STT selection (§8.2.1)
    language_providers: {}    # NO default; map BCP-47 lang => STT provider profile name
                              #   (e.g. route a language the default model covers poorly
                              #    to one that covers it). Empty => single-provider behavior.
  translate:
    enabled: false            # opt-in; off by default (§8.6.2)
    target_langs: []          # NO default; enabling with [] is CONFIG_INVALID
    glossary: {}              # optional terminology guidance for the chat translator (§8.6.2);
                              #   keyed per target language: { <lang>: { "<source term>": "<rendering>" } }
                              #   guides the prompt (handles morphology); distinct from subtitles.glossary
  subtitles:
    formats: [vtt, srt]       # always available, derived from segment spans (§8.6.3)
    ttml:
      enabled: false          # TTML + SMIL optional, off by default; fail-open if codec metadata absent
      align_tolerance_ms: 2500 # bilingual cue cross-language alignment tolerance (§8.6.10)
  sidecars:
    enabled: true             # ingest .vtt/.srt/.ttml next to media as the transcript (§8.6.4)
  variants:
    group: true               # group multi-rendition by normalized name (§8.6.5)
    select: best              # transcribe canonical/best rendition once, deterministically
  quality_gate:               # degenerate-output checks before indexing (§8.6.6)
    min_chars_per_minute: 1   # low-density threshold (tune per corpus)
    max_repetition_ratio: 0.5 # repetition/looping threshold
  diarize:                    # speaker diarization (§8.6.8; Status: Planned)
    enabled: false            # off by default; requires a diarization-capable STT backend (§8.5)
    # tri-state: omit => auto-enable when the STT backend advertises the
    # capability; false => force off; true => require it (CONFIG_INVALID if absent)
  clip:                       # media clip citations (§15.11; dir2mcp_open_media_clip)
    max_duration_ms: 120000   # max clip span; longer requests => CLIP_TOO_LARGE
    max_bytes: 26214400       # 25 MiB inline byte cap; over => CLIP_TOO_LARGE
  batch:                      # large-archive ergonomics (§8.6.11; Status: Planned)
    two_phase: false          # opt-in: transcribe-all pass, then translate/export pass; output-equivalent to single-pass
    progress: false           # opt-in side-channel progress reporting (never affects output)
    manifest: ""              # path to a JSONL run manifest (per-asset status/duration/outputs); empty => disabled

rerank:
  # Reranking auto-activates when a provider credential is present
  # (cohere.api_key / COHERE_API_KEY). `enabled` is an optional
  # override: omit for auto, `false` to force off even with a
  # credential, `true` to require it (warns + fails open if absent).
  provider: cohere        # cohere
  candidate_pool: 50      # fused candidates re-scored before truncation to k
  cohere:
    api_key: ${COHERE_API_KEY}   # presence of this credential auto-enables reranking
    model: rerank-v3.5

x402:
  # `mode` is the primary, authoritative field controlling whether
  # payment gating is active.  Allowed values are `off`, `on`, and
  # `required` (see x402 spec).  The configuration loader normalizes
  # the mode string and writes it back into the struct during
  # validation.  After validation callers should rely solely on
  # `mode`.
  #
  # `enabled` is retained only for historical compatibility and
  # as a convenience for simple boolean cases; it is not consulted by
  # the loader.  When both fields are present, `mode` wins and `enabled`
  # is effectively derived (`enabled` == mode != "off").  Operators are
  # encouraged to specify `mode` exclusively and may treat `enabled` as
  # deprecated.  Future releases may drop `enabled` entirely.
  enabled: false               # deprecated; use `mode` instead (see note above)
  mode: off                    # off|on|required
  facilitator_url: ""
  resource_base_url: ""
  route_policy:
    tools_call:
      enabled: false
      price: "0.001"
      network: "eip155:8453"
      scheme: "exact"
      asset: ""
      pay_to: ""
  bazaar:
    enabled: false
    metadata:
      description: ""

server:
  listen: "127.0.0.1:0"
  mcp_path: "/mcp"
  protocol_version: "2025-11-25"
  tls:
    enabled: false
    cert_file: ""
    key_file: ""
  public: false

# session timeouts for MCP sessions
# default inactivity ~24h, adjust as needed
# session_max_lifetime zero disables absolute limit
session_inactivity_timeout: "24h"
session_max_lifetime: "0"

secrets:
  provider: auto         # auto|keychain|file|env|session
  keychain:
    service: "dir2mcp"
    account: "default"
  file:
    path: ".dir2mcp/secret.env"
    mode: "0600"

security:
  auth:
    mode: auto           # auto|none|file
    token_file: ""       # used when mode=file
    token_env: "DIR2MCP_AUTH_TOKEN"
  allowed_origins:
    - "http://localhost"
    - "http://127.0.0.1"
  path_excludes:
    - "**/.git/**"
    - "**/node_modules/**"
    - "**/.dir2mcp/**"
    - "**/.env"
    - "**/*.pem"
    - "**/*.key"
    - "**/id_rsa"
  secret_patterns:
    - 'AKIA[0-9A-Z]{16}'
    - '(?i)(?:aws(?:[_\s.]{0,20})?secret(?:[_\s.]*(?:access[_\s.]*)?key)?|secret[_\s.]*access[_\s.]*key)\s*[:=]\s*[0-9A-Za-z/+=]{20,}'
    - '(?i)(?:authorization\s*[:=]\s*bearer\s+|(?:access|id|refresh)_token\s*[:=]\s*)[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}'
    - '(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}'
    - 'sk_[a-z0-9]{32}|api_[A-Za-z0-9]{32}'
```

---

## 17) Security and safety requirements (minimum)

* Root isolation: reject any `rel_path` resolving outside root (`PATH_OUTSIDE_ROOT`).
* Symlink policy: default no-follow, or follow only if resolved under root.
* Archive safety: prevent zip-slip/path traversal within archives.
* Auth:
  * bearer token required by default
  * do **not** pass tokens on the command line; arguments may be exposed to other users or processes
  * use `--auth file:<path>` to point to a user-provided token file with restrictive permissions (the auto-generated `.dir2mcp/secret.token` is created with `0600`, but any secured path works)
  * alternatively, set `DIR2MCP_AUTH_TOKEN` for environment-based tokens
  * token file path and environment variable are equivalent sources; `--auth file:` tells dir2mcp where to read the token
  * config parity: `security.auth.token_file` specifies a file path and `security.auth.token_env` specifies an environment variable name; when either is set in `.dir2mcp.yaml` the behavior is equivalent (providing a token from the named source) but they refer to different source types.
* Secret handling:
  * secret input in interactive prompts MUST be masked
  * plaintext secrets MUST never be written to logs, terminal progress lines, NDJSON events, or config snapshots
  * preferred storage is OS keychain when available; file storage is fallback and MUST enforce `0600` permissions
  * if `secrets.provider=session`, secrets are process-memory only and are discarded at exit
* Origin checks:

  * if `Origin` header is present, enforce allowlist
* Sensitive file defaults:

  * default excludes include the secret regex patterns listed in §7.2, and
    these patterns are configurable via `security.secret_patterns`.
  * the exclusion engine is consulted on every file access, including
    `open_file`; tool handlers must reject or return an empty result for
    any path or content matching the configured patterns or path excludes.

---

## 18) Native x402 integration requirements (minimum)

* x402 mode is optional and must be switchable via config/flags (`off|on|required`).
* Payment enforcement MUST happen at the HTTP/MCP request boundary, not in retrieval/indexing internals.
* When a paid route is called without valid payment, server returns HTTP `402 Payment Required` with machine-readable payment requirements in `PAYMENT-REQUIRED` (standard x402 v2 `PaymentRequired`: first-class `resource` + `accepts[]` with `maxTimeoutSeconds`). The adapter MUST enforce x402 v2's replay/binding primitives (no new wire fields): the client's single-use `authorization.nonce` is consumed exactly once via a replay ledger, the `validAfter`/`validBefore` window and `maxTimeoutSeconds` are checked adapter-side, and the proof is matched against the entire selected `PaymentRequirements` and the challenge `resource` — so a proof valid for one resource/price MUST NOT be valid for another. Wire profile `X402Version: 2` (current latest). Enforcement detail: `docs/x402-payment-adapter-spec.md`.
* Paid retry requests MUST be validated from `PAYMENT-SIGNATURE` (x402 v2 semantics).
* For paid requests, verification and settlement MUST be delegated to a facilitator (hosted or self-managed); dir2mcp remains non-custodial. The adapter→facilitator transport MUST be `https` when credentialed or when the facilitator host is non-loopback (a bearer token MUST NOT traverse plaintext `http` to a non-loopback host), in all modes including `on`.
* dir2mcp remains non-custodial but MAY persist a bounded, non-custodial replay ledger (consumed nonces / idempotency keys). A payment nonce MUST be consumed exactly once on the `verified -> settled` transition; a replay of a consumed nonce — or the same nonce with a different request — MUST be rejected and MUST NOT drive a second execution or settlement. Replay detection keys off the payment nonce, not raw request bytes.
* Successful paid responses SHOULD include facilitator settlement metadata via `PAYMENT-RESPONSE` when available.
* x402 network identifiers MUST use CAIP-2 format (for example: `eip155:8453`, `eip155:84532`, `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d`).
* Recommended paid scope: gate `tools/call` (or selected tool names); keep lifecycle (`initialize`, `tools/list`) ungated.
* Payment failures MUST map to canonical tool/transport errors (`UNAUTHORIZED`, `MISTRAL_FAILED`, plus x402-specific payment failure metadata).
* If enabled, server should emit payment telemetry in NDJSON (`payment_required|payment_verified|payment_settled|payment_failed`).
* Bazaar/discovery metadata is optional and additive; lack of Bazaar metadata must not affect core MCP behavior.
* If Bazaar support is enabled, discovery metadata SHOULD be emitted via x402 extension metadata and resolved through facilitator discovery APIs (for example, `GET {facilitator_url}/discovery/resources`).

---

## 19) Non-goals (scope control)

* External vector stores (Qdrant, pgvector) are **OPTIONAL, never required**: the
  default is the embedded zero-infra Tier A and a conforming deployment MUST run
  with no external vector store (§1.2, §6). Requiring an external store is the
  non-goal — supporting one as an opt-in (Tier C, §6.2) is not.
* `sqlite-vec` is **rejected**: it is a C extension, incompatible with the pure-Go
  `modernc.org/sqlite` driver under `CGO_ENABLED=0` (§6.5). No embedded backend
  may require it.
* No in-place deletions in the **embedded** ANN index (use tombstones +
  oversampling, §6.6). A Tier C external store MAY delete natively, but MUST still
  honor the SQLite tombstone as the source of truth (§6.6).
* No marketplace inside dir2mcp.
* No requirement that audio output (TTS) be enabled for core retrieval/inspection workflows.
* No “agent that executes shell commands” (dir2mcp is retrieval/inspection only).

---

## 20) Implementation guidance (non-normative)

Suggested Go libraries for the interactive CLI experience:

* CLI parsing: `spf13/cobra`
* Prompt flow (wizard/select/masked input): `charmbracelet/huh`
* Output styling/layout: `charmbracelet/lipgloss`
* Optional progress spinner: `briandowns/spinner`
* TTY detection: `golang.org/x/term`
* OS keychain integration: `github.com/zalando/go-keyring`

These are recommendations, not protocol requirements.
