# df-005: Span

- **ID:** df-005
- **Version:** 0.3.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §15.1.1 (migrated verbatim; one normative clarification added, see Changelog)

## Scope

A `Span` is the provenance coordinate for a citation: a line range, a page, a
time range, a page-region with a bounding box, or a whole document. It is a
shared type referenced by [df-006 (`Hit`/`Citation`)](df-006-hit-citation.md)
and by the `open_file` tool output.

## Specification (normative)

### Recognition metadata on a `time` span

A span produced by the recognition capability MAY carry the attribution the
recognizer recorded:

* `entities`, an array of entity ids;
* `event`, the producer's event token.

These are the values `dir2mcp_search` and `dir2mcp_ask` filter on through their
`entities` and `events` arguments (bs-007). A served hit that carries them lets a
caller see WHY it matched rather than only that it did. The filter is otherwise
opaque: a caller who asks for `events: ["home_run"]` and receives five hits has
no way to confirm all five carry that event, nor to tell a filtered result from
an unfiltered one.

A span MAY also carry `derivation`, either `observed` or `generated`. It answers
a question a reader of the text cannot: whether the annotation RECORDS something
or DESCRIBES it.

* `observed` is a reading of something the source recorded: a play-by-play feed
  entry, text read off an on-screen overlay, a face matched against a bank.
* `generated` is a model's description of the media in its own words.

Absent means `observed`, which keeps every producer written before this field
correct without changing its output.

**A consumer MUST NOT present a `generated` annotation as the source's own
account of what happened.** The two are not interchangeable. "Ball, called
strike" read from a feed is a fact about the game; "the crowd erupts as the ball
clears the wall" is one model's reading of some pixels, and it can be
confidently wrong in a way a feed entry cannot. Fluency is why the rule is
needed: a confident sentence reads like evidence, so the more convincing a
description is, the more a reader needs to know it is one.

A span MAY also carry `sources`, an array naming the recognizers that
contributed to the annotation, for example `["scorebug"]` or
`["playbyplay", "face"]`.

`sources` and `derivation` answer different questions and neither replaces the
other. `derivation` says whether an annotation RECORDS or DESCRIBES, which is a
two-value judgement about kind. `sources` says WHICH component produced it,
which is what lets a client weigh one reading against another.

That difference is practical, not theoretical. Recognizers within one
implementation vary widely in reliability: a scorebug reader that covers most of
a broadcast and a face matcher that covers a fraction of it are both `observed`,
and a viewer asking "how do you know" deserves to be told which one spoke. A
client MAY therefore use `sources` to explain a hit or to weigh it.

**The vocabulary is producer-defined and NOT enumerated here.** The recognizers
an implementation runs are its own design, and freezing their names in this
document would make adding one a spec change. A client MUST therefore treat an
unrecognized tag as opaque and render it verbatim rather than error, which is
the same rule the skip-reason enum states for its own additive growth.

`sources` is provenance, not a relevance signal. An implementation MUST NOT
require a client to read it in order to rank or filter correctly.

All four fields are **optional and additive**, exactly as `speaker` is: a
consumer that does not recognize them MUST treat the span as it did before. A
chunk that is not a recognition annotation carries none of them.


A `Span` is exactly one of five variants, selected by `kind`. Each variant is
`additionalProperties: false`.

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
        "speaker": { "type": "string", "description": "Optional (td-003): stable per-transcript speaker id on a diarized transcript." },
        "speaker_label": { "type": "string", "description": "Optional human-readable speaker name (td-003)." },
        "entities": { "type": "array", "items": { "type": "string" }, "description": "Optional (bs-007 design 0004): entity ids the recognizer attributed to this span." },
        "event": { "type": "string", "description": "Optional (bs-007 design 0004): the producer's event token for this span. Vocabulary is producer-defined." },
        "derivation": { "type": "string", "enum": ["observed", "generated"], "description": "Optional: whether this span records something observed or something a model generated. Absent means observed." },
        "sources": { "type": "array", "items": { "type": "string" }, "description": "Optional: the recognizers that contributed to this annotation. Producer-defined tags." }
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

A producer **MUST** emit exactly one of the five defined `kind` values. A chunk
that lacks finer provenance **MUST** be serialized as a `page` span (when a page
is known) or a `document` span — **never** with an empty or unrecognized `kind`.
A `Span` whose `kind` is absent, empty, or outside the enum is non-conforming
and **MUST** be rejected by a strict client (it matches no `oneOf` branch). *(This
clarification codifies the fix for dir2mcp #397: a BM25 hit lacking span
metadata previously serialized `{"kind":""}`, which a strict MCP client rejects
as "Failed to call tool"; the corrected behavior is a `document` fallback.)*

The `region` variant is emitted by structured document extraction (td-004). It
localizes a chunk to a page range (`start_page`/`end_page`, equal when
single-page) and always carries a bounding box (`bbox`); an element without
provenance is recorded as a `page` span instead, never a `region` span with a
missing `bbox`. The section breadcrumb (`section`) is optional (`[]` when none).
The `region` kind and its `section` field are **additive**: clients that do not
recognize the `region` kind, or that ignore `section`, **MUST** degrade
gracefully (treat as a page-level citation on `start_page`).

The `document` variant is emitted by `dir2mcp_open_file` when the requested
`rel_path` is a binary doc type (PDF, audio) and the caller did not supply
`page`, `start_ms`/`end_ms`, or `start_line`/`end_line`. It signals that
`content` is the full extracted / transcript representation rather than a paged
or timed slice.

## Examples

```json
{ "kind": "lines", "start_line": 10, "end_line": 18 }
{ "kind": "page", "page": 4 }
{ "kind": "time", "start_ms": 120000, "end_ms": 135000, "speaker": "S1", "speaker_label": "Anchor" }
{ "kind": "region", "start_page": 4, "end_page": 5,
  "bbox": { "page": 4, "l": 72.0, "t": 90.0, "r": 523.0, "b": 410.0, "coord_origin": "TOPLEFT" },
  "section": ["VIRGIN ISLANDS", "Power to provide assistance"] }
{ "kind": "document" }
```

## Changelog

- **0.3.0**: added the optional `time` span field `sources`, an array naming
  the recognizers that contributed to an annotation. It is not what `derivation`
  says: `derivation` reports whether an annotation records or describes, and
  `sources` reports which component produced it, so a client can weigh a
  scorebug reading against a face match. The vocabulary is producer-defined and
  deliberately not enumerated, and a client MUST render an unrecognized tag
  verbatim. Provenance only, never required for ranking. Unblocks dir2mcp #861.
- **0.2.0**: added three optional `time` span fields. `entities` and `event`
  carry the attribution a recognizer recorded, which is what the `entities` and
  `events` filters match on, so a served hit can show WHY it matched.
  `derivation` (`observed` or `generated`) separates a reading of something
  recorded from a model's description of the media; absent means `observed`, so
  every earlier producer stays correct. All three are additive in the same way
  `speaker` is: a consumer that does not recognize them treats the span as it
  did before. This document is the source of truth for them, and SPEC.md points
  here. Unblocks dir2mcp #856; precondition for dir2mcp #860.
- **0.1.0** — Migrated from SPEC.md §15.1.1. Added the normative MUST that a
  producer emit a defined non-empty `kind` (codifies dir2mcp #397). Updated
  internal cross-references from `§8.6.8`/`§7.4.B` to `td-003`/`td-004`.
