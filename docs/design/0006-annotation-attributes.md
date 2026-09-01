# Design 0006 — Annotation attributes: structured scopes a filter can require

**Status:** Proposed (targets a future MINOR spec bump)
**Author:** dirstral maintainers
**Extends:** SPEC §9.9 (annotation entity/event filter), Design 0004 (recognition capability, annotation wire contract), §15.2/§15.3 (tool inputs)
**Related:** SPEC §9.6 (date filter), §9.8 (media time-window filter): the same "structured attribute, not similarity" argument, one level more general

## 1. Summary

Let a recognition backend attach **flat key/value attributes** to an
annotation (`{"inning": "8", "half": "bottom"}`), persist them, and let
`dir2mcp_search` / `dir2mcp_ask` **require** them with a new optional
`attributes` filter. Values are matched literally; the vocabulary is
producer-defined; dir2mcp learns no domain.

## 2. Motivation, measured

On the SF Giants pilot (issue dir2mcp#928), `what happened in the 8th inning`
at `k=12` returned moments from the 3rd, 6th and 7th innings above 8th-inning
moments. The proximate cause was an MMR setting, but fixing it exposed the
structural limit: **the inning exists only as prose** ("bottom of the 8th"
inside the annotation text). A value that lives only in text can be
*preferred* by similarity, never *required* by a filter. The embedding barely
separates ordinals at all: the score spread across twelve play-by-play hits
was 0.0137–0.0250, i.e. the vector matches "baseball pitch text", not "8th".

The recognizer HAS this value as structured data (the play-by-play feed
publishes the inning per event) and is forced to throw that structure away at
the wire, because the Design 0004 annotation object is
`additionalProperties: false` with no field to carry it.

§9.9 (spec 0.58.0) records this gap explicitly and defers it here. The same
section records why the cheap fix is wrong: widening `events` into a general
attribute channel would overload one field with two meanings, and every
consumer of `event` (the §9.9 filter, recognitionSegments persistence, the
6.2 entity filter UI) would need to know which meaning it is looking at.

The shape is domain-neutral. An inning is a period of a baseball game; a
period of a hockey game, a chapter of an audiobook, an act of a play, a
quarter of an earnings call are the same thing: **a producer-defined scope a
user filters by, whose values are equality-matched, and which is not a time
range from the server's point of view** (the server does not know when the
8th inning starts; the producer does).

## 3. What this deliberately is not

* **Not a range query.** Values match by literal equality only. "innings 7
  through 9" is three values OR-ed, not a range. Ordering, ranges and
  arithmetic on attribute values are out of scope: the moment values have
  order, they have types, units and comparison semantics, and the right tool
  for "a contiguous stretch of the timeline" already exists (§9.8). A
  producer who wants range-like scoping SHOULD also emit accurate time spans,
  which it must do anyway (Design 0004 requires them).
* **Not nested.** One flat string→string map per annotation. Nesting invites
  schema growth without a driving use case; every motivating example above is
  flat.
* **Not an entity and not an event.** Entities name WHO is in the span,
  `event` names WHAT the span shows, attributes name WHERE IN THE WORK the
  span sits (plus whatever else the producer keys by). The three compose:
  `entities=[player:m-chapman] AND events=[home_run] AND
  attributes={"inning":["8"]}` is the fully scoped question.

## 4. The four layers

### 4.1 Recognizer wire contract (Design 0004 schema)

`Annotation` gains one optional property:

```json
"attributes": {
  "type": "object",
  "additionalProperties": { "type": "string" },
  "description": "Optional flat producer-defined key/value scopes for this annotation (e.g. {\"inning\": \"8\", \"half\": \"bottom\"}). Keys and values are opaque strings; the contract enumerates neither. Values are equality-matched by the SPEC 9.10 filter, so producers SHOULD normalize (lowercase, no leading zeros) at emission."
}
```

Values are **strings on the wire even when semantically numeric**, because
the filter is literal equality and one representation avoids `8` vs `"8"` vs
`8.0` mismatches; the producer normalizes once at emission. The object stays
`additionalProperties: false` overall; this is the one new key.

### 4.2 Persistence

Same clause as 0.58.0 gave entities and event: the attributes map MUST
survive ingestion and be recoverable per annotation. Persisting the
annotation text alone is non-conforming: it makes the filter unimplementable
and silently discards data the backend computed. In the reference
implementation the natural home is the annotation span's `extra_json`,
alongside `entities` and `event`.

### 4.3 Filter semantics (new SPEC §9.10, mirroring §9.9)

`dir2mcp_search` / `dir2mcp_ask` MAY accept optional `attributes`: an object
mapping attribute keys to arrays of acceptable values.

```json
"attributes": { "inning": ["8"], "half": ["bottom", "top"] }
```

* **Within one key: OR.** The annotation's value for that key must equal ANY
  listed value.
* **Across keys, and against every other filter: AND.**
* **Values match literally**, case-sensitively, as strings. No vocabulary is
  defined or constrained.
* **Eligibility:** only annotation-derived hits carry attributes; a hit from
  any other representation never matches a non-empty filter (mirrors §9.8 and
  §9.9). An annotation that lacks a requested KEY does not match.
* **Unknown keys or values are empty results, not errors** (mirrors §9.5,
  §9.6, §9.9).
* **Pipeline placement:** candidate-selection, before dedup/rerank/truncation,
  so `k` counts survivors (mirrors §9.9).

### 4.4 Tool schemas

`spec/tools/schemas/search.json` and `ask.json` inputs gain the `attributes`
object above. Additive and optional; existing callers observe no change.
The served schemas in the reference implementation gain it in the same PR
that implements the filter, per the 0.58.0 lesson (declare what is served,
in the same version that serves it).

## 5. Pilot fit (the concrete first consumer)

The pilot's annotator already receives per-event inning data from the feed
and already emits `event` and `entities` per annotation. Emitting
`attributes: {"inning": "8", "half": "bottom"}` is a few lines in the
play-by-play recognizer. Re-annotation of one game regenerates the corpus.
The demo question "what happened in the 8th inning" then becomes a query the
UI can send as `attributes={"inning":["8"]}` and the server can REQUIRE,
independent of embedding quality, MMR settings, or how the model words the
prose.

## 6. Spec deltas at promotion

MINOR bump, applied with the implementation per the spec-first loop:

- Design 0004 response schema: the `attributes` property (§4.1 above).
- Persistence clause extension (alongside the 0.58.0 entities/event clause).
- New SPEC §9.10 with the semantics of §4.3.
- `search.json` / `ask.json` inputs: the `attributes` object.
- §9.9's closing paragraph (the recorded gap) updated to point here.

## 7. Open questions

- **Key collision with future core semantics.** If dir2mcp ever wants a
  reserved attribute (say `lang`), producer-defined keys could collide. The
  contract could reserve a `dir2mcp:` prefix now, at zero cost, and leave the
  rest of the namespace to producers. Proposed: do exactly that.
- **Advertising available keys.** A client UI would like to know that this
  corpus has `inning` with values 1–9 before offering a filter control.
  `dir2mcp_stats` could aggregate distinct attribute keys (not values, which
  may be unbounded). Deferred: the pilot UI hardcodes its domain, and stats
  growth deserves its own thought.
