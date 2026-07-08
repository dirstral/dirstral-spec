# bs-003: Retrieval and answer generation

- **ID:** bs-003
- **Version:** 0.2.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §9

## Scope

Query-time behavior: search routing across the text and code indices, optional
reranking, hit structure and provenance, cross-file de-duplication, human-readable
citation formatting, RAG answer generation, and the optional per-language
retrieval filter. The retrieval hit/citation shape is normative in
[df-006](../data-formats/df-006-hit-citation.md); the `span` provenance type is
[df-005](../data-formats/df-005-span.md) — both are referenced here, not
redefined. The ANN/vector index and BM25/lexical mechanics consumed at candidate
generation are [bs-008](bs-008-vector-index.md).

## Specification (normative)

### Search routing

At query time:

- `index=auto`:
  - default to `text`
  - choose `code` if the query is code-oriented (heuristic) or filters target code
- `index=both`:
  - query both indices and fuse results
  - normalization: per-index score normalization, then merge

### Optional reranking

Reranking is optional; it is a retrieval-quality optimization, not a hard
dependency. It is **auto-enabled when a rerank provider credential is present**
(e.g. `COHERE_API_KEY`) and disabled otherwise. `rerank.enabled` is an optional
override (config; [td-001](../techniques/td-001-provider-model.md)): `false`
forces it off even with a credential present; an explicit `true` without a
credential MUST fall back (fail-open) and SHOULD warn.

When active, after candidate generation/fusion and **before** truncation to `k`:

- the top `rerank.candidate_pool` (default 50) fused candidates are re-scored by
  the configured rerank provider (td-001) using the query text and each
  candidate's `snippet`;
- those candidates are reordered by the provider's relevance score; when
  `rerank.candidate_pool < k`, the remaining (un-reranked) fused candidates MUST
  be appended **after** the reranked ones in their original deterministic fused
  order;
- the combined list is then truncated to `k`.

Rules:

- For `index=both`, reranking is applied **once to the merged candidate pool**
  (after per-index normalization and merge), not per-index.
- **Fail-open:** any provider error falls back to the pre-rerank fused order,
  truncated to `k`. A query MUST NOT fail because reranking failed.
- **No result loss:** reranking MUST NOT reduce the result count below what the
  pre-rerank fused order would return for the same `k`. When
  `rerank.candidate_pool < k`, the un-reranked fused tail is appended (in fused
  order) before truncation, so reranking only reorders and never drops results.
- Reranking only reorders results and MAY overwrite `score` with the provider's
  relevance score; it MUST NOT change the result structure (see *Result structure
  and provenance*) or add/remove fields.
- **Determinism:** ties in rerank score MUST be broken deterministically (e.g. by
  `chunk_id`).

### Result structure and provenance

Each hit includes:

- `chunk_id`, `rel_path`, `rep_type`, `score`, `snippet`
- `span` with one of:
  - `lines` (start_line/end_line)
  - `page` (page)
  - `time` (start_ms/end_ms; on a diarized transcript MAY also carry
    `speaker`/`speaker_label`, [td-003](../techniques/td-003-transcription-translation-subtitles.md))

The full hit schema is normative in
[df-006](../data-formats/df-006-hit-citation.md); the `span` type is
[df-005](../data-formats/df-005-span.md).

**Cross-file de-duplication.** When `dedup.retrieval: true`, search MUST collapse
candidate hits whose source documents belong to the same duplicate group
([bs-002](bs-002-ingestion-pipeline.md)) to a **single** hit — the best-ranked survivor —
keeping the canonical document's `rel_path` in the surviving hit. This applies
whether or not ingest-time canonicalization (bs-002) is enabled, so a corpus
indexed before dedup was turned on still de-duplicates at query time.

- **Ordering.** De-duplication runs after candidate generation/fusion and
  **before** reranking (*Optional reranking*) and truncation to `k`, so the
  *candidate pool* shrinks, not the rerank output. This preserves the
  no-result-loss guarantee, which is defined relative to the (now de-duplicated)
  candidate pool: reranking still only reorders and never drops results. Because
  dedup reduces the pool, a query MAY legitimately return fewer than `k` hits when
  the corpus does not contain `k` distinct (non-duplicate) results.
- **Determinism & order preservation.** Collapsing MUST keep the first (best
  pre-rerank) survivor per group and preserve the relative order of survivors.
- **Citations.** Citations (*Citation formatting*) reference the surviving
  (canonical) `rel_path`, so an answer never cites two byte-identical sources for
  the same fact.
- **Default off.** When `dedup.retrieval` is false (default), search returns the
  pre-dedup candidate set exactly as before.

### Citation formatting (human-readable)

Within answers, citations MUST be rendered as:

- code/text: `[path:L<start>-L<end>]`
- pdf OCR: `[path#p=<page>]`
- pdf structured (region): render the primary page (`bbox.page`) as
  `[path#p=<page>]`; when the span covers multiple pages
  (`start_page != end_page`) render the range `[path#p=<start_page>-<end_page>]`.
  Optionally suffix with the section breadcrumb when present, e.g.
  `[report.pdf#p=3 › Results › 3.1 Revenue]`
- transcript: `[path@t=<start>-<end>]` where `<start>/<end>` are `mm:ss` or `ms`.
  `<start>`/`<end>` MAY be word-snapped when the transcript carries per-word timing
  ([td-003](../techniques/td-003-transcription-translation-subtitles.md)); the citation **syntax
  is unchanged** and a consumer that ignores word timing renders the
  segment-level bounds identically. On a diarized transcript (td-003) the speaker
  MAY be appended, e.g. `[interview.mp4@t=02:13-02:41 › S2]`; the base form is
  used when no speaker is present.

### RAG generation

If enabled:

- build a prompt with:
  - system prompt
  - question
  - retrieved contexts + citations
- return answer text + citations list + underlying hits (structured output)

If disabled or `mode=search_only`:

- return hits only.

### Per-language retrieval filter (optional)

`dir2mcp_search` and `dir2mcp_ask` (tool definitions;
[bs-007](bs-007-tool-specifications.md)) MAY accept an **optional** `languages` filter that
restricts results to representations recorded in one or more languages (the
representation `language` written at ingestion;
[df-003](../data-formats/df-003-sqlite-schema.md) `representations`,
[td-001](../techniques/td-001-provider-model.md)). The filter is **additive and
off by default**: absent or empty ⇒ **no language filtering** and search/ask
behave exactly as today (unchanged results).

- **Argument shape.** `languages` is an array of BCP-47 language tags (e.g.
  `["en"]`, `["pt-BR", "es"]`). An empty array is equivalent to omitting it (no
  filter). The argument is OPTIONAL; existing callers that never send it observe
  no behavior change. An OPTIONAL companion argument `language_match` selects the
  matching mode for the whole array: `"primary"` (the DEFAULT — primary-subtag
  matching, below) or `"strict"` (opt-in region/script narrowing, below). Absent
  or empty ⇒ `"primary"`; existing callers that never send it observe no behavior
  change. An unrecognized `language_match` value is `INVALID_FIELD`
  ([df-008](../data-formats/df-008-error-taxonomy.md)).
- **Matching semantics (default — `language_match: "primary"`).** A hit matches
  when its source representation's recorded `language` (df-003) matches **any**
  requested tag (logical OR across the array). Matching is performed on the
  **BCP-47 primary subtag**, **case-insensitively**: a request for `en` matches a
  representation recorded as `en`, `EN`, or `en-US`, and a request for `pt-BR`
  matches `pt` (primary-subtag match). Region, script, and other subtags MUST NOT
  cause a match to be missed when the primary subtags agree. Implementations MAY
  additionally honor an exact full-tag match but MUST AT LEAST honor
  primary-subtag matching. This is the DEFAULT and is unchanged from prior
  versions; callers that omit `language_match` (or send `"primary"`) observe
  exactly this behavior.
- **Region/script narrowing (opt-in — `language_match: "strict"`).** When the
  caller sets `language_match` to `"strict"`, matching uses **BCP-47 Basic
  Filtering** (RFC 4647 §3.3.1) instead of primary-subtag matching: a requested
  tag matches a recorded `language` **iff** the recorded value equals the
  requested tag or extends it with additional subtags (the recorded tag begins
  with the requested tag followed by a `-` separator), compared
  **case-insensitively** on canonicalized subtags. Under `"strict"`, region,
  script, and variant subtags in the request DO narrow the match: `pt-BR` matches
  representations recorded as `pt-BR` (and `pt-BR-…`) but **not** bare `pt` or
  `pt-PT`; `zh-Hans` matches `zh-Hans`/`zh-Hans-CN` but **not** `zh-Hant` or bare
  `zh`. A request that carries only a primary subtag (e.g. `pt`) still matches
  that primary subtag and all its region/script extensions (`pt`, `pt-BR`,
  `pt-PT`), so `"strict"` narrows **only** to the precision the caller actually
  supplies. The default `"primary"` guarantee that region/script MUST NOT cause a
  miss is **unaffected**: narrowing occurs only when the caller explicitly opts in
  via `language_match: "strict"`.
- **Unknown / absent language.** A representation with **no** recorded language
  (unknown; td-001) **never** matches a specific language filter — it is excluded
  whenever `languages` is non-empty. When `languages` is absent/empty, unknown
  representations are **unaffected** (returned exactly as today). Implementations
  MAY offer an explicit opt-in sentinel for unknown (e.g. `"und"`, the BCP-47
  "undetermined" tag) to *include* unknown-language hits alongside a filter; this
  is OPTIONAL and, when unsupported, an unrecognized tag simply matches nothing.
- **Translated representations.** A translated transcript
  ([td-003](../techniques/td-003-transcription-translation-subtitles.md)) is recorded under its
  **target** language (df-003; td-001) and matches that target; its
  `source_language` is not the matched value. Filtering for a language thus
  returns both source-language representations in that language and translations
  *into* that language, which is the intended multilingual-corpus behavior.
- **Pipeline placement & guarantees.** The language filter is applied at
  **candidate selection** (alongside `path_prefix` / `file_glob` / `doc_types`),
  **before** cross-file de-duplication (*Result structure and provenance*),
  reranking (*Optional reranking*), and truncation to `k`. It only **removes**
  non-matching candidates; it MUST NOT reorder, add fields, or change the result
  structure or citation format (*Citation formatting*). As with any selective
  filter, a filtered query MAY return fewer than `k` hits.
- **No match is not an error.** A `languages` filter that excludes every candidate
  returns an empty `hits` list (and, for `ask`, an answer grounded in no contexts
  per *RAG generation*) — never an error. An unrecognized or malformed tag value
  (not a syntactically valid BCP-47 tag) is `INVALID_FIELD`
  ([df-008](../data-formats/df-008-error-taxonomy.md)); a syntactically valid tag
  that simply matches nothing in the corpus is **not** an error.

The filter matches the same recorded representation `language` that ingestion
writes (td-001), so a corpus indexed before any language was recorded simply has
unknown-language representations that no specific filter matches — there is no
migration and no breaking change.

## Changelog

- **0.2.0** — Per-language retrieval filter: added the OPTIONAL `language_match`
  mode (`"primary"` default / `"strict"` opt-in). `"strict"` selects BCP-47 Basic
  Filtering (RFC 4647 §3.3.1) so region/script/variant subtags narrow the match
  (`pt-BR` ≠ `pt`, `zh-Hans` ≠ `zh-Hant`). Additive and backward-compatible: the
  default preserves the primary-subtag "region/script MUST NOT cause a miss"
  guarantee. Syncs SPEC.md §9.5. Unblocks dir2mcp #558.
- **0.1.0** — Migrated from SPEC.md §9 (search routing, reranking, result
  structure/provenance, cross-file de-duplication, citation formatting, RAG
  generation, and the §9.5 per-language retrieval filter). Cross-references
  rewired to stable doc IDs: §5/§5.2 → df-003; §7.9 → bs-002; §8.4/§8.8 →
  td-001; §8.6.2/§8.6.8/§8.6.9 → td-003; §14 → df-008; §15.2/§15.3 → bs-007;
  hit/citation shape → df-006; span → df-005; vector/BM25 candidate generation →
  bs-008. Intra-section references (§9.1.1, §9.2, §9.3, §9.4, §9.5) rewritten as
  named subsection references within this doc.
