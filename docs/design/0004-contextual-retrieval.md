# Design 0004 — Contextual retrieval (document-aware per-chunk context)

**Status:** Proposed (targets spec `0.40.0`)
**Author:** dirstral maintainers
**Related:** SPEC §7.4 (chunking), §8.1.4 (embed identity), §8.6.7 (representation
provenance / re-derivation), §9 (retrieval), §16 (configuration); dir2mcp #330,
#395; Design [0001](0001-multi-provider.md) (provider model)

## 1. Summary

Adopt **Contextual Retrieval** (Anthropic, 2024): before a chunk is embedded,
prepend a short, LLM-generated, **document-aware** context string
("From the Q3 2026 earnings call, discussing ad revenue…") so an otherwise
ambiguous chunk carries the context a query needs to match it. The context is
prepended to the text used for **embedding** (and optionally BM25), while the
**displayed and cited** chunk text stays **unchanged** so citations remain
faithful.

Reported impact: ~**35%** fewer top-20 retrieval failures, up to **~67%** combined
with reranking (dir2mcp already has reranking, §8.4). This is one of the
highest-ROI retrieval improvements available and dir2mcp today embeds raw chunks
with **no** contextualization.

The feature is **opt-in, off by default, and domain-general** (no built-in
prompts or corpus assumptions).

## 2. The one correctness invariant

Contextualization **changes the embedded vector** for every chunk. It is therefore
a **corpus-lifetime, reindex-bound** choice exactly like the embed model or
`late_chunking` (§8.1.4): toggling it, or changing the generator/prompt that
produces the context, MUST re-embed the corpus, and a query-time identity mismatch
MUST refuse to mix vector spaces.

**Mechanism:** fold a contextualization token into the **embed identity** (§8.1.4)
as a **9th field**, adjacent to `late_chunking`:

```
provider|base_url|text_model|code_model|text_dim|code_dim|multimodal|late_chunking|contextual
```

`contextual` is `off` when disabled (byte-identical migration: a pre-feature
identity gains `|off`, so no existing corpus spuriously reindexes — the same
backward-compatible append the base_url/multimodal/late_chunking migrations use).
When enabled it encodes the context **generator identity** (provider, model, and a
prompt-template version), so a generator or prompt change re-embeds rather than
silently mixing differently-contextualized vectors.

This is the load-bearing design decision; everything else is mechanism.

## 3. Data model — embed text vs. cited text

The core separation (citation faithfulness, #403):

| Text | Contextualized? | Where it lives |
|------|-----------------|----------------|
| **embed input** (and, if enabled, BM25 input) | **yes** — `context + "\n\n" + chunk` | transient; not persisted as chunk text |
| **persisted / displayed / cited chunk text** (`chunks.text`, snippets, `open_file`) | **no** — the raw chunk, unchanged | `chunks.text` (unchanged from today) |

The context string itself is a **derived, cached** artifact (§8.6.7 machinery),
**not** a new `rep_type` and **not** part of `chunks.text`. It is content-addressed
by the same derivation-identity scheme transcripts/OCR/translation already use, so:

- it is generated **once** per (chunk, generator-identity) and reused across
  re-scans;
- it **re-derives** only when the source chunk content or the context-generator
  identity changes (§8.6.7);
- an operator can inspect/audit the generated context without it polluting the
  cited answer.

Storage options (to settle in the spec PR): a `chunk_context` column on `chunks`
(NULL when disabled) vs. a sidecar cache keyed by chunk content-hash. The column is
simpler for the embed worker; the sidecar keeps `chunks` lean. **Recommendation:** a
nullable `chunk_context` column plus the content-addressed cache for the *generation*
step, so the embed worker reads context locally without a provider round-trip.

## 4. Generation — cost and the prompt-cache trick

Naively, contextualization is one LLM call per chunk — expensive on a large corpus.
Anthropic's key cost lever: **prompt-cache the parent document once**, then issue a
short per-chunk completion that references it, so the dominant (document) tokens are
paid once per document, not once per chunk.

- The context generator is the configured **chat** provider (Design 0001 /
  §8.1.3), reusing the existing `Generator` seam. Providers with prompt caching
  (Anthropic, OpenAI, Gemini) get the cheap path; others still work, just costlier.
- Generation is **bounded** (a tight `max_tokens`, like the transcript-translation
  cap #500) — the context is meant to be one or two sentences.
- Generation is **fail-open per chunk**: if the generator errors for a chunk, that
  chunk embeds **without** context (and records that it did), rather than failing
  ingest. Coverage is honest (§8.2.1-style), not silent.
- The prompt template is **versioned** (folded into the identity, §2) and
  **general** — no domain terms; the operator MAY override it (§6).

## 5. Retrieval & BM25

- **Vector:** the query embeds normally; contextualized chunk vectors match better.
  No query-side change.
- **BM25 (optional):** Anthropic contextualizes the lexical index too. `chunks_fts`
  MAY be built over `context + chunk` when `contextual.bm25: true` — but BM25 text
  is **not** part of the embed identity (it does not change vectors), so toggling it
  rebuilds the FTS index, not the embeddings. Kept a separate switch for that
  reason. Default: contextualize embeddings only (the bigger win).
- **Citations unchanged:** hits still carry the raw chunk snippet + its real span
  (§8.6.1/§5.4). The context never appears in `snippet`, `open_file`, or an answer
  quote.

## 6. Configuration (sketch — normative form in the spec PR)

```yaml
retrieval:
  contextual:
    enabled: false            # opt-in; off by default (the §2 identity stays "…|off")
    provider: ""              # optional: a specific chat provider profile; empty => the configured chat provider
    model: ""                 # optional model override
    max_tokens: 128           # tight cap; context is 1–2 sentences
    bm25: false               # also contextualize the lexical (FTS) index (separate from embeddings)
    prompt_version: v1        # template version; part of the embed identity (§2) — a change re-embeds
    # prompt: ""              # optional operator override of the (general, domain-free) default template
```

Capability-driven (like OCR/STT): with a chat provider present and
`enabled: true`, contextualization activates; with no chat provider it fails open
(disabled + a warning), never a hard error.

## 7. Spec surface for the follow-up spec PR

1. **§8.1.4** — add `contextual` as the 9th embed-identity field + its
   backward-compat append rule (pre-feature identity → `|off`) + the migration note
   (no spurious reindex).
2. **§7.4 / §8.6.1** — define the embed-input transform (`context + chunk`), the
   invariant that `chunks.text` and citations are the **raw** chunk, and the
   fail-open-per-chunk honest-coverage rule.
3. **§8.6.7** — the context is a derived, content-addressed artifact with its own
   derivation identity (generator provider/model/prompt_version); re-derive on
   change.
4. **§16** — the `retrieval.contextual` config block.
5. Optional: surface a `contextualized` boolean on the stats/coverage side so an
   operator can see how many chunks carry context vs. fell open (a later, additive
   stats field — spec-first like watch_overflows / skip_reasons).

No new **tool** and no **served-schema** change: `Hit`/`Citation`/`Span` are
unchanged (the context is never on the wire). This keeps the conformance surface
(df-006/df-007) untouched.

## 8. Phasing

- **P1 (this design → spec PR → code):** embeddings-only contextualization, the
  §2 identity fold, the cache + fail-open generation, the config block. Delivers
  the bulk of the retrieval win.
- **P2:** contextual BM25 (`contextual.bm25`), and the stats coverage field.
- **P3:** operator prompt override + per-doc-type templates (still domain-general —
  templates, not baked terms).

## 9. Alternatives considered

- **Contextualize the *cited* text too** — rejected: breaks citation faithfulness
  (#403); a user would see the LLM's blurb as if it were the source.
- **A new `rep_type` (`contextual_transcript`)** — rejected: it is not a
  representation of the document, only an embed-time transform of a chunk; a
  content-addressed cache + a `chunk_context` column is lighter and avoids a
  rep_type explosion (cf. the §8.6.12 track-suffix concern).
- **Skip the identity fold** — rejected: it is the correctness bug that would let
  contextualized and raw vectors silently mix in one index (a §8.1.4 violation).
- **Hierarchical / parent-doc retrieval (#329)** — complementary, not a substitute;
  contextual retrieval is lower-risk and higher-ROI first.
