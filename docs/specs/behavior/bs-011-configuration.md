# bs-011: Configuration (single file)

- **ID:** bs-011
- **Version:** 0.3.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §16

## Scope

The single-file configuration model for dir2mcp: how settings are resolved
(precedence), how credential material is sourced (secret-source precedence), and
the full annotated minimal config template. This covers provider profiles,
per-capability model bindings, the corpus source, the vector index backend, RAG
parameters, ingestion, chunking, STT, the media surface, reranking, x402, the
server, session timeouts, secrets, and security.

A central invariant: **secrets are never persisted to the snapshot**. The config
snapshot (`.dir2mcp.yaml.snapshot`) records only secret *source* metadata, and
credentials are resolved at runtime per the secret-source precedence below.

## Specification (normative)

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
# `kind` selects the adapter/wire protocol; credentials follow the
# secret-source precedence (16.1.1) and are never persisted to the snapshot.
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
# credentialed profile that can serve the capability (td-001).
model:
  embed:                                   # reindex-bound (td-001)
    provider: mistral
    text_model: mistral-embed
    code_model: codestral-embed
    # Optional output dimensionality for Matryoshka/MRL models (td-001),
    # e.g. Gemini gemini-embedding-001 (native 3072, truncatable to
    # 1536/768). Omit to use the model's native dimension. Truncated
    # vectors are re-normalized by the adapter. Reindex-bound (td-001).
    # text_dim: 3072
    # code_dim: 3072
    # Optional multimodal embeddings (td-002): off (default) | augment |
    # replace. augment/replace require provider: gemini with BOTH
    # text_model AND code_model set to gemini-embedding-2 (all axes — a
    # mixed model is CONFIG_INVALID). Reindex-bound (td-001).
    # multimodal: off
  chat:
    provider: mistral
    model: mistral-small-2506
  ocr:
    provider: mistral-ocr
    model: mistral-ocr-latest

# Corpus source (bs-002). Default is a local filesystem path (the --dir root).
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
  #   # credentials resolve per the secret-source precedence (16.1.1,
  #   #   env/keychain/file); never persisted.

# Vector index backend (bs-008). Default is the embedded, zero-infra Tier A.
# An external store (qdrant|pgvector, Tier C) is OPTIONAL and never required.
index:
  backend: memory        # memory (Tier A, default) | disk (Tier B) | qdrant | pgvector (Tier C)
  # qdrant:              # required only when backend=qdrant
  #   url: http://127.0.0.1:6333
  #   api_key: ${QDRANT_API_KEY}   # secret-source precedence (16.1.1); never persisted
  # pgvector:            # required only when backend=pgvector
  #   dsn: ${PGVECTOR_DSN}         # secret-source precedence (16.1.1); never persisted
  # An unreachable Tier C backend fails preflight with CONFIG_INVALID
  # (no silent fallback to an embedded tier, bs-008).

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
  # auto = best-available per format (td-004 §B.1): highest-fidelity ACTIVE
  # engine that supports each format; no format routed to an engine that can't
  # read it, no higher-fidelity engine bypassed. A pinned engine (docling|
  # docling-serve|mistral) is honored exactly; formats it can't read degrade
  # per on_unsupported.
  on_unsupported: lenient   # lenient|strict (td-004 §B.2). lenient (default) =
    # skip-with-warning + name the gap in the coverage report (bs-002 §7.7);
    # strict = non-fatal per-document UNSUPPORTED_FORMAT error (bs-002 §7.7).
    # Backward-compatible: lenient preserves the current not-indexed outcome,
    # minus the silent part.
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
  # Late chunking (opt-in, off by default): embed the whole document through a
  # long-context model, then apply chunk boundaries and pool each chunk's token
  # vectors. Requires an embedder exposing token-level embeddings; one that
  # cannot falls back to chunk-then-embed. Part of the corpus-lifetime embed
  # identity (td-001 §8.1.4) — toggling it is reindex-bound.
  late_chunking: false

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

# Media transcription/translation/subtitle surface (td-003; Status: Planned).
# Domain-general: no built-in language list, no default target language.
media:
  # language: ""              # optional pin; omit => auto-detect source language
  translate:
    enabled: false            # opt-in; off by default (td-003)
    target_langs: []          # NO default; enabling with [] is CONFIG_INVALID
  subtitles:
    formats: [vtt, srt]       # always available, derived from segment spans (td-003)
    ttml:
      enabled: false          # TTML + SMIL optional, off by default; fail-open if codec metadata absent
      align_tolerance_ms: 2500 # bilingual cue cross-language alignment tolerance (td-003)
  sidecars:
    enabled: true             # ingest .vtt/.srt/.ttml next to media as the transcript (td-003)
  variants:
    group: true               # group multi-rendition by normalized name (td-003)
    select: best              # transcribe canonical/best rendition once, deterministically
  quality_gate:               # degenerate-output checks before indexing (td-003)
    min_chars_per_minute: 1   # low-density threshold (tune per corpus)
    max_repetition_ratio: 0.5 # repetition/looping threshold
  diarize:                    # speaker diarization (td-003; Status: Planned)
    enabled: false            # off by default; requires a diarization-capable STT backend (td-001)
    # tri-state: omit => auto-enable when the STT backend advertises the
    # capability; false => force off; true => require it (CONFIG_INVALID if absent)
  clip:                       # media clip citations (§15.11; dir2mcp_open_media_clip)
    max_duration_ms: 120000   # max clip span; longer requests => CLIP_TOO_LARGE
    max_bytes: 26214400       # 25 MiB inline byte cap; over => CLIP_TOO_LARGE
  batch:                      # large-archive ergonomics (td-003; Status: Planned)
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

> **Drift note:** the `media.clip` comment still cites `§15.11`
> (`dir2mcp_open_media_clip`), a section of the SPEC monolith that has no stable
> doc ID assigned in this restructure pass. Left as-is pending the media-clip
> tool's own numbered doc.

## Changelog

- **0.3.0** — ingest: added `ingest.late_chunking` (opt-in, off by default) to
  the §16.2 template. It is a component of the corpus-lifetime embed identity
  (td-001 §8.1.4; dir2mcp #332/#446), so toggling it is reindex-bound rather than
  a runtime knob.
- **0.2.0** — ingest: documented extractor=auto as best-available-per-format
  (td-004 §B.1) and added `ingest.on_unsupported: lenient|strict` (td-004 §B.2).
- **0.1.0** — Migrated from SPEC.md §16 (Configuration — single file), including
  §16.1 Precedence, §16.1.1 Secret source precedence, and §16.2 the full
  annotated minimal config template. The "secrets are never persisted to the
  snapshot" invariant and the secret-source precedence are preserved verbatim.
  Cross-references in the template comments rewired to stable doc IDs:
  §16.1.1 → secret-source precedence (this doc); §6/§6.3 → bs-008; §7.8 → bs-002;
  §8.1.3/§8.1.4/§8.1.6/§8.5 → td-001; §8.1.7 → td-002; §8.6 (and §8.6.2/.3/.4/.5/
  .6/.8/.10/.11) → td-003. `§15.11` (media clip) retained pending its own doc ID
  (see drift note).
