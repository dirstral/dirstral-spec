# Design 0004 — Recognition capability (`recognize`)

**Status:** Proposed (targets a future MINOR spec bump; supersedes this
note's earlier annotation-sidecar draft)
**Author:** dirstral maintainers
**Extends:** SPEC §8 (capabilities & provider bindings, [Design 0001](0001-multi-provider.md)), §8.6.7 (derivation identity), §5.2 (representation sources), §5.4 (spans)
**Related:** [Design 0002](0002-structured-extraction.md) (entity-aware querying, future), [Design 0003](0003-multimodal-embeddings.md) (fuzzy visual recall; media-fetch open question)

## 1. Summary

Add **`recognize`** as an optional per-corpus **capability binding**: during
ingest, dir2mcp runs a recognition backend over media files and persists the
result as a **derived annotation representation** — human-readable,
time-ranged statements about content ("Pitch: Logan Webb to Freddie Freeman,
00:42:10–00:42:31"), indexed like any text and cited with `time` spans.

This is the same architectural class as `ocr` and `stt`: a capability slot
in the Design-0001 provider model, producing a model-derived representation
with a derivation identity, re-derived when the backend changes. The
motivating pilot is player/event recognition over sports footage, but the
capability is domain-neutral: speaker diarization labels, scene/logo/object
recognition — anything that maps media time ranges to statements.

An earlier draft of this note proposed publishing recognizer output as
*annotation sidecar files* to keep vision outside the core. That approach is
**superseded**: it misused authored-sidecar semantics (never invalidated, no
derivation identity, §8.6.7) for machine-derived content, and it was
inconsistent with the codebase's own precedent — dir2mcp already runs
OCR/STT providers and manages a locally-served extraction tool (docling).
Recognition belongs in the same pattern.

## 2. Motivation

- **Consistency.** OCR, STT, and docling-based extraction are in-core
  capabilities that call models and persist derived representations.
  Recognition is the same shape; treating it differently forced a second,
  weaker contract (files-next-to-media) alongside the real one.
- **Correct derivation semantics for free.** As a derived representation,
  recognition output carries a derivation identity and is re-derived when
  the backend changes — exactly like an STT transcript, with no new
  supersession rules to invent.
- **Operational simplicity.** `dir2mcp up` with `recognize` configured
  indexes footage into queryable moments in one step: no separate
  publishing pipeline, and watch-mode/reindex semantics apply unchanged.
- **Retrieval needs nothing new.** Statements are text chunks with `time`
  spans; `search`/`ask` and the citation contract are untouched.

## 3. Configuration

Per the Design-0001 per-capability provider-selector pattern (as for
`stt.provider`):

```text
recognize.provider = off | serve      # default: off
recognize.base_url = http://127.0.0.1:<port>   # required for `serve`
recognize.serve_command = ...         # optional: dir2mcp launches the backend
```

- **`off`** (default): no recognition; zero change for existing corpora.
- **`serve`**: a locally served recognizer process. The reference backend is
  `dirstral-annotator serve` (lives in the dir2mcp repo under `annotator/`),
  which cascades play-by-play, scorebug OCR, jersey OCR, and face
  recognition and fuses them into confidence-scored statements. Two
  lifecycle modes:
  - **managed** — `recognize.serve_command` set: the daemon launches the
    command itself (own process group), waits for `GET /health` (bounded),
    and terminates the tree on shutdown. `dir2mcp up` is the only process
    the operator runs; a backend that exits early or never turns healthy
    fails startup loudly.
  - **connect-only** — no command: the daemon connects to an
    operator-started backend, probing `/health` once at startup (warning
    when unreachable; per-document ingest errors remain the hard signal).
- **Future providers:** hosted recognition APIs (e.g. face-collection
  services, video-capable multimodal chat models) slot in as additional
  provider values without contract changes — the capability, not the
  backend, is what this design fixes.

Domain configuration (rosters, image banks, event vocabularies) **and
confidence thresholds** belong to the **backend**, not to dir2mcp: the core
stays domain-neutral, passes media paths, and indexes what the backend
returns; the reference backend's fusion floor (`--min-confidence`) is where
low-confidence annotations are dropped.

Validation follows the strict-config precedent: `serve` without a usable
`base_url` is `CONFIG_INVALID` at startup.

## 4. Ingestion & representation

For each media document of a recognized type (v1: `video`; images/audio are
a follow-up), after transcript handling:

- dir2mcp calls the backend (§5) and receives annotations.
- It persists **one representation**: `rep_type: annotation`, meta_json
  `source: recognize` (new §5.2 value), plus the **derivation identity**:
  provider, backend name, backend version (from the response, §5). Per
  §8.6.7 semantics this representation IS invalidated and re-derived when
  the identity changes, and a forced reindex retires stale rows — the exact
  STT rules, applied to a new rep_type.
- **Chunks:** one chunk per annotation; chunk text is the statement
  (`text` field, with entity labels inline so plain text search finds
  players by name); each chunk carries exactly one `time` span. No new
  persisted span kind (§5.4). The response's `start_s`/`end_s` (seconds,
  floats) are converted to the integer-millisecond `time` span by rounding
  **outward** — `start_ms = floor(start_s * 1000)`, `end_ms = ceil(end_s *
  1000)` — so the persisted span never excludes a boundary the recognizer
  included, and an instantaneous annotation (`end_s == start_s`) still yields
  a non-empty `[start_ms, end_ms]` after rounding when the value is not an
  exact millisecond. A reversed span (`end_s < start_s`) is rejected as a
  malformed record before persistence (§5).
- Backend failure handling mirrors STT: per-document error recording, no
  partial silent success.

## 5. Wire contract (serve provider)

`POST {base_url}/recognize` with `{"path": "<absolute media path>"}`;
response is the **recognize response** JSON, schema alongside this note at
[`0004-recognize-response.schema.json`](0004-recognize-response.schema.json)
(draft, non-normative until promotion; the normative copy moves under
`spec/` with the v1 deltas):

```json
{
  "recognizer": {"name": "dirstral-annotator", "version": "0.2.0"},
  "entities": [
    {"id": "player:webb-logan", "label": "Logan Webb", "aliases": ["Webb", "#62"]}
  ],
  "annotations": [
    {
      "start_s": 2530.0,
      "end_s": 2551.0,
      "event": "pitch",
      "entities": ["player:webb-logan"],
      "text": "Pitch: Logan Webb to Freddie Freeman — fly out",
      "confidence": 0.97,
      "sources": ["scorebug", "face"]
    }
  ]
}
```

`recognizer.name`/`version` feed the derivation identity (§4). The
`entities` dictionary is provenance/context for clients and future
entity-aware features; v1 ingestion indexes the `text` statements.

## 6. Retrieval

No tool contract changes. Annotation chunks are ordinary text hits:
BM25 + vector searchable, cited with the file + `time` span, quotable by
`ask`. "Find all pitches by player X" is a plain query the moment the
corpus is indexed.

## 7. Proposed spec deltas (at promotion)

MINOR bump, applied together with the implementation per the spec-first
loop:

- **§8.1.2 capability matrix** — new optional row `recognize`.
- **§5.2** — new representation source value `recognize`.
- **New §8.7 (or next free) Recognition** — the capability, config keys,
  serve wire contract, derivation identity fields, confidence floor,
  failure semantics.
- **§16 config template** — `recognize.provider` / `recognize.base_url`
  (default `off`).
- Tool schemas (`Hit`/`Citation`/`Span` in `spec/tools/schemas/common.json`)
  — **unchanged**.

## 8. Out of scope / open questions

- **Entity-aware query filters** ("player X" as a structured filter rather
  than a text match) — deferred until text matching over canonical labels
  proves insufficient; rhymes with Design 0002.
- **Serving media/clips for editorial** — same open question as Design 0003
  §7.2/§10; unchanged by this design.
- **Hosted recognition providers** — the binding pattern accommodates them;
  specifying any concrete one is future work.
- **Modality coverage** — v1 recognizes `video` only; standalone images and
  audio (diarization-style) are natural follow-ups.
- The **eval harness** (ground-truth scoring against public play-by-play
  data) stays outside the core, shipped with the reference backend.
