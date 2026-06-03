# SPEC.md
## dir2mcp Output & Integration Specification (Go)

**Spec version:** `0.14.0`  
**MCP protocol target:** `2025-11-25` (Streamable HTTP transport, sessions, tools, structured tool output)  
**Primary goal:** one-command “deploy-now” directory RAG exposed as an **MCP Streamable HTTP** server, with an embedded on-disk index (**no external DB; no Qdrant**) and a single config file.  
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

### 0.2 Implementation status notes (April 2026)

Status tags used in this spec:

- **Implemented:** available in current repository/runtime behavior.
- **Partially implemented:** interface exists, but not all target behavior is complete.
- **In progress:** work underway but not yet complete (may overlap with "partially implemented").
- **Planned:** target behavior not yet fully implemented.

Current high-level status:

- CLI + MCP server lifecycle, indexing pipeline, and core tool surface: **Implemented**
- Multimodal ingestion (OCR/transcription/annotation) and retrieval workflows: **Implemented** (with ongoing quality/perf hardening)
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
- No external vector database is required or supported.

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
  "event": "index_loaded|server_started|connection|scan_progress|embed_progress|file_error|payment_required|payment_verified|payment_settled|payment_failed|fatal",
  "data": {}
}
```

Required events for `up`:

* `index_loaded`
* `server_started`
* `connection` (endpoint + headers + token reference)
* periodic `scan_progress` and `embed_progress`
* `file_error` for per-document failures (non-fatal)
* if x402 is enabled: `payment_required`, `payment_verified`, `payment_settled`, `payment_failed`

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

Written on `up`:

```json
{
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
    "errors": 1
  }
}
```

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

### 5.2 `representations`

* `rep_id` (PK)
* `doc_id` (FK)
* `rep_type` (`raw_text|extracted_markdown|transcript|annotation_text|annotation_json`)
* `rep_hash` (stable; changes when rep changes)
* `created_unix`
* `meta_json` (must include provider/model for OCR/transcription/annotations when applicable)
* `deleted` (boolean; tombstone)

**Transcript meta_json requirements**

* `provider`: `mistral|elevenlabs`
* `model`: string
* `timestamps`: boolean
* `language`: optional
* `duration_ms`: optional

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
  * `embed_text_model`, `embed_text_dim`
  * `embed_code_model`, `embed_code_dim`
  * `ocr_model`
  * `stt_provider`, `stt_model`
  * `chat_model`

---

## 6) Embedded ANN indices

### 6.1 Indices

* `vectors_text.hnsw`: embeddings for `index_kind=text` chunks (raw text, OCR markdown, transcripts, annotation_text).
* `vectors_code.hnsw`: embeddings for `index_kind=code` chunks (source code and code-like configs).

Dimensions may differ between indices; each index must be internally consistent.

### 6.2 Label mapping

* ANN label MUST equal `chunk_id` (integer), so a query result maps directly to chunk metadata.

### 6.3 Deletions (append-only index approach)

Indices are treated as append-only:

* Deleting documents/representations/chunks sets `deleted=1` in SQLite.
* Retrieval uses oversampling:

  * ask ANN for `k * oversample_factor` results
  * filter out `deleted=1`
  * return first `k` remaining
* Default `oversample_factor`: 5 (configurable).

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

* Generate `raw_text` (normalized UTF-8, `\n` line endings).
* Route to index kind:

  * code → `index_kind=code`
  * others → `index_kind=text`

#### B) PDF/image/document

* Generate `extracted_markdown` via configured extractor (`ingest.extractor`):
  * `auto` (default): prefer docling, fallback to Mistral OCR
  * `docling`: require docling command/binary
  * `docling-serve`: require a reachable docling-serve HTTP endpoint (see below)
  * `mistral`: require Mistral OCR key/config
  * `off`: skip extracted representation
* Route to `index_kind=text`.
* Cache extracted output if enabled.

**Extractor transport.** The `docling` *engine* produces the same structured
document regardless of how it is reached; the `ingest.extractor` value selects
the transport explicitly: `docling` invokes a local CLI subprocess, while
`docling-serve` calls a docling-serve HTTP service at the endpoint addressed by
`ingest.docling.serve_url` (§16.2). Both transports MUST produce identical
output (the same `extracted_markdown` representation and `region` spans defined
below); the choice is operational and carries no wire- or schema-level
difference. Extraction is selected via `ingest.extractor` and is independent of
the model/provider bindings in §8 — it is not a provider capability.

Selecting `docling-serve` REQUIRES a non-empty, reachable `serve_url`. An empty
or unreachable endpoint makes the `docling-serve` extractor **unavailable** — a
disabled extractor for diagnostic purposes (§7.7), exactly as a missing docling
binary disables `docling` — and MUST NOT silently fall back to the CLI. (Under
`extractor: auto` the transport is implementation-determined: an empty
`serve_url` simply means the HTTP transport is not considered, and `auto` may
use the CLI or another configured extractor as usual.)

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

#### 8.1.3 Provider selection

For each capability, with `<cap>.provider`:

1. **Set** → use that profile, validated against 8.1.2. If it is required and the profile is not eligible (no credential present **and** not credential-less) → `CONFIG_INVALID` with remediation.
2. **Unset (auto)** → select the first profile, by a fixed deterministic precedence, that both (a) is **eligible** — a credential is present, or the profile is credential-less (e.g. a local endpoint) — and (b) can serve the capability. This generalizes the capability-driven activation rule already used by rerank (8.4) and STT (8.2).
3. **None qualify** → a *required* capability (`embed`) fails the startup preflight (§2.5); an *optional* one (`rerank`) stays off silently.

#### 8.1.4 Embeddings are a corpus-lifetime invariant

Vectors from different embed providers/models are not comparable. The embed **identity** — provider, per-axis model, **and the requested output dimension** (8.1.6, recorded as `embed_text_dim`/`embed_code_dim`, §5.5) — is bound to the index at first build and recorded in the config snapshot. On load, if the configured embed identity differs from the index's, the server MUST refuse to mix vector spaces — either erroring (`CONFIG_INVALID`) or triggering a full reindex. `embed.provider`/`embed.text_model`/`embed.code_model`/`embed.text_dim`/`embed.code_dim` — **and the multimodal mode (8.1.7)** — are therefore deploy-time, reindex-bound choices; `chat`/`ocr`/`stt`/`rerank` providers are runtime-swappable. The input role (8.1.5) is **not** part of this identity.

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
  * `time` (start_ms/end_ms)

### 9.3 Citation formatting (human-readable)

Within answers, citations must be rendered as:

* code/text: `[path:L<start>-L<end>]`
* pdf OCR: `[path#p=<page>]`
* pdf structured (region): render the primary page (`bbox.page`) as
  `[path#p=<page>]`; when the span covers multiple pages
  (`start_page != end_page`) render the range `[path#p=<start_page>-<end_page>]`.
  Optionally suffix with the section breadcrumb when present, e.g.
  `[report.pdf#p=3 › Results › 3.1 Revenue]`
* transcript: `[path@t=<start>-<end>]` where `<start>/<end>` are `mm:ss` or `ms`

### 9.4 RAG generation

If enabled:

* build a prompt with:

  * system prompt
  * question
  * retrieved contexts + citations
* return answer text + citations list + underlying hits (structured output)

If disabled or `mode=search_only`:

* return hits only.

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
* `OCR_FAILED`
* `TRANSCRIBE_FAILED`
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
      "properties": { "kind": { "const": "time" }, "start_ms": { "type": "integer" }, "end_ms": { "type": "integer" } },
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
    "doc_types": { "type": "array", "items": { "type": "string" } }
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
    "doc_types": { "type": "array", "items": { "type": "string" } }
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
        "stt_provider": { "type": "string", "enum": ["mistral", "elevenlabs"] },
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
    "stt_provider": { "type": "string", "enum": ["mistral", "elevenlabs"] },
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

**Description:** same as `ask` but includes audio output (TTS). Optional and additive. The input schema inherits all fields of `dir2mcp_ask` (`question`, `k`, `mode`, `index`, `path_prefix`, `file_glob`, `doc_types`) plus the audio-specific fields shown below.

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
  extractor: auto      # auto|docling|docling-serve|mistral|off
  docling:
    # HTTP endpoint of a running docling-serve container. REQUIRED when
    # extractor=docling-serve: an empty or unreachable URL disables that
    # extractor (no silent fallback to the docling CLI). Under extractor=auto
    # an empty value simply means the HTTP transport is not used.
    serve_url: ""      # e.g. http://127.0.0.1:5001
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
  elevenlabs:
    api_key: ${ELEVENLABS_API_KEY}
    model: scribe_v1
    timestamps: true

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
* When a paid route is called without valid payment, server returns HTTP `402 Payment Required` with machine-readable payment requirements in `PAYMENT-REQUIRED`.
* Paid retry requests MUST be validated from `PAYMENT-SIGNATURE` (x402 v2 semantics).
* For paid requests, verification and settlement MUST be delegated to a facilitator (hosted or self-managed); dir2mcp remains non-custodial.
* Successful paid responses SHOULD include facilitator settlement metadata via `PAYMENT-RESPONSE` when available.
* x402 network identifiers MUST use CAIP-2 format (for example: `eip155:8453`, `eip155:84532`, `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d`).
* Recommended paid scope: gate `tools/call` (or selected tool names); keep lifecycle (`initialize`, `tools/list`) ungated.
* Payment failures MUST map to canonical tool/transport errors (`UNAUTHORIZED`, `MISTRAL_FAILED`, plus x402-specific payment failure metadata).
* If enabled, server should emit payment telemetry in NDJSON (`payment_required|payment_verified|payment_settled|payment_failed`).
* Bazaar/discovery metadata is optional and additive; lack of Bazaar metadata must not affect core MCP behavior.
* If Bazaar support is enabled, discovery metadata SHOULD be emitted via x402 extension metadata and resolved through facilitator discovery APIs (for example, `GET {facilitator_url}/discovery/resources`).

---

## 19) Non-goals (scope control)

* No external vector DB backends (no Qdrant).
* No in-place deletions in ANN index (use tombstones + oversampling).
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
