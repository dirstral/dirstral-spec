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
the reference backend. Each annotation names two participants in fixed roles:
the pitcher who threw the pitch and the batter who faced it.

**Retrieval configuration.** `mistral-embed`; hybrid (BM25 + vector) fusion
on; rerank off; MMR on; cross-file dedup off; `min_score` 0.05; `k = 5`. Only
the annotation text changes between variants, and the corpus is re-embedded
in full for each.

**Queries and gold rule.** Six team-scoped queries. A hit is correct iff
**both** hold:

1. the club asked about is the club of the **batter** the annotation names
   (resolved from the game's roster), and
2. the annotation's outcome satisfies the query's event term, matched against
   the play description's own vocabulary: *home run* = `homers` or
   `grand slam`; *hit* = those plus `singles` / `doubles` / `triples`;
   *scored* = those plus `scores`.

Both conditions are needed. Scoring the club alone measures **attribution**,
not the query: a Nationals ground-out counts as a correct answer to
"Nationals home run" because the club matches. The distinction is not
academic here, and it is reported below.

**Variants.** The same annotations with the club written into the text three
ways:

| variant | annotation text |
|---|---|
| A: club omitted (shipped) | `Pitch: Robbie Ray to Dylan Crews` |
| B: club appended to each name | `Pitch: Robbie Ray (San Francisco Giants) to Dylan Crews (Washington Nationals)` |
| C: club and role appended | `Pitch: Robbie Ray (pitching for San Francisco Giants) to Dylan Crews (batting for Washington Nationals)` |

**Per-query results** (correct / hits returned, at `k = 5`):

| query | A | B | C |
|---|---|---|---|
| "When did the Giants score" | 3/4 | 2/4 | 2/4 |
| "Giants home run" | 3/4 | **0/4** | **0/4** |
| "Giants get a hit" | 2/4 | 2/3 | 2/4 |
| "When did the Nationals score" | 2/4 | 2/4 | 1/4 |
| "Nationals home run" | 1/4 | 1/4 | 0/3 |
| "Nationals get a hit" | 3/4 | 4/4 | 3/4 |
| **precision** | **58.3%** | **47.8%** | **34.8%** |
| precision, club only | 58.3% | 65.2% | 47.8% |

**Result.** Writing the club into the text degrades retrieval monotonically:
58.3% to 47.8% to 34.8%. There is no trade-off to weigh.

The last row is kept deliberately. Scored on club attribution alone, variant B
appears to *improve* on the shipped text (58.3% to 65.2%), and that reading is
an artefact of the weaker rule. "Nationals home run" scores a perfect 4/4 on
club attribution while only **one** of those four annotations is a home run,
and that query alone contributes three of the four hits separating the two
rows for variant B. An implementer measuring attribution instead of the query
would conclude the opposite of the truth.

The clearest single case is "Giants home run": three correct hits with no club
in the text, **zero** with it, and the top results become a Giants **pitcher**
throwing balls and fouls. Adding the label moved the query's mass onto the
club token, which every annotation carries in **both** roles, and the words
that identified the event stopped deciding the ranking. Marking the role in
prose made it worse again.

The failure is structural, not a matter of phrasing. A label that appears on
every candidate cannot discriminate between candidates, and a label carried in
two roles cannot answer a question about one role. Both properties are normal
for the entities a recognition backend reports: teams, programmes, channels,
recurring participants. Only per-actor labels of high specificity, such as a
player's name, survive text matching, which is why the §6 claim held for the
motivating example and hides the general case.

**Implication for implementers.** Writing entity labels into the annotation
text remains correct and useful for the actor case, and this section does not
retract it. It is not, on its own, a substitute for selecting on the entity.
The wire contract already carries `annotations[].entities` and the entity
dictionary that gives each id a label; an implementation that discards those
ids keeps text matching, with the limits measured above, and loses the
structured entity filter and role-specific selection entirely.

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
  keeping the entity ids without it recovers only half the query.

  **Stored shape.** Per annotation, and recoverable from the chunk that
  annotation produced: the ordered list of entity ids, and the `event` string.
  The corpus-level entity dictionary (id, label, aliases) is stored once per
  representation; it is what renders a filtered result back to a user, and
  discarding it leaves ids nobody can read. Any storage that supports
  "candidates whose annotation references id X" satisfies this; the shape is
  not otherwise constrained. (In the reference implementation the natural home
  is the annotation span's `extra_json`, alongside `words` and `speaker`.)
- **New optional retrieval filter** (next free §9 subsection, alongside the
  language §9.5, date §9.6 and media time-window §9.8 filters). This one
  **does** touch the tool schemas, unlike the row above. Contract:

  * **Request fields.** `dir2mcp_search` / `dir2mcp_ask` MAY accept
    `entities` and `events`, each an array of strings. Absent or empty means
    no filtering on that field, so existing callers observe no change.
  * **Values.** An `entities` element is an id exactly as it appears in
    `annotations[].entities`. An `events` element is matched **literally**
    against the annotation's `event`. The wire's `event` is a free-form
    backend-declared string, so this filter defines **no vocabulary** and
    MUST NOT constrain one; `pitch` and `at_bat` are the reference backend's
    values, not normative ones.
  * **Multi-value semantics.** Within a field, **OR**: a hit matches if the
    annotation references **any** requested entity id (respectively, if its
    `event` equals any requested value). Across fields and against every
    other filter, **AND**. So `entities=[team:x] AND events=[at_bat]` is the
    role-exact selection §8 describes.
  * **Eligibility.** Only hits derived from a recognition annotation carry
    entities and an event. A hit from any other representation does **not**
    match a non-empty filter, mirroring how the media time-window filter
    (§9.8) admits only time-spanned hits.
  * **Unknown values are empty, not errors.** An id or event value that
    exists nowhere in the corpus matches nothing and returns an empty result
    set, exactly as the language (§9.5) and date (§9.6) filters.
  * **Pipeline placement.** Candidate-selection, before dedup, rerank and
    truncation to `k`, so `k` counts only surviving hits and a filtered query
    MAY return fewer than `k`. It removes candidates only; it never reorders
    results or changes result structure (§9.2) or citation format (§9.3).

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

  The v1 shape cannot express a role **within** an annotation: `entities` is
  a flat array of ids and `event` is a single scalar describing the annotation
  as a whole. Role is therefore carried only by the **granularity** of the
  annotations themselves. A backend expresses it by emitting **one annotation
  per role** and distinguishing them with `event`, which the reference backend
  does: a pitch is reported once as `pitch` keyed on the pitcher and once as
  `at_bat` keyed on the batter. An entity filter conjoined with `event` is
  then role-exact, with no new vocabulary.

  This matters because that same duplication has a known cost on the retrieval
  side, where two annotations covering one moment are counted twice by
  generated answers (dir2mcp#784). The two changes therefore constrain each
  other, and consolidating one moment into one annotation is **not** a purely
  local fix: with a flat `entities` array and one `event`, a merged annotation
  has nowhere to record that this id pitched and that one batted, so the
  role-exact selection is lost rather than preserved.

  Consequently: either **keep** role-specific annotations separate and address
  the double-count elsewhere (for example by grouping on the shared time span
  at answer time), **or** define a per-entity role structure in the wire shape
  first and consolidate afterwards. Choosing the second is a v2 wire change and
  belongs in its own design, not in a retrieval bug fix.
- **Serving media/clips for editorial** — same open question as Design 0003
  §7.2/§10; unchanged by this design.
- **Hosted recognition providers** — the binding pattern accommodates them;
  specifying any concrete one is future work.
- **Modality coverage** — v1 recognizes `video` only; standalone images and
  audio (diarization-style) are natural follow-ups.
- The **eval harness** (ground-truth scoring against public play-by-play
  data) stays outside the core, shipped with the reference backend.
