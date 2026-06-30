# td-005: Distributed embedding (coordinator + workers)

- **ID:** td-005
- **Version:** 0.1.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §8.7

## Scope

The **optional** contract for embedding a corpus with **multiple workers on
separate machines** (e.g. a pool of GPU hosts) instead of the single in-process
embedding loop.

> **Feature status: Planned.** This contract is **off by default** and
> **additive**: a conforming deployment still runs the whole pipeline in **one
> binary on one machine** with no broker ([df-000](../data-formats/df-000-base.md),
> local-first single-binary default). It is **implementation-agnostic** — it names
> a job-queue *contract*, not a specific broker. Implementation lands in follow-up
> dir2mcp code PRs (dir2mcp #248 distributed workers, dir2mcp #249 standalone
> embed-worker mode) once this spec change is merged.

By default, embedding runs **in-process**: the same binary that discovers,
chunks, stores, and serves also embeds pending chunks (the chunk-level
`embedding_status` machinery of [df-003](../data-formats/df-003-sqlite-schema.md)
/ [bs-002](../behavior/bs-002-ingestion-pipeline.md)). The distributed mode
**separates the control plane from the embedding compute** so embedding can scale
across hosts; it changes **where embedding happens**, not **what is persisted** —
the store shape ([df-003](../data-formats/df-003-sqlite-schema.md)), embed identity
([td-001](td-001-provider-model.md)), and retrieval contract
([bs-003](../behavior/bs-003-retrieval-and-answer.md)) are unchanged.

## Specification (normative)

### Roles

* **Coordinator (control plane).** Exactly one logical coordinator per corpus does
  discovery, representation generation, chunking
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)), store ownership
  ([df-003](../data-formats/df-003-sqlite-schema.md)), MCP serving
  ([bs-004](../behavior/bs-004-mcp-transport.md)), and retrieval
  ([bs-003](../behavior/bs-003-retrieval-and-answer.md)). It **enqueues** embedding
  jobs for chunks whose `embedding_status` is `pending`
  ([df-003](../data-formats/df-003-sqlite-schema.md)) and records results written
  back to the store. The coordinator owns the **local** state directory
  ([df-000](../data-formats/df-000-base.md)) — SQLite metadata and, for the
  embedded tiers, the vector index.
* **Embed-worker (compute plane).** Zero or more stateless workers (e.g. on GPU
  hosts) **pull** jobs, read the referenced corpus bytes, call the configured
  **embed** provider ([td-001](td-001-provider-model.md), typically a co-located
  self-hosted endpoint), and **write the resulting vectors and chunk status back to
  the shared store**. A worker does **no** discovery, chunking, MCP serving, or
  retrieval. A standalone worker run mode (dir2mcp #249) is exactly this role with
  no serving responsibilities.

The single-binary default is the **degenerate case** of this contract: one process
plays both roles with an in-process queue and no external broker. Enabling the
distributed mode MUST NOT change results versus the in-process default for the
same corpus and embed identity.

### Job description

An embedding job MUST identify its work precisely enough that any worker can
execute it without coordinator-relayed payload bytes:

* a **corpus reference** — which corpus/`corpus_id`
  ([df-003](../data-formats/df-003-sqlite-schema.md)) and the `source` binding
  needed to read bytes via CorpusFS
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md));
* a **chunk identity** — the `chunk_id`
  ([df-003](../data-formats/df-003-sqlite-schema.md), the ANN label) and the
  `index_kind` (`text|code`, [bs-008](../behavior/bs-008-vector-index.md)) so the
  worker writes to the correct axis;
* a **payload identity** — the chunk's `text_hash`
  ([df-003](../data-formats/df-003-sqlite-schema.md)) for text chunks, or, for a
  media chunk ([td-001](td-001-provider-model.md)), the `rel_path`/media ref plus the
  chunk's span ([df-003](../data-formats/df-003-sqlite-schema.md)
  `page`/`time`/`region`) so the worker can fetch and window the exact media bytes
  via CorpusFS range reads ([bs-002](../behavior/bs-002-ingestion-pipeline.md));
* the **embed identity** ([td-001](td-001-provider-model.md)) the job was enqueued under
  (`provider | text_model | code_model | text_dim | code_dim | multimodal`), so a
  worker can **reject** a job whose embed identity does not match its configured
  provider rather than silently writing vectors from the wrong space.

A worker reads corpus bytes **directly from the source** via CorpusFS
([bs-002](../behavior/bs-002-ingestion-pipeline.md)) — never relayed through the
coordinator — so a remote (`s3`/`nfs`) corpus and a worker pool can share the same
bytes without the coordinator becoming a data-plane bottleneck.

### Idempotency, ordering, and identity

* **Idempotent writes.** A job MUST be safe to execute **more than once**
  (at-least-once delivery is assumed). Writing a vector is keyed by `chunk_id`
  ([bs-008](../behavior/bs-008-vector-index.md)), so a re-delivered or duplicated
  job overwrites the same vector and sets the same terminal `embedding_status` —
  re-running a completed job is a no-op, never a duplicate vector. A worker MUST
  NOT assume exactly-once delivery.
* **No global ordering requirement.** Embedding jobs are **independent**; workers
  MAY drain them in any order and in parallel. Retrieval already operates on a
  partial index ([df-000](../data-formats/df-000-base.md)), so chunks becoming
  searchable in arbitrary order is acceptable. The only ordering constraint is
  causal: a chunk MUST exist in the store (enqueued by the coordinator) before a
  job for it can be claimed.
* **Embed identity is enforced per job.** The embed identity
  ([td-001](td-001-provider-model.md)) is part of the job (see *Job description* above).
  A worker whose configured embed provider/model/dim/multimodal does not match the
  job's embed identity MUST fail the job (returning it for redelivery or
  dead-lettering) rather than write a vector — this preserves the corpus-lifetime
  single-space invariant ([bs-008](../behavior/bs-008-vector-index.md),
  [td-001](td-001-provider-model.md)) across a heterogeneous worker pool.
* **Failure handling.** A job failure is **non-fatal** to the corpus: the chunk's
  `embedding_status` records `error`
  ([df-003](../data-formats/df-003-sqlite-schema.md)) and the job MAY be retried
  (broker redelivery) up to an implementation-defined limit, after which it is
  dead-lettered and surfaced as a per-document/per-chunk error
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)), exactly as an in-process
  embedding failure is today. A stuck/abandoned in-flight job MUST become
  re-claimable (visibility timeout / lease expiry) so a crashed worker does not
  strand a chunk in `pending` forever.
* **Tombstone safety.** A job for a `chunk_id` that has since been tombstoned
  (`deleted=1`, [bs-008](../behavior/bs-008-vector-index.md)) MUST NOT resurrect
  it: the write either is skipped or is harmless because retrieval honors the
  tombstone ([bs-008](../behavior/bs-008-vector-index.md)) regardless of vector
  presence.

### Shared store and broker

* **Shared vector store.** Workers and coordinator MUST write to a **shared**
  vector home. The embedded tiers (Tier A/B,
  [bs-008](../behavior/bs-008-vector-index.md)) are **single-node** and are
  therefore **not** a shared store across machines; a distributed worker pool
  REQUIRES an external store reachable by all participants — a **Tier C** backend
  (`qdrant`/`pgvector`, [bs-008](../behavior/bs-008-vector-index.md)) addressed by
  the `corpus_id`-derived collection/namespace. This is the one configuration where
  Tier C stops being merely optional and becomes a **prerequisite of the
  distributed mode** — the embedded default remains correct for the single-machine
  case ([df-000](../data-formats/df-000-base.md)). Chunk metadata/status
  ([df-003](../data-formats/df-003-sqlite-schema.md)) likewise lives in a store
  reachable by all workers.
* **Broker is implementation-defined.** The transport that carries jobs
  (coordinator → workers) is **not** specified here — any queue/broker providing
  at-least-once delivery, a redelivery/visibility mechanism, and a dead-letter path
  satisfies the *Idempotency, ordering, and identity* requirements above (e.g.
  NATS, Redis, SQS). The in-process default needs no broker. Broker connection
  parameters and credentials follow
  [bs-011](../behavior/bs-011-configuration.md) (resolved from a secret source,
  **never persisted** to the config snapshot), consistent with every other
  provider/store credential.
* **Capability-driven, off by default.** The distributed mode activates only when a
  broker/worker topology is configured; with no such config, the pipeline runs the
  in-process embedding loop unchanged ([df-000](../data-formats/df-000-base.md)).
  The standalone embed-worker run mode (dir2mcp #249) is the worker role packaged
  without serving — it joins the pool, pulls jobs, reads corpus bytes via CorpusFS
  ([bs-002](../behavior/bs-002-ingestion-pipeline.md)), embeds via its configured
  provider ([td-001](td-001-provider-model.md)), and writes back; it never serves MCP or
  runs discovery.

## Changelog

- **0.1.0** — Migrated from SPEC.md §8.7 (verbatim normative content; prose lightly
  tightened, no requirements dropped). Cross-references rewired to stable doc IDs
  per the mapping: §1→df-000, §5→df-003 (including chunk `embedding_status`
  ok/pending/error), §6→bs-008, §8.1–8.5→td-001, §16→bs-011. Refs outside the
  supplied mapping were rewired per established repo convention: §7→bs-002
  (ingestion/discovery/chunking/CorpusFS/per-chunk errors), §9→bs-003 (retrieval),
  §10→bs-004 (MCP serving). Internal back-references (§8.7.2/§8.7.3) became
  in-document section references. The feature's "Planned" status is retained as a
  body note; the document's own lifecycle status is Draft.
  - Drift note: this doc links `bs-011` (configuration, for §16.1.1 broker
    credentials) and `td-001` (providers) as the stable target IDs; those files are
    not yet present in `docs/specs/` at migration time. Link targets, not fixed
    here.
