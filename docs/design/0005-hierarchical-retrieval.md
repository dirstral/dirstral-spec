# Design 0005 — Hierarchical retrieval (summary representations + coarse-to-fine)

**Status:** Proposed (targets spec `0.41.0`)
**Author:** dirstral maintainers
**Related:** SPEC §5.2 (representations), §7.4 (chunking), §8.6.1 (transcript
segments), §8.6.7 (representation provenance / re-derivation), §9 (retrieval),
§16 (configuration); dir2mcp #329, #395; Design
[0004](0004-contextual-retrieval.md) (contextual retrieval, complementary)

## 1. Summary

Add a **`summary` representation** per document (and per media **event/section** —
a summary over N adjacent transcript/clip chunks), embedded alongside the fine
chunks, and a **coarse-to-fine** retrieval step: match a coarse summary, then drill
into the fine evidence beneath it. This is the well-known **RAPTOR /
parent-document** technique (CastleRAG's "2-minute event summaries" over 30s clips
are the media instance). For long documents and long media the answer often needs
document- or section-level context that no single 120s window or text chunk
carries.

**Opt-in, off by default, domain-general.** Complementary to Design 0004: 0004
*transforms* a chunk's embed input (and re-embeds); this *adds* coarser vectors and
changes the retrieval **flow** — the two compose.

## 2. What this is NOT (the key contrast with 0004)

Contextual retrieval (0004) changes the vector of **every existing chunk**, so it
folds into the §8.1.4 embed identity and forces a re-embed. Hierarchical retrieval
does **not**: a `summary` is an **additional** representation with its **own**
vectors, embedded by the **same** model into the **same** space as chunks. Turning
it on **adds** summary vectors; turning it off **removes** them. Existing chunk
vectors are untouched.

Therefore:

- **No embed-identity change.** `summary` vectors live in the same axis/space as
  their doc's chunks (same embed model), so they are directly comparable — no
  §8.1.4 field, no chunk re-embed on toggle.
- **A `summary` IS a derived representation** (§5.2, §8.6.7), like `transcript` /
  `extracted_markdown` / `annotation_text`. It carries a **derivation identity** and
  **re-derives** (text **and** child linkage, §4) whenever any of its inputs change:
  1. the **source content** it summarizes (the covered chunks/segments);
  2. the **generator identity** — provider, model, and the **effective prompt**.
     `prompt_version` names the built-in template; an operator `prompt` **override**
     is **hashed into the identity** (a version tag alone cannot detect an
     edited override), so any change to the effective prompt re-derives;
  3. for a **section** summary, the **windowing inputs** — `level`,
     `section_units` / `section_seconds`, and the identity of the underlying
     chunking / transcript segmentation that defines the fine units. Changing
     `section_units` (say 8→16) re-windows the sections, so the summary **text** and
     its **child span-range linkage** (§4) both re-derive; a stale linkage is
     invalidated, not just stale text. (A **document**-level summary has no
     windowing input — inputs 1–2 only.)
  This is the standard §8.6.7 provenance machinery with the section-windowing inputs
  folded in — not a new mechanism.

## 3. Citation faithfulness — summaries retrieve, chunks cite

A `summary` is **model-generated prose**, not source text. It MUST NOT be presented
as a verbatim quote. The rule:

> A summary hit **drives retrieval** (it expands to the fine chunks beneath it); the
> **citations come from those fine chunks**, which carry the real `Span` (§5.4) into
> the source. A summary is **never** a `Citation.snippet` or an answer quote.

So hierarchical retrieval changes *which fine chunks surface*, never *what a citation
points at* — `Hit`/`Citation`/`Span` (df-006/df-005) are unchanged, and the
conformance surface is untouched. (If a future tool wants to *return* a summary, it
does so as an explicitly-labelled `rep_type: summary`, never as a source quote — out
of scope here.)

## 4. Data model

- **`summary` `rep_type`** (§5.2), `index_kind=text`. Two levels:
  - **document** — one summary over the whole document.
  - **section** — a summary over a deterministic window of N adjacent fine units
    (text chunks, or transcript segments / media clips for time media — CastleRAG's
    event summaries). Windows are deterministic (like §8.1.7 / §8.6.1 windowing) so
    summary spans are stable across re-index.
- **Parent→child linkage.** Each `summary` records the **span range of the fine
  units it covers** (a document span, or a `time`/`lines` range), so a summary hit
  can be expanded to exactly its children without a separate join table — the range
  is the link. Section summaries at the same level MUST tile without overlap so a
  child maps to exactly one section summary (dedup correctness, §5).
- Summaries are embedded and BM25-indexed as normal text representations; they are
  tagged `rep_type=summary` (+ level) so retrieval can distinguish coarse from fine.

## 5. Retrieval flow (unified index, coarse-to-fine expand)

Reuse the **single** vector+BM25 index (no separate summary store):

1. Retrieve the top candidates across **both** summary and chunk vectors (one search,
   as today).
2. For each **summary** hit, **expand** to its child fine chunks (§4 range) and add
   them to the candidate pool; a **chunk** hit is kept as-is.
3. **Dedup** (a fine chunk reached both directly and via its summary appears once)
   and **rerank** (§8.4) the merged pool, then truncate to `k`.
4. Return `Hit`s — **fine chunks only** by default (summaries retrieve, chunks cite,
   §3). A summary hit with no surviving children after filtering contributes nothing
   to the result (it is a routing device, not an answer).

This keeps one index, one search, and the existing rerank/dedup — the addition is
the *expand* step. It also degrades gracefully: a corpus with no summaries (feature
off, or a doc whose summary failed, §7) behaves exactly as flat retrieval today.

## 6. Generation — cost, bounds, fail-open

- Summaries are generated via the configured **chat/annotator** generator (§8.1.3),
  bounded (`max_tokens`), and cached content-addressed (§8.6.7) so they re-derive
  only on change.
- Cost is **O(documents + sections)**, not O(chunks) — far cheaper than 0004's
  per-chunk generation; a document summary is one call, a section summary one per
  window.
- **Fail-open per document/section:** if summary generation fails for a unit, that
  unit simply has **no summary** — retrieval falls back to flat behavior **for that
  unit**, and the miss is recorded (honest coverage, §8) and **retried** on the next
  scan. No summary ever blocks ingest of the underlying chunks.

## 7. Interaction with existing surfaces

- **Media (§8.6):** section summaries over adjacent transcript segments / clips are
  the "event summary" case; they use `time` spans and coexist with per-track
  transcripts (Design/§8.6.12) — a summary is over a track's segments.
- **Chunking (§7.4):** summaries are an **additional** representation, not a change
  to how chunks are cut; flat chunking is unchanged.
- **more_like_this / related (§15.12, #324):** a `chunk_id` there still targets a
  fine chunk; summaries are internal routing and are not addressable as a source.

## 8. Honest coverage

Track, per document, whether a summary exists (`present` / `fallback` / `disabled`)
and surface a document-summary coverage count as a later additive `dir2mcp_stats`
field (spec-first like watch_overflows / skip_reasons), so an operator sees how much
of the corpus is hierarchically indexed vs. flat.

## 9. Configuration (sketch — normative form in the spec PR)

```yaml
retrieval:
  hierarchical:
    enabled: false            # opt-in; off by default (flat retrieval unchanged)
    levels: [document]        # document | section (or both); section adds windowed summaries
    section_units: 8          # section summary window size: N adjacent chunks
    section_seconds: 120      # media: N seconds of adjacent segments per event summary
    provider: ""              # optional generator profile; empty => configured chat/annotator
    max_tokens: 512
    prompt_version: v1        # names the built-in template; part of the summary derivation identity (§2)
    # prompt: ""              # optional override of the (general, domain-free) template; the EFFECTIVE
                              #   prompt is hashed into the derivation identity, so an edited override
                              #   re-derives summaries even without bumping prompt_version
```

## 10. Spec surface for the follow-up spec PR

1. **§5.2** — add `summary` to the `rep_type` enum + its parent→child span-range
   linkage and the tiling-without-overlap rule (§4).
2. **§8.6.7** — the `summary` derivation identity: source content + generator
   (provider/model/**effective prompt**, override hashed) + (section only) the
   windowing inputs (`level`/`section_units`/`section_seconds` + fine-unit
   chunking/segmentation identity). Re-derive **text and child linkage** on change;
   **not** an embed-identity field (§2).
3. **§9** — the coarse-to-fine retrieval flow: expand summary hits to children,
   dedup, rerank, return fine chunks (§5); citation faithfulness (§3).
4. **§16** — the `retrieval.hierarchical` config block.
5. Optional additive stats coverage field (§8).

No new **tool** and **no served-schema change**: results are the existing `Hit`
shape; summaries never reach the wire as citations.

## 11. Phasing

- **P1:** document-level summaries + coarse-to-fine expand (the bulk of the win,
  cheapest generation).
- **P2:** section/event summaries (text windows + media time-windows).
- **P3:** the stats coverage field; optional multi-level (RAPTOR-style recursive)
  summaries if P1/P2 prove valuable.

## 12. Alternatives considered

- **Separate summary index / two-stage retrieval** — rejected for P1: a unified
  index with an expand step reuses the existing search/rerank/dedup and avoids a
  second store and a second round-trip; two-stage can be revisited if recall demands
  it.
- **Return summaries as answers/citations** — rejected: breaks faithfulness (§3); a
  generated summary is not source text.
- **Fold hierarchical into the embed identity** — rejected: summaries are *additive*
  vectors in the same space, not a transform of chunk vectors, so a toggle is an
  index add/remove, not a re-embed (§2). Conflating it with 0004's identity fold
  would force needless full reindexes.
