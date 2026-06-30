# bs-008: Vector index backends & embed identity

- **ID:** bs-008
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §6

## Scope

How a corpus's vectors are stored and queried: the selectable index backends
(`index.backend`), the two logical axes (text/code) that every backend exposes,
the external-store addressing rules, and the corpus-lifetime **embed identity**
that binds the index regardless of backend. The chunk-level `index_kind`
(`text | code`), the `chunk_id` ANN label, and the `settings` keys live in
[df-003](../data-formats/df-003-sqlite-schema.md); the zero-infra default and the
index version fence are anchored in
[df-000](../data-formats/df-000-base.md); backend connection parameters and
credential handling live in the configuration doc (bs-011).

## Specification (normative)

The vector index is selected by `index.backend` (config; bs-011). The
**default** backend is embedded and requires no external service (zero-infra
invariant, [df-000](../data-formats/df-000-base.md)); an external store MAY be
selected but is **optional, never required**. Whatever backend is chosen, the two
logical axes and the embed-identity binding below are invariant.

### 6.1 Logical axes (text/code)

Independent of backend, vectors are partitioned into two logical axes:

- **text** axis: embeddings for `index_kind=text` chunks (raw text,
  OCR/extracted markdown, transcripts, annotation_text, and — under multimodal
  media-chunk embedding ([td-002](../techniques/td-002-multimodal-media-chunks.md)) — media chunks).
- **code** axis: embeddings for `index_kind=code` chunks (source code and
  code-like configs).

Dimensions MAY differ between axes; each axis MUST be internally consistent. The
**label / payload key** for every vector MUST be its `chunk_id` (integer;
[df-003](../data-formats/df-003-sqlite-schema.md)), so a query result maps
directly to chunk metadata. In the embedded backends the two axes are the two
on-disk files (`vectors_text.*`, `vectors_code.*`); in an external store they are
two collections/namespaces (§6.3).

### 6.2 Backend tiers (`index.backend`)

| `index.backend` | Tier | Description | External infra | Default |
|---|:--:|---|:--:|:--:|
| `memory` | **A** | In-memory HNSW, **pure-Go**, persisted/snapshotted to the local state dir | none | **✅ default** |
| `disk`   | **B** | Pure-Go on-disk / memmapped single-node index in the local state dir | none | |
| `qdrant` | **C** | External Qdrant collection | required | |
| `pgvector` | **C** | External PostgreSQL + pgvector | required | |

- **Tier A (`memory`, default)** — an in-memory HNSW graph built in pure Go,
  snapshotted to the local state dir (`vectors_text.*` / `vectors_code.*`) so it
  survives restarts. Requires no external service. This is the zero-infra default
  ([df-000](../data-formats/df-000-base.md)).
- **Tier B (`disk`)** — a pure-Go on-disk / memory-mapped index for single-node
  corpora too large to hold fully in RAM. It is single-node (no clustering) and,
  like Tier A, MUST remain buildable with `CGO_ENABLED=0` (§6.5).
- **Tier C (`qdrant` / `pgvector`)** — an external vector store. It is
  **optional and MUST NOT be required**: a conforming deployment runs on Tier A
  with no external infrastructure ([df-000](../data-formats/df-000-base.md);
  SPEC.md §19). Tier C is for operators who already run such a store or who need
  horizontal scale.

### 6.3 External store addressing (Tier C)

- A Tier C backend is addressed by a **collection / namespace derived from
  `corpus_id`** ([df-003](../data-formats/df-003-sqlite-schema.md) `settings`),
  so multiple corpora can share one external store without collision. The two
  axes map to two collections/namespaces (one for text, one for code).
- Connection parameters for Tier C live under `index:` (config; bs-011);
  credentials follow the secret-handling rules (bs-011) and MUST NOT be persisted
  to the snapshot.
- **No silent fallback.** If a configured Tier C backend is **unreachable at
  preflight** (SPEC.md §2.5), startup MUST fail with `CONFIG_INVALID` and
  remediation. An unreachable external store MUST NOT silently downgrade to an
  embedded tier — that would change the corpus's vector home invisibly.

### 6.4 Embed identity binds every backend

The corpus-lifetime **embed identity** — `provider | text_model | code_model |
text_dim | code_dim | multimodal` (td-001) — binds the index **regardless of
backend**. On load, if the configured embed identity differs from the one
recorded for the index (embedded snapshot or external collection metadata), the
server MUST refuse to mix vector spaces: it either errors (`CONFIG_INVALID`) or
triggers a full reindex (td-001). A backend MUST NOT silently serve a collection
built under a different embed identity.

### 6.5 Pure-Go / `CGO_ENABLED=0` (normative)

The embedded backends (Tier A and Tier B) MUST be implementable in **pure Go**
and buildable with **`CGO_ENABLED=0`** — the reference store uses
`modernc.org/sqlite` (a pure-Go SQLite) and a pure-Go ANN implementation
specifically to keep the single-binary, cross-compiled, CGO-free build.

- **`sqlite-vec` is rejected** for the embedded path: it is a C extension and is
  incompatible with the pure-Go `modernc.org/sqlite` driver under
  `CGO_ENABLED=0`. Implementations MUST NOT make `sqlite-vec` (or any other C
  SQLite extension) a requirement of an embedded backend.
- Tier C backends are out-of-process (network clients), so they impose no CGO
  requirement on the dir2mcp binary.

### 6.6 Deletions

- **Embedded backends (Tier A/B)** are treated as **append-only**: deleting
  documents/representations/chunks sets `deleted=1` in SQLite (the tombstone),
  and retrieval uses **oversampling** — ask the index for `k * oversample_factor`
  results, filter out `deleted=1`, return the first `k` remaining. Default
  `oversample_factor`: 5 (configurable).
- **External backends (Tier C)** MAY delete vectors **natively** (e.g. delete by
  `chunk_id` payload) instead of relying solely on oversampling. A Tier C backend
  MUST still **honor the SQLite `deleted=1` tombstone** as the source of truth —
  a vector whose `chunk_id` is tombstoned MUST NOT appear in results even if its
  native deletion has not yet propagated — so retrieval semantics are identical
  across backends.

## Changelog

- **0.1.0** — Migrated from SPEC.md §6 (vector index backends and identity),
  preserving every normative requirement: the embedded-default = zero-infra
  invariant, the Tier A/B/C backend model, the text/code logical axes with
  `chunk_id` as the mandatory vector label, the corpus-lifetime embed identity
  (`provider | text_model | code_model | text_dim | code_dim | multimodal`) and
  its refuse-to-mix / full-reindex rule, the external-store-optional-never-required
  and no-silent-fallback rules, the `CGO_ENABLED=0` / `sqlite-vec`-rejected
  constraints, and the tombstone + oversampling deletion semantics. Cross-refs
  rewired to stable doc IDs: §1.2 → df-000 (zero-infra invariant); §5/§5.5 →
  df-003 (`settings` keys, `chunk_id` ANN label, `index_kind`); §8.1.x embed
  identity/reindex → td-001; §8.1.7 multimodal media chunks → td-002;
  §16/§16.1.1/§16.2 → bs-011 (`index.backend`, `index:` connection params,
  credential handling). Internal §6.x references kept. **Drift note:** SPEC.md
  §2.5 (preflight) and §19 (non-goals) are referenced here but have no migrated
  doc ID yet, so they remain `SPEC.md §…` pointers pending their own migration.
