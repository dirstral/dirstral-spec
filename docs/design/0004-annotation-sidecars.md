# Design 0004 — Annotation sidecars (external recognizer output as citable representations)

**Status:** Proposed (v0 requires **no spec change**; v1 targets a future MINOR bump)
**Author:** dirstral maintainers
**Extends:** SPEC §8.6.4 (subtitle sidecars), §5.2 (representation sources), §5.4 (spans)
**Related:** [Design 0003 §7.2/§10](0003-multimodal-embeddings.md) (media inspection / media-fetch open question), [Design 0002](0002-structured-extraction.md) (structured extraction)

## 1. Summary

Let **external annotation pipelines** — computer-vision or other recognizers
that dir2mcp deliberately does *not* contain — publish their output as
**time-coded sidecar files next to media**, which dir2mcp indexes into
ordinary retrievable, citable representations. A query like *"all pitches by
player X in game Y"* then resolves through the existing `search`/`ask` tools
to hits with `time`-span citations an editor can cut clips from.

Two slices:

- **v0 (now, zero core change):** a *convention* — annotators emit their
  output as WebVTT cues, and the existing subtitle-sidecar mechanism
  (§8.6.4) indexes them. This is enough for a pilot.
- **v1 (proposed, spec change):** a dedicated machine-readable annotation
  sidecar format carrying **entities, event types, confidence, and
  annotator provenance**, with derivation-identity semantics that authored
  subtitles deliberately lack.

The motivating use case is a sports-video pilot (player identity + event
recognition over broadcast/archival baseball footage), but nothing here is
sports-specific: the same shape fits speaker diarization, scene labeling,
logo/object detection, or any recognizer that maps media time ranges to
statements about content.

## 2. Motivation

- **Keep vision out of the core.** dir2mcp is an index/retrieve/cite server
  (VISION.md's non-goals are explicit). Face recognition, jersey OCR,
  scorebug OCR, action detection — these belong in a separate annotator
  component with its own stack. The contract between the two should be a
  *file format*, not an API or plugin surface.
- **The retrieval side already fits.** Time spans are a persisted
  `span_kind` (§5.4); subtitle sidecars already flow into indexed,
  BM25 + vector-searchable transcript representations with time-coded
  citations; `avutil`-style window extraction already exists in the
  reference implementation for multimodal chunks (Design 0003). The delta
  for v0 is literally zero.
- **Complementary to Design 0003.** Multimodal embeddings give *fuzzy*
  visual recall ("a pitcher mid-windup"); annotations give *exact* entity
  and event recall ("Logan Webb, pitch, 00:42:10"). A corpus benefits most
  from both in the same index.

## 3. v0 — the WebVTT convention (no spec change)

An annotator writes, next to `game7.mp4`, a file the existing §8.6.4
mechanism already recognises (`.vtt`/`.srt`/`.ttml`, checked
case-insensitively), e.g. `game7.vtt`:

```
WEBVTT

00:42:10.000 --> 00:42:31.000
Pitch: Logan Webb (#62) to Freddie Freeman — fly out [sources: scorebug+face; confidence 0.97]
```

What falls out for free:

- The cues become an indexed transcript representation
  (`source: sidecar`, §5.2); cue text is chunked, BM25- and
  vector-searchable; each hit cites the file plus a `time` span.
- `open_file` returns the cue text as the document representation, so
  `ask` can ground and quote it.
- No new tool, schema, config, or error code. `Hit`/`Citation`/`Span` in
  `spec/tools/schemas/common.json` are untouched.

**Known v0 limitations** (accepted for a pilot, and the reason v1 exists):

1. **Wrong provenance semantics.** §8.6.7 treats sidecars as *authored*:
   they carry no derivation identity and are never invalidated by a model
   change. Machine-generated annotations are *derived* — re-running an
   improved recognizer should be able to supersede them. In v0 the
   annotator must overwrite the file itself (mtime/content-hash change
   triggers re-ingest), which works but conflates authored and derived
   content.
2. **One sidecar slot.** A media file with a real subtitle sidecar *and* an
   annotation sidecar contends for the same convention. v0 workaround:
   merge annotations into one VTT, or use the language-suffix convention
   to keep files distinct — both are workarounds, not designs.
3. **Entities are strings.** "Logan Webb" is text to match, not an entity
   to filter on. Retrieval quality then depends on annotators emitting
   consistent, canonical name spellings (which the pilot's annotator can
   simply do).
4. **Confidence is prose.** Thresholding or ranking by recognizer
   confidence is impossible; it is display-only.

## 4. v1 — a dedicated annotation sidecar (proposed)

A sibling file, e.g. `game7.annotations.json` (exact suffix TBD), shaped
roughly as below — a draft (non-normative) JSON Schema for this format
lives alongside this note at
[`0004-annotation-sidecar.schema.json`](0004-annotation-sidecar.schema.json);
the normative copy moves under `spec/` with the v1 spec deltas:

```json
{
  "annotator": {"name": "dirstral-sports-annotator", "version": "0.3.1"},
  "media": "game7.mp4",
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

Ingestion maps each annotation to a chunk whose text is the `text` field
(so BM25/vector search work unchanged), with a `time` span from
`start_s`/`end_s` — reusing the persisted `span_kind` set exactly as
Design 0003 does (no new persisted kind).

**Semantics that differ from authored sidecars:**

- **New representation source** (§5.2): `annotation` — machine-derived,
  distinct from both `sidecar` (authored, never invalidated) and `stt`
  (derived from the media by a configured provider).
- **Derivation identity** is the annotator `name`/`version` *declared in
  the file*, not a dir2mcp-configured provider: dir2mcp never runs the
  annotator, so a changed identity in a re-published file supersedes the
  old representation, but no dir2mcp model-config change ever invalidates
  annotations (§8.6.7 stays untouched for real sidecars).
- **Confidence** is stored per chunk; a per-corpus floor (config, default
  0) filters at ingest. Ranking *by* confidence is explicitly out of scope
  for v1 (it is provenance metadata, not a relevance signal).

**Spec deltas at v1** (MINOR, exact version assigned when promoted):
`§5.2` new source value; a new `§8.6.x` defining the file convention,
JSON shape, and supersession rule; `§16` the confidence-floor key.
Tool schemas unchanged — hits and citations are ordinary time-span hits.
Per the spec-first loop, these land here before any implementation PR
re-pins.

## 5. Out of scope (recorded, not designed)

- **Entity-aware query surface.** "player X" as a structured *filter*
  (rather than a text match) touches the tool contracts (`search` input
  schema) and rhymes with Design 0002's structured-extraction direction —
  deferred until v1 usage shows text matching over canonical labels is
  insufficient.
- **Serving the media itself** (clip fetch for editorial). Same open
  question as Design 0003 §7.2/§10; a successful pilot is the forcing
  function to resolve it once, for both designs.
- **Running annotators.** dir2mcp never invokes, schedules, or validates
  recognizers; it indexes what they publish. This is a hard boundary, not
  a phasing decision.

## 6. Open questions

- Sidecar coexistence: exact filename grammar so subtitle sidecars,
  translated sidecars (§8.6.2), and annotation sidecars never collide.
- Multiple annotators per media file: last-writer-wins vs. coexisting
  representations keyed by annotator name.
- Whether `entities[].id` deserves a registry convention (stable IDs
  across a corpus) or stays free-form per file.
- Whether annotation text should be excluded from `ask` grounding when
  confidence is below a (second) threshold, or grounding trusts whatever
  passed ingest.
