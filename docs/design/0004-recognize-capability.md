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
  each to the **nearest** millisecond — `start_ms = round(start_s * 1000)`,
  `end_ms = round(end_s * 1000)` (millisecond resolution is far finer than a
  video frame, so nearest-rounding never moves a boundary by a
  perceptible amount). An annotation that is **malformed** — empty or
  whitespace-only `text`, or a reversed span (`end_s < start_s`, i.e.
  `end_ms < start_ms` after rounding) — is **dropped** (not persisted); the
  remaining annotations of the same document proceed. A zero-length span
  (`end_ms == start_ms`) is kept and denotes an instantaneous event.
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

Annotation chunks are ordinary text hits: BM25 + vector searchable, cited
with the file + `time` span, quotable by `ask`. "Find all pitches by player
X" is a plain query the moment the corpus is indexed.

That holds for the **actor** of a statement, whose canonical label is
distinctive and appears in the annotations that are actually about them. It
does **not** hold for an entity that (a) appears in most or all annotations
and (b) participates in more than one **role**. §6.1 records the measurement;
§8 promotes the entity filter that follows from it.

### 6.1 Measured limit of text-only entity matching (pilot, 2026-08)

Design 0004 deferred entity-aware filters "until text matching over canonical
labels proves insufficient" (§8). This section records the run that met that
condition, so the promotion in §7 rests on evidence rather than preference.

**Corpus.** One baseball broadcast, 346 recognition annotations produced by
the reference backend, embedded with `mistral-embed`, hybrid retrieval on.
Each annotation names two participants in fixed roles: the pitcher who threw
the pitch and the batter who faced it.

**Task.** Six team-scoped queries ("when did the Giants score", "Nationals
home run", ...). A hit is correct iff the team asked about is the team of the
**batter** the annotation names, judged against the roster.

**Variants.** The same annotations, re-embedded, with the team label written
into the chunk text three ways:

| variant | annotation text | precision |
|---|---|---|
| labels omitted | `Pitch: Robbie Ray to Dylan Crews` | 58.3% |
| team appended to each name | `Pitch: Robbie Ray (San Francisco Giants) to Dylan Crews (Washington Nationals)` | 65.2% |
| team + role appended | `Pitch: Robbie Ray (pitching for San Francisco Giants) to Dylan Crews (batting for Washington Nationals)` | 47.8% |

**Result.** No variant is usable, and the ranking is not the interesting part.
Under the best-scoring variant the query "Giants home run" returned **zero**
correct hits where the label-free text returned three; its top results were a
Giants **pitcher** throwing balls and fouls. Adding the label moved the
query's mass onto the team token, which every annotation carries in **both**
roles, and the words that identified the event ("home run") stopped deciding
the ranking. Marking the role in prose made it worse, not better.

The failure is structural, not a matter of phrasing. A label that appears on
every candidate cannot discriminate between candidates, and a label carried in
two roles cannot answer a question about one role. Both properties are normal
for the entities a recognition backend reports: teams, programmes, channels,
recurring participants. Only per-actor labels of high specificity, such as a
player's name, survive text matching, which is why the §6 claim held for the
motivating example and hides the general case.

**Implication for implementers.** Writing entity labels into the annotation
text remains correct and useful, and this section does not retract it. It is
not, on its own, a substitute for selecting on the entity. The wire contract
already carries `annotations[].entities` and the entity dictionary that gives
each id a label; a conforming implementation that discards those ids has no
way to express the query at all.

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
- **Annotation entities and `event` are persisted.** The ids in
  `annotations[].entities`, the `entities` dictionary that gives each id its
  label and aliases, and the annotation's `event`, MUST survive ingestion and
  be recoverable per annotation. Persisting the annotation text alone is not
  conforming: it makes §6.1's filter unimplementable and silently discards
  data the backend was required to compute. `event` is named here rather than
  left implicit because it is what makes the filter role-exact (§8), so
  keeping the entity ids without it recovers only half the query. (In the
  reference implementation the natural home is the annotation span's
  `extra_json`, alongside `words` and `speaker`.)
- **New optional retrieval filter** (next free §9 subsection, alongside the
  language §9.5, date §9.6 and media time-window §9.8 filters):
  `dir2mcp_search` / `dir2mcp_ask` MAY accept an optional set of entity ids
  that restricts results to annotations referencing them. Additive and off by
  default; conjunctive with every other filter; an empty result is not an
  error. This one **does** touch the tool schemas, unlike the row above.

## 8. Out of scope / open questions

- ~~**Entity-aware query filters** ("player X" as a structured filter rather
  than a text match) — deferred until text matching over canonical labels
  proves insufficient; rhymes with Design 0002.~~ **Resolved (2026-08):** the
  deferral condition was met. §6.1 records the measurement; §7 carries the
  resulting deltas. The filter selects on the entity ids the wire contract
  already defines, so no new vocabulary is introduced here. Whether an entity
  additionally carries its **role** in an annotation (this id was the pitcher,
  that one the batter) is the one genuinely open question the measurement
  raises: without a role, "Giants batting" and "Giants pitching" remain
  indistinguishable to the filter exactly as they were to text matching.

  A backend can already express the distinction without new vocabulary, by
  emitting **one annotation per role** and distinguishing them with `event`
  (the reference backend does this: a pitch is reported once as `pitch` keyed
  on the pitcher and once as `at_bat` keyed on the batter). An entity filter
  conjoined with `event` is then role-exact. This is worth stating plainly
  because the same duplication has a known cost on the retrieval side, where
  two annotations covering one moment are counted twice by generated answers
  (dir2mcp#784). Any consolidation of those annotations MUST preserve both
  role-attributed entity references, or it will fix the double-count by
  destroying the only signal that makes the filter role-exact.
- **Serving media/clips for editorial** — same open question as Design 0003
  §7.2/§10; unchanged by this design.
- **Hosted recognition providers** — the binding pattern accommodates them;
  specifying any concrete one is future work.
- **Modality coverage** — v1 recognizes `video` only; standalone images and
  audio (diarization-style) are natural follow-ups.
- The **eval harness** (ground-truth scoring against public play-by-play
  data) stays outside the core, shipped with the reference backend.
