# df-008: Error taxonomy (canonical codes)

- **ID:** df-008
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §14 (migrated verbatim; conformance note added — see Changelog)

## Scope

The canonical machine-readable error codes a conforming dir2mcp implementation
returns. A tool error is delivered as a normal tool result with `isError: true`
and a `structuredContent.error` object (see the result contract below), **not**
as a JSON-RPC protocol error.

## Specification (normative)

### Error result shape

Each tool error returns an object with:

| Field | Type | Meaning |
|-------|------|---------|
| `code` | string | One of the canonical codes below. Implementations **MUST** emit the canonical code, never an ad-hoc string. |
| `message` | string | Human-readable detail. **MUST NOT** contain secrets or raw sensitive payloads. |
| `retryable` | boolean | Whether retrying the same call may succeed (e.g. after indexing completes or a rate-limit window passes). |

### 14.1 Auth / transport

- `UNAUTHORIZED` — missing/invalid token.
- `FORBIDDEN_ORIGIN` — Origin not allowed.
- `SESSION_NOT_FOUND` — invalid `MCP-Session-Id`.
- `BIND_FAILED` — cannot bind host/port.
- `TLS_CONFIG_INVALID`.

### 14.2 Input validation

- `MISSING_FIELD`
- `INVALID_FIELD`
- `INVALID_RANGE`
- `CLIP_TOO_LARGE` — `dir2mcp_open_media_clip` when the requested time span
  exceeds the configured maximum clip duration/size bound. **Non-retryable** —
  the caller must request a shorter span.
- `FORBIDDEN` — path/content blocked by policy.
- `PATH_OUTSIDE_ROOT`
- `FILE_NOT_FOUND`
- `DOC_TYPE_UNSUPPORTED`
- `OCR_NOT_READY` — `dir2mcp_open_file` for a binary doc type (PDF, audio) when
  no OCR/transcript representation is cached yet. **Retryable** once ingestion
  completes.
- `MEDIA_NO_TEXT` — `dir2mcp_open_file` for a `replace`-mode multimodal media
  chunk (td-002) with **no** text representation. **Non-retryable** — the gap is
  permanent (unlike `OCR_NOT_READY`); the hit can still be cited.

### 14.3 Index / state

- `CONFIG_INVALID`
- `STORE_CORRUPT`
- `INDEX_VERSION_MISMATCH` — the on-disk index/schema version is incompatible
  with the running binary (see the `format_version` / `PRAGMA user_version`
  fence in [df-000](df-000-base.md)).
- `INDEX_NOT_READY` — should be rare; prefer partial results.

### 14.4 Ingestion / extraction

- `EXTRACT_FAILED`
- `OCR_FAILED` — also covers an OCR output **rejected by the degenerate-output
  quality gate** (td-003), not only a provider/transport failure.
- `TRANSCRIBE_FAILED` — also covers a transcript output rejected by the
  quality gate (empty / repetition / low density).
- `TRANSLATE_FAILED` — translation failed, including output rejected by the
  quality gate.
- `MEDIA_CLIP_FAILED` — clip extraction failed (`dir2mcp_open_media_clip`): the
  media is unreadable, the extraction tool (e.g. `ffmpeg`) is unavailable, or
  segment extraction errored. Distinct from `CLIP_TOO_LARGE` (a bounds rejection)
  and `MEDIA_NO_TEXT` (a missing-text condition on `open_file`).
- `ANNOTATE_FAILED`
- `FILE_TOO_LARGE`
- `BINARY_SKIPPED`

### 14.5 Provider / API

- `MISTRAL_AUTH`, `MISTRAL_RATE_LIMIT`, `MISTRAL_FAILED`
- `ELEVENLABS_AUTH`, `ELEVENLABS_RATE_LIMIT`, `ELEVENLABS_FAILED`

> Provider codes are namespaced per adapter. An adapter for another provider
> (OpenAI, Cohere, Gemini, omniembed) returns its own `<PROVIDER>_AUTH` /
> `_RATE_LIMIT` / `_FAILED` triad; the `retryable` flag distinguishes transient
> (rate-limit / network / 5xx) from permanent (auth / bad-request) failures.

## Conformance

Every canonical code above is normative: an implementation that hits one of
these conditions **MUST** emit the canonical code, not an ad-hoc Go error string.
*(The dir2mcp spec-conformance audit found several codes with no producer —
`OCR_FAILED`, `TRANSLATE_FAILED`, `FILE_TOO_LARGE`, `BINARY_SKIPPED`,
`INDEX_VERSION_MISMATCH`, `FORBIDDEN`, `BIND_FAILED`, `TLS_CONFIG_INVALID` —
those ingestion/state failures currently surface as ad-hoc strings or no machine
code. Wiring them to these canonical codes is the implementation side of this
document.)* The `retryable` boolean is part of the contract and MUST be set for
every error.

## Example

```json
{
  "isError": true,
  "content": [{ "type": "text", "text": "ocr payload too large" }],
  "structuredContent": {
    "error": { "code": "FILE_TOO_LARGE", "message": "document exceeds the OCR payload limit", "retryable": false }
  }
}
```

## Changelog

- **0.1.0** — Migrated from SPEC.md §14. Added the error-result-shape table and
  the conformance note (implementations MUST emit the canonical code; flags the
  currently-unimplemented codes). Updated cross-references (`§8.6.6` → td-003;
  `§8.1.7` → td-002; `INDEX_VERSION_MISMATCH` → df-000 version fence).
