# Versioning Policy

## Spec versioning

The spec uses [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

| Change type | Version bump |
|-------------|-------------|
| Breaking wire/schema behavior | Major |
| New optional fields, new optional tools | Minor |
| Clarifications, doc fixes | Patch |

**Pre-1.0 (beta) policy.** While the spec is `0.x` the project is pre-institutional and treated as **beta**: the `MAJOR` component stays `0`; **both** breaking wire/schema changes **and** new optional fields/tools bump the `MINOR` (e.g. `0.4.0 → 0.5.0`); only clarifications/doc-fixes bump the `PATCH`. (The SemVer table above describes post-`1.0` semantics — breaking → `MAJOR`, new optional → `MINOR` — and takes effect at `1.0.0`. The "Non-breaking additions" section below remains accurate: new optional surface is a `MINOR` bump in either regime.)

**Current spec version:** `0.22.0`
**MCP protocol target:** `2025-11-25`

## Implementation compatibility

Each implementation declares the spec version(s) it supports. `dirstral-cli` validates the supported spec version at runtime during `initialize`.

## Compatibility matrix

| Impl | Supported spec versions | Notes |
|------|------------------------|-------|
| `dir2mcp` (Go) | `0.21.x` (0.22 partial) | Reference implementation; audited against `internal/` 2026-06-30 (dirstral-spec#29). Implements the full **0.15–0.21** surface: extractor functional-check + graceful degradation (0.15); dual-machine corpus/index (`index.backend` memory/disk/qdrant/pgvector; remote `source.kind` local/nfs/s3 with stable `rel_path`) + the media transcription/translation/subtitle/quality-gate/derivation-identity surface (0.16); media-clip citations (`dir2mcp_open_media_clip` over `avutil.ExtractSegment`) + diarization speaker-aligned chunk boundary (0.17/0.19); CorpusFS abstraction + off-by-default distributed embedding (coordinator/workers/broker) (0.21). 0.18 retrieval-time dedup (`dedup.retrieval`) shipped; 0.18 §7.9 ingest-time canonicalization is itself spec-**Planned** and unbuilt. **0.22 per-language retrieval filter shipped end-to-end** (`languages[]` on `search`/`ask`, BCP-47 primary-subtag matching), but §8.8 **on-by-default language auto-detection is not yet wired** — language lands only via operator pin (`configured`) or sidecar (`declared`), not auto-`detected` (dir2mcp#471) — so the row is **`0.22 partial`**. Minor known gaps: distinct `TRANSLATE_FAILED`/`OCR_FAILED` codes (0.16 §14.4; dir2mcp#473) and the transcript-level `meta_json.words` granularity flag (0.20 §8.6.9; dir2mcp#472). The 0.13/0.14 multimodal-embedding arc (`gemini-embedding-2`, §8.1.7) is implemented but default-off and gated on the model's GA (Public Preview). The spec is authoritative — discrepancies are filed as spec-gap issues; maintainers decide whether to correct the spec or the implementation. |
| `dirstral-cli` | `0.4.x` | MUST update to `0.7.x` before releasing against spec `0.7.0`. No client code change for `0.6.0`/`0.7.0` (reranking and multi-provider selection are server-side; the wire/result contract is unchanged); the `0.5.0` tool-name rename remains the only wire-visible delta in this range. |
| `landfall` | TBD | |

## Contract freeze (issue #104)

As of spec version `0.4.0`, the following machine-readable artifacts have been added:

- `spec/tools/schemas/` — JSON Schema Draft-07 files for all 9 tools
- `spec/errors/taxonomy.md` — complete error code table including tool-execution errors
- `spec/sessions/lifecycle.md` — session expiry and `X-MCP-Session-Expired` header documented
- `spec/x402/extension.md` — `upto` scheme and `maxAmountRequired` field documented

Spec gaps identified during the review (see `<!-- spec-gap: ... -->` comments in each file):

- `SESSION_NOT_FOUND` JSON-RPC code was documented as `-32002`; implementation uses `-32001`
- `UNAUTHORIZED` JSON-RPC code was documented as `-32001`; implementation uses `-32000`
- Error `data` envelope (`{"code": ..., "retryable": ...}`) was not documented
- Tool execution errors return HTTP 200 with `isError: true`; this was not explicitly stated
- Several error codes (`MISSING_FIELD`, `INVALID_FIELD`, `INVALID_RANGE`, `STORE_CORRUPT`, `INTERNAL_ERROR`, `FORBIDDEN_ORIGIN`, `METHOD_NOT_FOUND`) were absent from the taxonomy

## 0.22.0 — detected-language metadata + per-language retrieval filter

Pins the contract for dir2mcp #267 item 4: **store detected language per
representation** to enable multilingual-corpus filtering and **per-language
retrieval**. `MINOR` bump per the pre-1.0 policy; fully **additive**, general-
purpose (auto-detect, no fixed language), and **off by default** — search/ask
return identical results when the new filter is omitted, and a corpus indexed
before any language was recorded simply carries unknown-language representations
that no specific filter matches (no migration, no breaking change).

**New / extended surface:**

- §5.2 **(extended) representation `meta_json` — detected-language metadata** —
  any representation (transcript, OCR `extracted_markdown`, plain `raw_text`) MAY
  record `language` (BCP-47, the effective indexed language), `language_source`
  (`detected` | `configured` | `declared`), and `language_confidence` (`[0,1]`,
  informational). All optional and additive; absent ⇒ unknown (never an error).
  Distinguishes the operator-pinned/expected language from the auto-detected one
  via `language_source`, consistent with the existing transcript/translation
  `language` / `source_language` fields (§8.6.2). A translated transcript's
  `language` is its **target** language; `source_language` is what it was
  translated from — both matchable.
- §8.8 **(new) Detected-language resolution (representation language)** —
  best-effort, on-by-default auto-detection with an optional operator pin; a
  deterministic resolution precedence (`configured` > `declared` > `detected`)
  recorded in `language_source`; graceful degradation (no signal ⇒ unknown, a
  first-class non-error state); an optional confidence floor at detection time;
  and detection stability/re-derivation rules (a pure detector change MAY refresh
  `language` without re-embedding, since language metadata never changes chunk
  `text`).
- §9.5 **(new) Per-language retrieval filter (optional)** — an OPTIONAL
  `languages` array (BCP-47 tags) on `dir2mcp_search` (§15.2) and `dir2mcp_ask`
  (§15.3). Matching is **case-insensitive on the BCP-47 primary subtag**, logical
  OR across the array. Absent/empty ⇒ no filtering (today's behavior, unchanged).
  An unknown/absent-language representation never matches a specific filter but is
  unaffected when no filter is given; an optional `und` sentinel MAY include
  unknown hits. Applied at candidate selection (with `path_prefix`/`file_glob`/
  `doc_types`), before dedup (§9.2), rerank (§9.1.1), and truncation; it only
  removes non-matching candidates and never reorders or changes result structure
  (§9.2) or citation format (§9.3). No match is an empty result, never an error;
  a syntactically invalid tag is `INVALID_FIELD` (§14).
- §15.2 / §15.3 + `spec/tools/schemas/search.json` & `ask.json` **(extended)** —
  add the optional `languages: string[]` input property to the `dir2mcp_search`
  and `dir2mcp_ask` schemas. Additive only; not `required`; existing callers that
  never send it observe no behavior change.

**No change to:** the persisted store shape beyond the optional §5.2 meta fields,
the result structure (§9.2), citation format (§9.3), the RAG contract (§9.4), any
output schema, span kinds, the error taxonomy (§14) beyond reusing the existing
`INVALID_FIELD`, or any other tool. No new tool or error code.

**Config:** the existing language pins (`media.language` / per-provider
`stt_language`, §16.2) are reused; an analogous pin for non-media text and an
optional detection confidence floor are implementation-defined and out of scope
for the normative template. No required new keys.

**Implementation note:** lands in a follow-up dir2mcp code PR once this spec
change is merged (gated submodule re-pin), unblocking dir2mcp #267 item 4
(detected-language metadata + per-language retrieval). **Status: Planned.**

## 0.21.0 — CorpusFS abstraction + distributed embedding (coordinator + workers)

Completes the distributed-ingest governance gate (dir2mcp #239) by adding the two
contracts the `0.16.0` dual-machine surface (§6, §7.8, §8.5) did not yet pin: the
**CorpusFS** corpus-filesystem abstraction (dir2mcp #242) and the **distributed
embedding** coordinator/worker job-queue contract (dir2mcp #248 distributed
workers, dir2mcp #249 standalone embed-worker mode). `MINOR` bump per the pre-1.0
policy; fully **additive** and **off by default** — the local-first single-binary
deployment (§1.2) runs the in-process embedding loop with no broker and no
external store, exactly as before.

**New surface:**

- §7.10 **(new) CorpusFS — corpus filesystem abstraction** — formalizes the
  backend-neutral **logical contract** that the §7.8 schemes (`local`/`nfs`/`s3`)
  implement, so discovery and media byte-reads work against any backing store
  without callers caring. Three capabilities: **list** (enumerate with `rel_path`
  + size + the backend's cheap change signal — `(size,mtime)` for fs, **ETag** for
  s3, §7.8), **stat** (single-`rel_path` metadata; a missing path is
  distinguishable from an error → drives the deletion/tombstone path), and
  **open / range-read** (random-access ranged reads, required for media windowing
  §8.1.7, per-page PDF, and `open_media_clip` §15.11). Invariants:
  backend-independent identity (`rel_path`/`content_hash` identical across schemes
  ⇒ `local⇄nfs⇄s3` relocation forces no reindex, §7.8), `PATH_OUTSIDE_ROOT`
  isolation on every capability (§17), state-stays-local + read-only-corpus
  (§1.2). No new config (the §16.2 `source:` block already selects the backing
  store). **Status: Planned.**
- §8.7 **(new) Distributed embedding (coordinator + workers)** — **optional,
  off-by-default** contract for embedding a corpus with multiple workers on
  separate machines (GPU pool) instead of the in-process loop; the single-binary
  default is the degenerate one-process case (§1.2). (8.7.1) **roles** —
  coordinator owns discovery/chunking/store/serve/retrieval + enqueues `pending`
  chunks (§5.3); stateless **embed-workers** pull jobs, read corpus bytes directly
  via CorpusFS (§7.10), embed via the configured provider (typically a co-located
  self-hosted endpoint §8.5), and write vectors + status back to the **shared**
  store. (8.7.2) **job description** — corpus ref (`corpus_id` + `source`), chunk
  identity (`chunk_id` + `index_kind`), payload identity (`text_hash`, or
  media `rel_path`+span for §8.1.7 media chunks), and the **embed identity**
  (§8.1.4) the job was enqueued under; workers read bytes directly from source
  (not relayed). (8.7.3) **idempotency/ordering** — at-least-once delivery,
  idempotent `chunk_id`-keyed writes (re-run = no-op, no duplicate vectors), no
  global ordering (partial-index retrieval, §1.2), per-job embed-identity
  enforcement (mismatch ⇒ fail, never mix spaces, §6.4), non-fatal failures with
  redelivery/dead-letter + lease expiry for crashed workers, tombstone safety
  (§6.6). (8.7.4) **shared store + broker** — a distributed pool REQUIRES a
  **Tier C** external store (§6.2/§6.3) reachable by all participants (the
  embedded Tier A/B are single-node); the broker is implementation-defined
  (NATS/Redis/SQS — any at-least-once + redelivery + dead-letter), credentials per
  §16.1.1 (never persisted); capability-driven + off by default; the standalone
  embed-worker run mode (dir2mcp #249) is the worker role without serving.
  **Status: Planned.**

**No change to:** the persisted store shape (§5), embed identity (§8.1.4),
retrieval/answer contract (§9), the MCP tool surface (§12–§15), the error taxonomy
(§14), or the §8.1.2 capability matrix. No new tool, error code, span kind, or
wire-contract change; the distributed mode changes **where** embedding runs, not
**what** is persisted. The existing `0.16.0` remote-source (§7.8), backend-tier
(§6), and self-hosted-endpoint (§8.5) contracts are referenced, not modified.

**Config:** no new keys. CorpusFS is selected by the existing §16.2 `source:`
block; the distributed mode reuses the existing Tier C `index:` connection (§16.2)
and adds only implementation-defined broker connection parameters (resolved per
§16.1.1, never persisted), out of scope for the normative template.

**Implementation note:** lands in follow-up dir2mcp code PRs once this spec change
is merged (gated submodule re-pin): CorpusFS local+S3 (#242), distributed workers
via job queue (#248), and the standalone embed-worker CLI mode (#249). These build
on the already-merged corpus-source config (#244) and self-hosted endpoints (#240)
under the same epic (#250).

## 0.20.0 — transcription word-level timing + bilingual subtitle export + batch ergonomics

Completes the §8.6 media surface for the `livevtt archive_transcriber` absorption
(dir2mcp #251) by adding the three downstream contracts that §8.6.1–§8.6.8 left
open: word-level timestamp normalization/surfacing (#252), bilingual TTML + SMIL
packaging (#255), and two-phase batch + progress/manifest ergonomics (#260).
`MINOR` bump per the pre-1.0 policy; **fully additive** and consistent with the
existing transcript-span (§8.6.1), diarization (§8.6.8), and media-chunk-window
behavior — every new surface is **optional and off/segment-level by default**, so a
conforming deployment behaves unchanged.

**New media subsections (all Status: Planned, domain-general — no language/
broadcaster specifics):**

- §8.6.9 **(new) Word-level timing: capture, normalization, and surfacing**
  (#252) — refines §8.6.1's per-word rule. Granularity is **recorded** via the
  `meta_json.words` flag, not assumed; absent ⇒ segment-only, graceful-degrade.
  Defines normalization of an OpenAI-compatible / faster-whisper `verbose_json`
  `words` array (`{word,start,end}` seconds) into the §8.6.1 `{t,d,w}` ms shape
  (deterministic round-half-up) as a **sibling** parser to the existing Mistral
  `[mm:ss]` segment path. Reaffirms `words` is metadata-only (no extra chunks, no
  text/boundary change ⇒ citation-stable). A `time` span MAY OPTIONALLY narrow to
  word boundaries **within** its segment without adding/dropping hits; citation
  syntax `[path@t=<start>-<end>]` is **unchanged** (bounds MAY be word-snapped).
- §8.6.10 **(new) Bilingual subtitle export (TTML + SMIL)** (#255) — refines
  §8.6.3's optional/off-by-default TTML/SMIL. Per-cue primary + secondary language
  text aligned to the **same** segment time region, each run `xml:lang`-tagged,
  both tracing back to the same transcript span. Deterministic cross-language cue
  alignment within `media.subtitles.ttml.align_tolerance_ms` (default `2500`);
  unmatched secondary cues are emitted, not dropped. SMIL packaging carries probed
  track metadata (codec/bitrate/width/height via `ffprobe`/`avutil`) and references
  the subtitle docs. **Fail-open** on missing metadata (omit SMIL, keep text
  subtitles). Missing-language export = `INVALID_FIELD`; bilingual requires
  translation (§8.6.2) or degrades to monolingual. Speaker voice markup per §8.6.8.
- §8.6.11 **(new) Two-phase batch transcription, progress, and run manifest**
  (#260) — opt-in two-pass ingest (transcribe-all, then translate/export) that is
  **observably output-equivalent** to single-pass (ordering/reporting only),
  independently resumable via existing identity/cache state (§7.6/§8.6.7). Optional
  **side-channel** progress that never affects output, monotonic against a
  pass-start total (cache hits count as completed). A **JSONL run manifest** (one
  record/asset) recording at least `rel_path` + `content_hash`, terminal `status`
  (+ canonical §14.4 error code), `duration_ms`, processing time, and outputs
  produced; **advisory for resume** only — live identity/cache/mtime gates win.
  Deterministic asset ordering. Worker-pool / multi-GPU distribution is out of scope.

**Supporting edits:**

- §8.6.1 — cross-references §8.6.9 for word normalization/surfacing.
- §8.6.3 — cross-references §8.6.10 for the bilingual TTML/SMIL contract.
- §9.3 **citation formatting** — transcript bounds MAY be word-snapped (§8.6.9);
  syntax unchanged.
- §16.2 **config template** — `media.subtitles.ttml.align_tolerance_ms` (default
  2500) and a new `media.batch:` block (`two_phase`/`progress`/`manifest`, all
  off/empty by default).
- Implementation note: lands in follow-up dir2mcp code PRs (#252/#255/#260) once
  this spec change is merged (gated submodule re-pin); the surfaces are **Planned**.

## 0.19.0 — diarization speaker-aligned chunk boundary

Refines the §8.6.8 speaker-diarization contract (dir2mcp #266) so it is
internally consistent and implementable. The original wording said diarization
changes neither chunk `text` nor segment boundaries, but the one-`speaker`-per-
span model (§5.4/§9.3) cannot hold when the char-budget chunker merges cues
across speaker turns. `MINOR` bump per the pre-1.0 policy (normative behavior
refinement on an opt-in, default-off path).

- §8.6.8 — clarifies that diarization is metadata-only for transcript **content**
  (never edits/reorders/re-times text) but MAY introduce a **chunk boundary at a
  speaker change** so every emitted chunk carries a single `speaker`. This
  speaker-aligned split is the **only** boundary effect permitted, applies **only
  when diarization is active**, and a transcript with no speaker attribution MUST
  chunk **identically** to the non-diarized path (default-off ⇒ unchanged).

## 0.18.0 — cross-file dedup & canonicalization

General corpus hygiene (dir2mcp #265): real corpora contain duplicates — the same
content at multiple paths, mirrored directories, byte-identical copies — and
indexing every copy bloats the index and returns the same content multiple times
per query. This release adds an **optional, off-by-default** cross-file
canonicalization surface. `MINOR` bump per the pre-1.0 policy; fully additive
(default-off ⇒ behavior unchanged).

**New surface:**

- §7.9 **Cross-file canonicalization (optional)** (new) — `dedup.exact: true`
  groups documents by identical `content_hash` (§7.6) into a **duplicate group**,
  selects one **canonical** document deterministically (`dedup.select: best|first`,
  sharing the media-variant selection vocabulary, §8.6.5), and generates
  representations/chunks/embeddings **only for the canonical**. Non-canonical
  members become **aliases** (discoverable + resolvable, but zero chunks/hits).
  Canonical removal **promotes** an alias. Near-duplicate (similarity) detection is
  explicitly **out of scope / future** and, if added, MUST stay opt-in.
- §9.2 **Cross-file de-duplication at retrieval** (addendum) — `dedup.retrieval:
  true` collapses candidate hits whose documents share a duplicate group to a
  single best-ranked survivor, **before** reranking and truncation to `k`. The
  candidate *pool* shrinks, not the rerank output, so the §9.1.1 **no-result-loss**
  guarantee is preserved (defined relative to the de-duplicated pool); a query MAY
  therefore return fewer than `k` hits when the corpus lacks `k` distinct results.
  Works whether or not ingest-time canonicalization is enabled.
- §5.1 `documents` (addendum) — optional `canonical_doc_id` / `is_alias` columns;
  alias rows share the canonical `content_hash` and hold no representations.
- §8.6.5 (addendum) — media variant selection is documented as the media-specific
  special case of §7.9, sharing the `best|first` selection vocabulary.

**Config (additive, default-off):** `dedup.exact` (bool), `dedup.select`
(`best|first`), `dedup.retrieval` (bool).

## 0.17.0 — media clip citations + speaker diarization

Two coordinated media-retrieval feature contracts land together to avoid a
SPEC.md collision: **media clip citations** (dir2mcp #264) and **speaker
diarization** (dir2mcp #266). Both build on the `0.16.0` media surface (§8.6) and
are **additive** — `MINOR` bump per the pre-1.0 policy (new optional tool, new
optional fields, new error codes; no breaking wire/schema change). Both ship
**Status: Planned**; implementation lands in follow-up dir2mcp code PRs (gated
submodule re-pin).

**(A) Media clip citations (#264):**

- §13.2 **Recommended extended tools** — adds `dir2mcp_open_media_clip`, the
  time-media analogue of `dir2mcp_open_file`: given a `chunk_id` (or
  `rel_path` + `start_ms`/`end_ms`), it extracts the **actual audio/video snippet**
  for the hit's `time` span (via the existing `avutil.ExtractSegment` seam) and
  returns it `inline` (base64) or by `reference` (short-lived `uri`).
- §15.11 **(new) `dir2mcp_open_media_clip`** — full input/output schema; selection
  rules (chunk-id resolution vs. explicit range); **normative bounds**
  (`media.clip.max_duration_ms` default 120000, `media.clip.max_bytes` default
  25 MiB); relationship to `open_file` (text vs. bytes for the same span);
  optional word-level deep-link refinement (§8.6.1 word spans); exclusion-engine
  and x402 gating inherited from `open_file`.
- §14.2 / §14.4 — new error codes `CLIP_TOO_LARGE` (bounds rejection,
  non-retryable) and `MEDIA_CLIP_FAILED` (extraction failure), distinct from the
  existing `MEDIA_NO_TEXT` on `open_file`.
- §16.2 — new `media.clip.{max_duration_ms,max_bytes}` config keys.
- New machine-readable contract `spec/tools/schemas/open_media_clip.json`;
  `spec/tools/schemas.md` index updated.

**(B) Speaker diarization (#266, Status: Planned):**

- §8.6.8 **(new) Speaker diarization** — **optional, off by default,
  provider-dependent** (requires a diarization-capable STT backend, §8.5). Speaker
  attribution is **metadata only** (does not change chunk `text` or segment
  boundaries). `media.diarize.enabled` is tri-state (omit ⇒ auto-enable when the
  backend advertises the capability; `false` ⇒ force off; `true` ⇒ require it,
  `CONFIG_INVALID` if absent). Speaker ids MUST be stable/deterministic across
  re-indexing; the diarization provider/model joins the transcript derivation
  identity (§8.6.7). Sidecar voice markup (WebVTT `<v>`) MAY populate speakers
  without a model derivation.
- §5.4 **spans** — `time`-span `extra_json` MAY carry `speaker` (stable id) and
  optional `speaker_label`; additive (consumers degrade to a flat citation).
- §5.2 **transcript meta_json** — adds `diarized`, `diarize_provider`,
  `diarize_model`, and a `speakers` set.
- §9.2 / §9.3 — a `time` hit `span` MAY surface `speaker`/`speaker_label`; human
  citations MAY append the speaker (e.g. `[interview.mp4@t=02:13-02:41 › S2]`).
- §15.2 **`dir2mcp_search`** — adds an optional `speaker` filter (restricts
  time-spanned transcript hits); `spec/tools/schemas/search.json` updated.
- §15.1.1 **`Span`** / `spec/tools/schemas/common.json` — the `time` variant gains
  optional `speaker`/`speaker_label` properties.
- §16.2 — new `media.diarize` config block.

No breaking wire/schema change; clients that ignore the new optional tool, fields,
and error codes interoperate unchanged.

## 0.16.0 — dual-machine corpus/index + media transcription surface

Two coordinated governance gates land together: the **dual-machine contract**
(remote corpus sources + pluggable vector-index backends, dir2mcp #239) and the
**media transcription/translation/subtitle surface** (absorbing the retired
`livevtt archive_transcriber`, dir2mcp #251). `MINOR` bump per the pre-1.0 policy.

> **Partly breaking — invariant relaxation.** This release **relaxes the
> long-standing "no external vector DB / no Qdrant" invariant** (§1.2, §19). The
> embedded, zero-infra index stays the **default** (Tier A), but an external
> vector store (Qdrant/pgvector, Tier C) MAY now be configured — it is
> **optional and never required**. A conforming deployment still runs with zero
> external infrastructure beyond model providers. Under the pre-1.0 policy a
> breaking change is still a `MINOR` bump.

**Dual-machine contract (#239):**

- §1.2 **Invariants** — the two vector-DB lines are replaced: default index is
  embedded/on-disk (no external service); an external store MAY be configured but
  MUST NOT be required; the state dir is always local even when the corpus root is
  remote.
- §6 **Vector index backends and identity** (rewritten from "Embedded ANN
  indices") — `index.backend` selector + tier table: `memory` (Tier A, in-memory
  pure-Go HNSW, **default**), `disk` (Tier B, pure-Go on-disk/memmapped,
  single-node), `qdrant`/`pgvector` (Tier C, external, optional-never-required).
  Two logical axes (text/code) with `chunk_id` as label/payload key. Embed
  identity (§8.1.4) binds **every** backend; Tier C is addressed by a
  collection/namespace derived from `corpus_id`; an unreachable Tier C at
  preflight fails `CONFIG_INVALID` (no silent fallback). New normative pure-Go /
  `CGO_ENABLED=0` subsection that explicitly **rejects `sqlite-vec`**. Tier C MAY
  delete natively while honoring the SQLite tombstone.
- §7.8 **(new) Remote corpus sources** — `source.kind` ∈ `local|nfs|s3`;
  filesystem walk for local/nfs, object enumeration for s3. `rel_path` is stable
  across schemes (s3 = object key minus prefix) so a corpus relocates
  local⇄nfs⇄s3 without changing identity; root-escape protections apply to every
  scheme. Change detection: (size, mtime)+content_hash for local/nfs; ETag (not
  MD5 for multipart/SSE-KMS) as the cheap signal for s3, with content_hash still
  reading the body. Missing object → tombstone. State dir stays local.
- §8.5 **(new) Self-hosted / OpenAI-compatible provider endpoints** — a
  self-hosted model server is first-class via the existing `kind: openai` contract
  (no new kind); MAY be credential-less and still auto-selectable. Capability
  mapping (embed/chat/stt via `/v1/embeddings`,`/v1/chat/completions`,
  `/v1/audio/transcriptions`; ocr has no OpenAI analog; STT validated at first
  use). Self-hosted embed bound by embed identity; STT normalization deferred to
  §8.6. No shipped self-hosted default.
- §19 **Non-goals** — external stores reframed as optional-never-required;
  `sqlite-vec` rejected (C extension); embedded no-in-place-delete retained, Tier
  C native delete allowed while honoring the tombstone.

**Media contract (#251, Status: Planned):**

- §8.6 **(new) Media transcription, translation, and subtitle surface** —
  domain-general, no language/broadcaster specifics. (8.6.1) transcript = TEXT
  with `time` spans, per-segment timestamps MUST when returned, per-word MAY in
  `spans.extra_json.words` ({t,d,w}) without extra chunks, no-timestamp fallback
  flagged `timing:"estimated"`, deterministic windowing. (8.6.2) source
  auto-detected (optional pin), translation opt-in/off by default, target langs
  configurable with no default (enabling with none = `CONFIG_INVALID`),
  per-language keying (`TranscriptLangSuffix`), translated transcripts record
  source_language + translate provider/model. (8.6.3) VTT/SRT always available
  (no re-transcription), TTML/SMIL optional/off (fail-open), missing language =
  `INVALID_FIELD`. (8.6.4) sidecar `.vtt/.srt/.ttml` ingested as transcript
  instead of STT, mtime-gated, `--force` overrides, multiple → per-language.
  (8.6.5) variant grouping, transcribe canonical/best once, no duplicate
  chunks/embeddings. (8.6.6) degenerate-output quality gates (empty/repetition/
  low-density; flag detected≠pinned) → non-fatal `TRANSCRIBE_FAILED`/`OCR_FAILED`/
  `TRANSLATE_FAILED`, deterministic. (8.6.7) representation provenance +
  derivation identity {capability,provider,model,version,language}; mismatch →
  re-derive+re-chunk+re-embed (per-representation analogue of embed-identity →
  reindex); sidecar transcripts are not model-derived.

**Supporting edits (both):**

- §5.2 **transcript meta_json** — provider enum no longer closed to
  `mistral|elevenlabs`; adds `model_version`, `timing`, `words`, `source`;
  translated transcripts add `source_language`/`translate_provider`/
  `translate_model`.
- §14.4 **error taxonomy** — new `TRANSLATE_FAILED`; `TRANSCRIBE_FAILED`/
  `OCR_FAILED` noted to also cover degenerate-output rejection. Mirrored in
  `spec/errors/taxonomy.md` (new Ingestion/extraction section).
- §16.2 **config template** — new `source:` block (`kind` local|nfs|s3 + s3
  bucket/prefix/region/endpoint, credentials per §16.1.1, never persisted), new
  `index:` block (`backend` memory|disk|qdrant|pgvector, default memory; Tier C
  connection optional), and new `media:` block (translate off + empty targets;
  subtitles vtt/srt + ttml off; sidecars on; variants group/best; quality_gate
  thresholds).
- §15.6 / §15.7 / `spec/tools/schemas/{stats,transcribe,transcribe_and_ask}.json`
  — `stt_provider` relaxed from the closed `["mistral","elevenlabs"]` enum to an
  open string (any STT-capable provider per §8.2/§8.5), matching the §5.2 change.
- Section-number map: §8.5 = self-hosted endpoints, §8.6 = media surface (both
  after §8.4 Rerank; resolves the dual-draft "§8.5" collision).
- Implementation note: both contracts land in follow-up dir2mcp code PRs once
  this spec change is merged (gated submodule re-pin). The media surface is
  **Planned** and ships phased.

## 0.15.0 — extractor availability (functional check)

Refines §7.4 so an extractor counts as *available* only when it can actually
run, not merely when it is configured. The `docling` CLI must both resolve and
pass a lightweight functional probe; a present-but-non-functional command (e.g.
a bundled virtualenv with ABI-incompatible dependencies) is **unavailable**,
exactly as an unreachable `serve_url` already makes `docling-serve` unavailable.
`MINOR` bump per the pre-1.0 policy (changes `auto` selection behavior; no new
tool, error code, config field, span kind, or wire-contract change). The §8.1.2
capability matrix is unchanged.

- §7.4 **(new) Extractor availability** — availability = resolves **and** passes
  a functional check; implementations SHOULD probe and MUST treat a
  present-but-broken extractor as unavailable, caching the result for the run.
- §7.4 **graceful degradation** — under `extractor: auto` an unavailable docling
  CLI is skipped and the cascade continues (docling-serve → Mistral OCR →
  disabled) instead of failing every document; under explicit `extractor:
  docling` an unavailable command disables extraction with no silent fallback,
  mirroring explicit `docling-serve`.
- §7.4 / §7.7 — the availability decision and its reason MUST be surfaced in
  startup diagnostics and `dir2mcp doctor`, so a broken extractor is visible
  rather than reported as healthy.
- No new tool, error code, config field, span kind, or wire-contract change.
- Implementation note: lands in a follow-up dir2mcp code PR (`doctor` functional
  probe + `auto` fallback). The dependency lock (homebrew-tap) and docling
  subprocess env isolation (#234) that make the bundled venv reproducible and
  immune to host-Python shadowing ship alongside.

## 0.14.0 — audio/video media windowing

Extends the multimodal-embedding surface (0.13.0, §8.1.7) with normative
**time-window chunking** for audio and video direct embedding under
`model.embed.multimodal` `augment`/`replace`, completing the modality set
after images and PDFs. `MINOR` bump per the pre-1.0 policy (refines optional
multimodal behavior; no new tool, error code, config field, span kind, or
wire-contract change — `time` spans already exist, §5.4). The §8.1.2
capability matrix is unchanged.

- §8.1.7 **(new) Media chunking (windowing)** — audio/video are split into
  non-overlapping, contiguous **time windows** (one media chunk each, `time`
  span) that MUST respect both the per-modality duration cap (audio ≤ 180 s,
  video ≤ 120 s) and the unified 8192-token budget; window boundaries are
  deterministic for stable citations. Image = one chunk; PDF = one chunk/page.
- §8.1.7 **fallback** — a file whose duration cannot be determined is a
  non-fatal per-document condition (§7.7): not directly embedded, warned;
  modalities with a text path (image/PDF OCR, audio transcript) keep that text
  representation even under `replace`, while video (no text path) is left
  unindexed.
- §8.1.7 **video** — has no default text representation (no video→text
  analogue to audio STT, §7.4.C): searchable only via media windows.
- §5.1 / §7.3 — `video` added to the `documents.doc_type` enumeration and the
  type-classification list (`.mp4`, `.mov`).
- No new tool, error code, config field, span kind, or wire-contract change.
- Implementation note: lands in a follow-up dir2mcp code PR (Phase 2c); the
  reference impl reads media duration via an `ffprobe` subprocess with a
  graceful skip when absent.

## 0.13.0 — multimodal embeddings (gemini-embedding-2)

Promotes [Design 0003](../docs/design/0003-multimodal-embeddings.md) to normative
spec: optional native **multimodal embeddings** that map text + images + audio +
video + PDFs into one shared vector space, via Google `gemini-embedding-2`,
behind a per-corpus `model.embed.multimodal` toggle. `MINOR` bump per the pre-1.0
policy (new optional config + one new tool-execution error code; no wire/tool
shape change). The §8.1.2 capability matrix is unchanged — multimodality is a
property of the chosen embed model, not a new capability cell. The model is
**Public Preview**: §8.1.7 carries a re-verify caveat and the implementation is
phased + GA-gated (compatibility row stays pending).

- §8.1.7 **(new) Multimodal embeddings** — `gemini-embedding-2`, modalities +
  per-request limits + the unified 8192-token window, the
  `model.embed.multimodal` (`off|augment|replace`) toggle, the single-shared-
  space constraint (`augment`/`replace` ⇒ `gemini-embedding-2` for all
  modalities, else `CONFIG_INVALID`), reindex-bound mode, provenance reusing
  existing span kinds, the page-image retrieval dedup rule, the `ask` grounding
  rule, and inspection via `MEDIA_NO_TEXT`.
- §8.1.4 **embed identity** — gains the multimodal mode.
- §14.2 **error taxonomy** — new non-retryable `MEDIA_NO_TEXT` (`open_file` on a
  `replace`-mode media chunk with no text representation); mirrored in
  `spec/errors/taxonomy.md`.
- §15.4 **`open_file`** — documents the `MEDIA_NO_TEXT` outcome.
- §16.2 **config template** — `model.embed.multimodal` (default `off`).
- No new tool, span kind, or wire-contract change.
- Implementation note: lands in phased follow-up dir2mcp code PRs (adapter →
  ingestion → store → retrieval → `ask` → `replace`), starting with the
  default-off adapter slice; preview limits/endpoints re-verified at GA.

## 0.12.0 — native Gemini STT/TTS

Pins the native wire mechanism for the already-`✅` Gemini STT and TTS
capability cells (8.1.2). Gemini's OpenAI-compatible layer does not expose
`/v1/audio/*`, so audio must use the native `models/{model}:generateContent`
surface — this release makes that normative and defines the TTS audio
container. `MINOR` bump per the pre-1.0 policy (provider-behavior change on
already-`✅` matrix cells; no new tool, error code, config field, or
wire-contract change — STT/TTS knobs `stt_model`/`stt_language`/`tts_model`/
`tts_voice` already exist since 0.7.0).

- §8.1.1 **provider profiles**: `gemini` STT/TTS clarified as native
  (`generateContent`); the `kind: openai` Gemini path serves chat only.
- §8.2 **STT**: Gemini transcribes via `generateContent` with the audio as
  an inline-data part; output normalized to the `transcript` representation
  like every other provider. `stt_model` default `gemini-2.5-flash`.
- §8.3 **TTS**: Gemini synthesizes via `generateContent` with
  `generationConfig.responseModalities: ["AUDIO"]` + a `speechConfig` voice (`tts_voice`,
  default `Kore`; `tts_model` default `gemini-2.5-flash-preview-tts`). The
  returned raw PCM (s16le, 24 kHz, mono) MUST be wrapped in a WAV container
  so the bytes are directly playable, matching ElevenLabs/OpenAI. TTS stays
  fail-open (8.3).
- No new tool, error code, config field, span kind, or wire-contract
  change; the §8.1.2 matrix is unchanged (`gemini` STT/TTS were already `✅`).
- Implementation note: the native Gemini STT/TTS backend lands in a
  follow-up dir2mcp code PR once this spec change is merged (replacing the
  current OpenAI-compat `/audio/*` shim, which Gemini does not serve).

## 0.11.0 — native Gemini embedding parity (taskType + Matryoshka)

Promotes the `gemini` embed adapter from the OpenAI-compatible shim to Gemini's
**native** embed surface so it reaches feature parity with `gemini-embedding-001`:
asymmetric `taskType` (document/query, with a code-aware refinement) and
configurable Matryoshka output dimensionality. `MINOR` bump per the pre-1.0
policy (new optional config fields + provider-behavior change on an already-`✅`
matrix cell; no new tool, error code, or wire-contract change). The §8.1.2
matrix is unchanged (`gemini` embed was already `✅`).

- §8.1.1 **provider profiles**: `gemini` embed clarified as native, **asymmetric**
  via `taskType`, with Matryoshka output dimensionality; the OpenAI-compatible
  alternative forgoes `taskType` and stays symmetric.
- §8.1.4 **embed identity**: the requested output dimension joins provider+model
  in the corpus-lifetime embed identity (recorded as `embed_text_dim`/
  `embed_code_dim`, §5.5); changing it forces a reindex / `CONFIG_INVALID`.
- §8.1.5 **asymmetric embeddings**: `gemini` added alongside Cohere/Voyage. Role
  mapping: `document`→`RETRIEVAL_DOCUMENT`, `query`→`RETRIEVAL_QUERY`; for the
  configured **code** model a `query` maps to `CODE_RETRIEVAL_QUERY`.
- §8.1.6 **configurable embedding dimensionality (Matryoshka/MRL)** (new):
  optional `model.embed.text_dim`/`code_dim`; adapters request `outputDimensionality`
  where supported and **re-normalize** truncated vectors; unsupported dimensions
  are `CONFIG_INVALID`, never silently ignored.
- §16.2 **config template**: `model.embed` gains optional `text_dim`/`code_dim`
  (commented; native dimension when omitted). No provider/matrix change.
- No new tool, error code, span kind, or wire-contract change. STT/TTS remain
  `✅` in the matrix; native Gemini STT/TTS implementation is a separate slice.
- Implementation note: the native Gemini embedding backend (and, separately,
  native Gemini STT/TTS) land in follow-up dir2mcp code PRs once this spec
  change is merged.

## 0.10.0 — docling-serve HTTP extraction transport (docling-serve)

Adds `docling-serve` as an `ingest.extractor` value: an alternative *transport*
for docling extraction that talks to a local docling-serve HTTP container
instead of the docling CLI subprocess. Extraction remains its own selection axis
(`ingest.extractor`), independent of the §8 model/provider bindings — it is
deliberately **not** modeled as a provider capability (the §8.1.2 matrix is
unchanged). `MINOR` bump per the pre-1.0 policy (new optional `ingest.extractor`
value + one new optional config field, non-breaking). Output is byte-identical
to the docling CLI path (same `extracted_markdown` + `region` spans from 0.9.0).

- §7.4.B **representation**: `ingest.extractor` gains `docling-serve`. The `ingest.extractor` value selects the transport explicitly (`docling` = CLI subprocess, `docling-serve` = HTTP service); both transports MUST produce identical output, and extraction is independent of the §8 provider bindings. Selecting `docling-serve` requires a non-empty, reachable `serve_url`; an empty or unreachable endpoint disables that extractor for diagnostics (§7.7), like a missing docling binary, and MUST NOT silently fall back to the CLI. (Under `extractor: auto` the transport is implementation-determined.)
- §16.2 **config template**: add `ingest.docling.serve_url` (empty by default; required when `extractor=docling-serve`, otherwise the HTTP transport is simply not used). One new optional config field; no provider/matrix change.
- No new tool, error code, span kind, provider kind, or wire-contract change. The MCP tool surface and persisted store shape are unchanged.
- Implementation note: the dir2mcp docling-serve backend lands in a follow-up code PR once this spec change is merged (HTTP extractor reusing the existing DoclingDocument parser; doctor probes the endpoint's `/ready`). Container lifecycle is user-managed (dir2mcp probes and fails fast; it does not start/stop the container).

## 0.9.0 — structured docling extraction (region provenance)

Formalizes structured `DoclingDocument` ingestion as the docling extraction contract (previously flat Markdown) and adds region-level provenance for precise citations. `MINOR` bump per the pre-1.0 policy (new optional span kind + new optional citation fields, non-breaking — clients that ignore the new fields continue to work). Design: [docs/design/0002-structured-extraction.md](../docs/design/0002-structured-extraction.md).

- §5.4 **spans**: new `region` span kind — page range in `start`/`end`, with `bbox` (primary-page bounding box), `section` breadcrumb, and element `label` carried in the existing `extra_json` column. **No schema migration** (reuses `extra_json`).
- §7.4.B **representation**: structured extraction preserves reading order, section hierarchy, per-element page/bbox provenance, atomic tables, and figure captions/classifications; title from the model's title element. The persisted `extracted_markdown` representation remains rendered Markdown (structure lands in spans); raw `DoclingDocument` JSON is an implementation-private cache, not a representation. Page-separated OCR fallback (Mistral) is unchanged.
- §7.5 **chunking**: section/element-aware chunking for structured documents (group by section breadcrumb, keep tables atomic).
- §15.1.1 **`Span`** (client-facing) + `spec/tools/schemas/common.json`: additive `region` variant (`start_page`/`end_page`, required `bbox`, optional `section`). A `region` span always carries a `bbox` — an element without provenance is recorded as a `page` span instead. The machine-readable `common.json` `Span` (previously the drifted minimal `{start,end}` shape) is brought in line with the kind-tagged `§15.1.1` union (`lines|page|time|region|document`) so the authoritative JSON schema matches the prose. Backward compatible — existing kinds unchanged; clients MUST ignore unrecognized kinds/fields.
- §9.2 **result objects** / §9.3 **citation rendering**: `region` added to the hit span-kind list; region citations render the primary page (`bbox.page`), or a page range when `start_page != end_page`, optionally suffixed with the section breadcrumb.
- No new tool, no new error code, no config-shape change. `spec/errors/taxonomy.md` and `spec/sessions/lifecycle.md` are unaffected (header version bump only). A new `dirstral-conformance` test for the `region` span variant is recommended-not-required.
- Implementation note: the structured pipeline lands in a follow-up dir2mcp PR (extractor → `--to json`, store `extra_json` read/write, section-aware chunking, retrieval + MCP citation surface) once this spec change is merged.

## 0.8.0 — stats.recent_failures (per-document failure visibility)

Extends `dir2mcp_stats` output with an **optional** `recent_failures` array surfacing the most-recent documents with `status='error'` along with a short, sanitized `error_message`. `MINOR` bump per the pre-1.0 policy (new optional field on an existing tool, non-breaking — clients that ignore the field continue to work).

Motivation: a maintainer triaging a failed corpus today can see *that* documents failed (existing `indexing.errors` counter) but not *which* documents or *why*. The information already exists in the implementation's metadata store (per-document `error_message`) and ships in `dir2mcp support-bundle`'s `list-files.json`; this spec change makes it available programmatically through the spec-blessed diagnostic surface (`stats`) so doctor-style dashboards and remote diagnostics can render it without scraping the bundle.

- §15.6 `dir2mcp_stats` output: optional `recent_failures` array (`additionalProperties: false`), each item `{rel_path, doc_type, mtime_unix, error_message}` (all required when an item is present), newest-first by `mtime_unix`. Implementations SHOULD cap at 20 entries by default and SHOULD cap `error_message` at 512 bytes on a UTF-8 rune boundary with control characters stripped (one-line render). Implementations MAY omit the field entirely when no failures are recorded; clients MUST treat omission as "no recent failures" (not "unsupported") per the existing "Clients MUST ignore unknown fields" rule. `error_message` is normative as a **diagnostic** signal — it MUST NOT contain secrets, raw file content, or unsanitized provider response bodies.
- `spec/tools/schemas/stats.json`: mirrored.
- No new tool, no new error code, no config-shape change. `spec/errors/taxonomy.md`, the rest of `spec/tools/schemas/*`, and the `dirstral-conformance` suites are unaffected (a new conformance test for the optional field is recommended-not-required).
- Implementation note: dir2mcp reference impl 0.5.8+ persists per-document `documents.error_message` (additive SQLite migration; introduced in dir2mcp #212). The stats wiring lands in a follow-up dir2mcp PR once this spec change is merged.

## 0.7.0 — multi-provider model abstraction

Generalizes the model pipeline from Mistral-centric to **provider-agnostic**: every capability (embed/chat/ocr/stt/rerank) binds to a configurable provider profile. A `MINOR` bump per the pre-1.0 policy — it is both a config-shape break (the monolithic `mistral:` block is removed) and new optional surface; a clean break is acceptable (no compatibility users). Design: [docs/design/0001-multi-provider.md](../docs/design/0001-multi-provider.md).

- §1 **Implementation goal** rewritten provider-agnostic; Mistral is the default profile, not privileged.
- §8.1 **Provider model**: profiles (`kind` = `openai`/`mistral`/`anthropic`/`gemini`/`cohere`/`elevenlabs`), the OpenAI-compatible backbone covering OpenAI/OpenRouter/Groq/Azure/local **and Mistral chat+embed**, bespoke adapters only for non-OpenAI surfaces (Mistral `/v1/ocr`, Anthropic, Cohere rerank, ElevenLabs).
- §8.1.2 **Capability matrix** (normative): binding a capability to an incapable `kind` is `CONFIG_INVALID`.
- §8.1.3 **Provider selection**: explicit `<cap>.provider`, else capability-driven auto-pick by precedence among credentialed+capable profiles (generalizes the rerank/STT rule).
- §8.1.4 **Embeddings corpus-lifetime invariant**: embed identity is bound to the index; mismatched reload MUST error or reindex (no silent vector-space mixing).
- §8.1.5 **Asymmetric embeddings (input role)**: every embedding call carries a document/query input role; asymmetric providers (Cohere `input_type`, Voyage) MUST honor it, symmetric providers ignore it. The reference `Embedder` interface gains the role parameter (clean internal pre-1.0 break).
- **Full Cohere**: `kind: cohere` serves embed + chat (`/v2/chat`) + rerank in 0.7.0 (not rerank-only).
- **Provider-agnostic STT/TTS**: §8.2/§8.3 generalized — STT/TTS are selected per §8.1.3 among capable profiles (Mistral/ElevenLabs/OpenAI/Gemini for STT; ElevenLabs/OpenAI/Gemini for TTS). `kind: openai` audio is endpoint-dependent (validated at first use, never `CONFIG_INVALID`); every other matrix `✅` is statically valid. No provider is left half-wired.
- §16.2 config template: monolithic `mistral:` replaced by `providers:` map + `model:` capability bindings; `stt:`/`rerank:` shapes retained.
- §2.5 startup preflight generalized from "requires Mistral API key" to per-capability provider credentials.
- **No new tool, tool-schema field, or error code** (one new config-validation case reuses `CONFIG_INVALID`). `spec/tools/schemas/*` and `spec/errors/taxonomy.md` unchanged; `dirstral-conformance` unaffected.

## 0.6.0 — optional reranking (Cohere)

New **optional** retrieval-quality stage; capability-driven (auto-activates only when a rerank provider credential is present, off otherwise), non-breaking — `MINOR` bump per the pre-1.0 policy (new optional surface → `MINOR`).

- §8.4 **Rerank providers (optional)**: Cohere (`POST /v2/rerank`, default `rerank-v3.5`); capability-driven activation (auto-on when a credential is present, mirroring embedding/OCR provider gating); `rerank.enabled` is a tri-state override (unset → auto, `false` → force off, `true` → require + warn/fail-open if absent); fail-open; key not persisted.
- §9.1.1 **Optional reranking**: post-fusion re-scoring of the top `rerank.candidate_pool` (default 50) candidates before truncation to `k`; reorder-only (result structure §9.2 unchanged); `index=both` reranks once on the merged pool; deterministic tie-break by `chunk_id`.
- §16.2 config template: `rerank:` block (mirrors the `stt:` provider-selector shape).
- No new tool, tool-schema field, or error code (fail-open surfaces no new tool error). `spec/tools/schemas/*` and `spec/errors/taxonomy.md` unchanged.

## 0.5.0 — reconcile shipped dir2mcp (spec-gap resolution)

Protocol-council decision: the dir2mcp reference implementation had shipped behavior that diverged from canonical `0.4.0`. Per the pre-1.0 beta policy and the "spec is authoritative; maintainers decide spec-vs-impl direction" rule, all of the following were resolved **impl → spec** (the spec now ratifies shipped behavior); breaking deltas bump `MINOR` (`0.4.0 → 0.5.0`):

- **Tool naming** `dir2mcp.<tool>` → `dir2mcp_<tool>` (breaking; ratifies dir2mcp #172). The former dotted-namespace rule is **superseded** — underscore form is canonical across `docs/SPEC.md`, `spec/tools/schemas.md`, and every `spec/tools/schemas/*.json` title.
- **`rep_type` enum** `ocr_markdown` → `extracted_markdown` (breaking; ratifies dir2mcp #152 docling extractor abstraction).
- **`k` default** `10` → `15` for `search`/`ask`/`ask_audio`/`transcribe_and_ask` (ratifies dir2mcp #163).
- **`OCR_NOT_READY`** tool-execution error added + `open_file` binary-doc semantics + `span.kind="document"` variant (new optional; ratifies dir2mcp #180).
- **`serverInfo.name`** per-instance auto-derivation + `dir2mcp-dev-` prefix for dev builds (new optional; ratifies dir2mcp #184/#185).
- **x402 adapter**: facilitator defaults to the Coinbase x402 Go SDK client (clarification).

`dirstral-conformance` SHOULD extend suites for the renamed tool surface before any impl releases against `0.5.0`.

## Breaking change process

1. Open a spec PR with the proposed change
2. Maintainer review required (protocol council gate)
3. Bump the version in `spec/versioning.md` (while `0.x`: breaking → `MINOR` per the pre-1.0 policy; post-`1.0`: `MAJOR`)
4. All implementation repos must update their compatibility matrix before releasing against the new spec version
5. `dirstral-conformance` must add a new test suite for the new behavior

## Non-breaking additions

New optional tools or optional fields in existing tool schemas may be added in a minor version without breaking existing clients. Clients MUST ignore unknown fields.
