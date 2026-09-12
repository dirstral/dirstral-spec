# bs-002: Ingestion pipeline

- **ID:** bs-002
- **Version:** 0.3.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §7 (excl. §7.4 → td-004)

## Scope

The ingestion pipeline: how dir2mcp discovers files under a corpus root, excludes
unsafe content by default, classifies document types, chunks the generated
representations, indexes incrementally, handles per-document errors, reads remote
corpus sources, canonicalizes cross-file duplicates, and the **CorpusFS**
abstraction that the corpus schemes implement.

Per-type **representation generation** (raw-text/extraction/OCR/STT/annotation
rules) is **out of scope here** and is specified in td-004 (§7.4). The metadata
tables this pipeline writes (`documents`, `representations`, `chunks`, `spans`)
are defined in [df-003](../data-formats/df-003-sqlite-schema.md); the at-rest
span shape is [df-005](../data-formats/df-005-span.md); vector-index identity and
deletion/tombstone semantics are in bs-008.

## Specification (normative)

### 7.1 Discovery

- Recursive walk from the corpus root.
- Default ignore list includes: `.git/`, `node_modules/`, `dist/`, `build/`,
  `.venv/`, `.dir2mcp/`.
- Optional `.gitignore` support.
- Symlink policy:
  - default: do **not** follow symlinks
  - if enabled: follow **only** if the target resolves under the root

### 7.2 Safety exclusions (default)

- Exclude obvious secrets/credentials patterns (regexes applied to file
  **contents**):
  - AWS Access Key ID: `AKIA[0-9A-Z]{16}`
  - AWS/Secret assignment heuristic: `(?i)(?:aws(?:[_\s.]{0,20})?secret(?:[_\s.]*(?:access[_\s.]*)?key)?|secret[_\s.]*access[_\s.]*key)\s*[:=]\s*[0-9A-Za-z/+=]{20,}`
  - JWTs: `(?i)(?:authorization\s*[:=]\s*bearer\s+|(?:access|id|refresh)_token\s*[:=]\s*)[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}` (context-anchored)
  - Generic bearer token: `(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}`
  - Common API key formats (e.g. `sk_[a-z0-9]{32}`, `api_[A-Za-z0-9]{32}`)

  These patterns are the **defaults**; they live in configuration under
  `security.secret_patterns` (bs-011) and can be extended or overridden by users.

  Expected false positives and tuning notes (each note maps to its default rule):

  - **AWS Access Key ID** (`AKIA[0-9A-Z]{16}`): may match synthetic examples in
    docs/tests or random uppercase identifiers of the same shape.
  - **AWS/Secret assignment heuristic** (`(?i)(?:aws(?:[_\s.]{0,20})?secret(?:[_\s.]*(?:access[_\s.]*)?key)?|secret[_\s.]*access[_\s.]*key)\s*[:=]\s*[0-9A-Za-z/+=]{20,}`):
    reduces prose false positives (for example "AWS Secrets Manager") by
    requiring assignment-like context.
  - **JWTs** (`(?i)(?:authorization\s*[:=]\s*bearer\s+|(?:access|id|refresh)_token\s*[:=]\s*)[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}`):
    reduced false positives via auth/key context and minimum segment lengths; can
    still match synthetic token-like test strings with those contexts.
  - **Generic bearer token** (`(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}`): can match
    innocuous config values named `token` (feature tokens, cache tokens) that are
    not credentials.
  - **Common API key formats** (`sk_[a-z0-9]{32}`, `api_[A-Za-z0-9]{32}`): can
    match placeholders, test fixtures, or generated IDs that happen to share the
    prefix/length.

  Refinement guidance via `security.secret_patterns`:

  - Tighten permissive rules (for example JWT/bearer) with context anchors such
    as preceding auth headers, key names, or delimiters.
  - Replace broad alternations with environment-specific patterns (known provider
    prefixes, expected lengths/alphabets).
  - Exclude known-safe paths with `security.path_excludes` (for example fixtures,
    snapshots, generated test vectors) instead of weakening global rules.
  - Keep broad defaults as a baseline, then add narrower allowlist/exception
    handling in path rules where operationally safe.

  Example tuning directions:

  - JWT: anchor to `Authorization: Bearer` or token key names and enforce minimum
    segment lengths.
  - Bearer token: constrain key names (`access_token`, `bearer_token`) and reduce
    accidental matches in generic `token=` fields.
  - AWS secret heuristic: keep assignment/credential context anchors and avoid
    broad prose matching.

  Testing approach for pattern updates:

  - Build a small positive/negative corpus per rule (must-hit secret samples and
    must-not-hit benign samples).
  - Run scanner tests in CI on both corpora and assert precision/recall
    thresholds appropriate for your risk posture.
  - Add regression fixtures for every incident-driven rule change (new false
    positive or false negative).
  - Review CI diffs of matched files/lines before merging pattern changes;
    iterate by tightening context anchors or path excludes.

- Exclude large binaries by default:
  - configurable max file size per `doc_type`.

- Path-based exclusions use optional `.gitignore`-style syntax. Users may provide
  additional ignore files or patterns via `security.path_excludes` in config
  (bs-011; a list of glob patterns); the default set includes the same patterns
  used for ingestion (`.git/`, `node_modules/`, `.dir2mcp/`, etc.) plus any
  sensitive filenames detected.

### 7.3 Type classification

Use extension + MIME sniff + binary heuristics to classify:

- `code`: go/rs/py/js/ts/java/c/cpp/…
- `md`/`text`/`data`/`html`
- `pdf`, `image`, `audio`, `video`
- `archive` (zip/tar/tar.gz) — optionally deep-extracts members
- `binary_ignored`

### 7.4 Representation generation

Representation generation (per-type extraction: raw-text routing, PDF/image
extraction and structured/OCR provenance, audio transcription, and on-demand
annotations) is specified in **td-004**.

### 7.5 Chunking defaults

- Global character-based chunking:
  - `max_chars`, `overlap_chars`, `min_chars`
- Code:
  - line-window chunking (`max_lines`, `overlap_lines`)
  - store `lines` spans
- Structured document (docling):
  - section/element-aware: group consecutive elements under the same section
    breadcrumb, then split by size constraints (`max_chars`, `overlap_chars`,
    `min_chars`)
  - keep tables atomic (never split a table across chunks)
  - store `region` spans (page + bbox + section breadcrumb); fall back to `page`
    spans where provenance is missing
- OCR (page-separated):
  - per page, then within page by size constraints
  - store `page` spans
- Transcript:
  - segment by time if available
  - store `time` spans

The structured/OCR/transcript representations these chunkers consume are produced
per td-004 (§7.4); span shapes are [df-005](../data-formats/df-005-span.md) and
their at-rest form is [df-003](../data-formats/df-003-sqlite-schema.md) §5.4.

### 7.6 Incremental indexing

- Document-level:
  - compute `content_hash`; if unchanged and not deleted → **skip** rep
    generation
- Representation-level:
  - compute `rep_hash`; if unchanged → **skip** chunk rebuild
- Chunk-level:
  - compute `text_hash`; if unchanged → **skip** embedding

### 7.7 Per-document error handling

Non-fatal per-doc errors:

- mark `documents.status=error`, record the error
- **continue** indexing

Fatal errors:

- root inaccessible
- cannot write state (disk full, permissions)
- irrecoverable state corruption

**Honest coverage report (normative).** Startup diagnostics and `dir2mcp doctor`
MUST report extraction coverage honestly, extending the existing requirement
that a present-but-broken extractor be visible rather than reported as healthy
(td-004 §B). The report MUST:

- list the **extraction engines the `ingest.extractor` policy makes eligible**
  and, per engine, its availability and (when unavailable) the reason;
- name every **corpus format class present but not covered** by any active
  engine (per the td-004 §B.1 matrix) — e.g. "`.odt`, `.tiff` present, no active
  engine covers them";
- for each uncovered class, name a **remediation** — the engine/config to add
  (e.g. "install docling for `.tiff`; install `pandoc` for `.odt` (#393); or
  set `ingest.on_unsupported: strict` to fail instead of skip").

**Basis of "present" and of the engine list.** Two readings of the report have
proven wrong in practice and are ruled out here:

* A format class is **present** when the durable document record holds at least
  one non-deleted document of an extractable type (pdf/image/document) in that
  class, **whatever `status` the run stamped on it**. An uncovered document
  with no other searchable representation is recorded as `status=skipped`
  (lenient) or `status=error` (strict), and one that keeps another searchable
  representation stays `status=ok` (td-004 §B.2); a report that considers only
  `status=ok` documents therefore cannot see the first group, which is the gap
  it exists to name. (Measured on the reference
  implementation: once #584 recorded the lenient outcome durably, `doctor`
  reported the coverage check healthy with an `.odt` durably skipped and a
  `.tiff` errored in the store.) Before the first scan has recorded the corpus
  the startup report names the formats the record already holds; a gap that
  first scan finds is recorded durably and MUST be named by the next startup
  and by `doctor`, not only by the run that found it.
* The **engine list** covers every engine the `ingest.extractor` policy makes
  eligible (td-004 §B.1), including a capability-activated secondary engine such as
  `pandoc` under `auto`, and not only the primary engine of the cascade. An eligible
  engine that is unavailable is listed with its reason, because its absence is
  what leaves a format class uncovered.

**Transcript coverage in the same report (normative).** The report above names
what could not be READ. A corpus can also be missing what was never HEARD, and
that gap hides for the opposite reason: the document is `ok`, its chunks answer
`search` and `ask`, and the audio that was never decoded is recorded on one
representation's `meta_json` ([td-003](../techniques/td-003-transcription-translation-subtitles.md)
§8.6.13) and nowhere else. Startup diagnostics and `dir2mcp doctor` MUST
therefore also report, over the durable document record:

- the **number of transcripts whose recorded coverage does not state
  completeness** (td-003 §8.6.13);
- the **decoded audio length against the recorded length** those transcripts
  total (summed `decoded_ms` against summed `duration_ms`), so the report states
  how much speech the corpus is missing and not only how many files are
  affected. A transcript whose `duration_ms` is `0` (td-003 §8.6.13: the
  duration probe failed) counts toward the file total, is excluded from the
  length total, and the count of those excluded MUST be reported rather than
  summed as zero: an unknown length reported as no shortfall is the silence this
  section removes;
- a **remediation**, as the extraction report requires — the STT endpoint whose
  windows did not decode, and that a re-index re-decodes them.

**Basis of "partial".** A transcript is partial when the td-003 §8.6.13 `coverage`
object on its representation does not state completeness, **whatever `status`
the document carries**. Completeness is the measured question wherever it can
be asked: with a known `duration_ms` a transcript is complete when its
`decoded_ms` reaches that length, and only when `duration_ms` is `0` do the
window counts decide (`windows_decoded == windows_attempted`). The two normally
agree, because the scheduled windows tile the recording. Where they disagree the
measured answer is the one to report: every window came back and the decoded
time still falls short of the recording, which is a gap the window counts cannot
see and therefore the gap most worth naming. Under `media.stt.on_partial_transcript: skip` an item left with no
other searchable representation is recorded `status=skipped` with
`skip_reason=transcript_partial`, and that path already aggregates through
`skip_reasons`. Under `warn`, which is the **default**, the document stays
`status=ok` and its partial transcript is indexed. The `warn` path is the one
this report exists for: nothing aggregates it today, so a corpus can hold hours
of undecoded speech behind documents that every other report calls healthy.

A transcript whose record states completeness is NOT partial.
Neither is a single-request decode, which records no `coverage` at all, nor a
cache entry predating the coverage record (td-003 §8.6.13). Absence is "no
assertion" ([df-003](../data-formats/df-003-sqlite-schema.md) §5.2), and MUST
NOT be counted as either complete or partial; td-003 §8.6.13 requires the object
on every multi-window decode precisely so that absence means "not a windowed
decode" rather than "complete enough".

When the record holds no partial transcript the report MUST say so positively
rather than omit the line. An omitted line and a clean corpus read identically,
and the operator this report serves is deciding whether to trust a search result
over an archive.

The closing rule below ("MUST never be reported as an indexed document") governs
a document left with **no** searchable representation. A partial transcript is a
different case and `warn` indexes it deliberately: ten decoded minutes of a
73-minute recording are worth having, and the operator asked for them. What MUST
never be silent is the **shortfall**, which is what this report states.

Under `ingest.on_unsupported: lenient` the uncovered classes are warnings, and a
document left with no searchable representation is recorded as a durable
`status=skipped` (unsupported-format `skip_reason`) so the gap survives the run
that produced it (td-004 §B.2); under `strict` the affected documents are recorded
as `status=error` (td-004 §B.2). A coverage gap MUST never be silent, and MUST
never be reported as an indexed document.

### 7.8 Remote corpus sources

The corpus root MAY live on a remote source. `source.kind` (bs-011) selects the
scheme:

- `local` (default) — a local filesystem path.
- `nfs` — a mounted network filesystem path.
- `s3` — objects under an S3 bucket + prefix (`source.s3.bucket`,
  `source.s3.prefix`, plus region/endpoint; credentials per bs-011, never
  persisted).

**Enumeration.** `local` and `nfs` are walked as filesystems and obey the same
discovery, symlink, and ignore rules as §7.1 (they are ordinary directory trees).
`s3` enumerates objects under `bucket`/`prefix` (a flat object listing, not a
filesystem walk).

**Stable `rel_path` across schemes.** `rel_path`
([df-003](../data-formats/df-003-sqlite-schema.md) §5.1) is defined relative to
the corpus root for every scheme: for `local`/`nfs` it is the path under the root
directory; for `s3` it is the **object key minus the configured prefix**. The
normalization MUST be chosen so that the *same logical corpus* yields the *same*
`rel_path` set under any scheme — a corpus may be relocated `local ⇄ nfs ⇄ s3`
**without changing its document identity** (and therefore without a forced
reindex on relocation alone). Traversal / root-escape protections (bs-009) apply
to **every** scheme: an object key or path that resolves outside the configured
root/prefix MUST be rejected (`PATH_OUTSIDE_ROOT`).

**Change-detection identity.** Incremental indexing (§7.6) keys off a cheap
signal first, then confirms with `content_hash`:

- `local` / `nfs`: the cheap pre-check is `(size, mtime)`; on a change,
  `content_hash` over the file body **confirms** before re-ingest.
- `s3`: the cheap signal is the object **ETag** (alongside `size` and
  `last_modified`). The ETag MUST NOT be treated as a content hash: multipart and
  SSE-KMS ETags are **not** MD5 of the body. `content_hash` therefore still
  requires **reading the object body**; the ETag only decides *whether* a re-read
  is warranted.

**Deletions.** A source object/file that is no longer present at enumeration is a
deletion → it is **tombstoned** (`deleted=1`,
[df-003](../data-formats/df-003-sqlite-schema.md) §5.1), exactly as for a removed
local file.

**State stays local.** Regardless of `source.kind`, the **state directory**
(SQLite metadata, the embedded index, and caches) is always **local**
([df-000](../data-formats/df-000-base.md)): dir2mcp never writes its index/state
back to the remote source. Only the corpus *content* is remote.

### 7.9 Cross-file canonicalization (optional)

Real corpora contain **duplicates**: the same logical content present at multiple
paths (mirrored directories, the same file copied across folders) or in
byte-identical copies. Indexing every copy bloats the index and returns the same
content multiple times for one query, degrading answer quality. Cross-file
canonicalization collapses duplicates to a single **canonical** document while
keeping the others discoverable as **aliases**. It is **optional and off by
default**; when disabled, behavior is exactly as before (every file is indexed
independently).

**Duplicate grouping (exact).** When `dedup.exact: true`, documents that share an
identical `content_hash` (§7.6) form a **duplicate group**. Grouping is by content
identity, not by name — it therefore also collapses the same bytes stored under
different paths.

**Canonical selection.** The pipeline selects exactly one canonical document per
group **deterministically**, using the same policy vocabulary as media variant
selection (td-003):

- `dedup.select: best` (default) — prefer the **richest/largest** rendition:
  highest detected resolution (when applicable), then largest `size_bytes`, then
  the lexically-lowest `rel_path`.
- `dedup.select: first` — the lexically-lowest `rel_path`.

The choice MUST NOT depend on enumeration order beyond the stated tiebreaks, so
re-runs over an unchanged corpus are stable.

**Canonical vs alias behavior.** The pipeline generates representations, chunks,
and embeddings **only for the canonical** document. Non-canonical members are
recorded as **aliases** ([df-003](../data-formats/df-003-sqlite-schema.md) §5.1
`is_alias`/`canonical_doc_id`): they remain discoverable (`list_files`) and
resolvable (`open_file` returns their own byte-identical content), are
**tombstoned** on removal exactly like any document, but contribute **no** chunks
or embeddings and therefore **no** retrieval hits.

**Canonical removal.** When the canonical document of a group is removed
(tombstoned, [df-003](../data-formats/df-003-sqlite-schema.md) §5.1), an alias of
that group MUST be **promoted** to canonical and (re-)indexed deterministically by
the same selection policy, so the group's content does not silently disappear
from retrieval.

**Relationship to media variants (td-003).** Variant/multi-rendition selection is
the **media-specific special case** of this rule: it groups by *normalized name*
and selects the best rendition. `media.variants` and `dedup` share the
`best|first` canonical-selection vocabulary. When both are configured, variant
selection applies first (within a logical media's renditions) and cross-file
dedup then applies across the remaining distinct-content documents.

**Near-duplicates (non-normative, future).** Re-encodes and
same-document-in-another-format (e.g. PDF + DOCX) have **different bytes** and are
therefore *not* collapsed by exact grouping. Similarity-based near-duplicate
detection (e.g. embedding-centroid or MinHash) is **out of scope** for this
version and, if added later, MUST remain opt-in and additive on top of the alias
machinery defined here.

**Retrieval-time de-duplication.** See bs-003: a query MUST NOT return multiple
hits whose source documents belong to the same duplicate group.

### 7.10 CorpusFS — corpus filesystem abstraction

> **Status: Planned.** This subsection formalizes the **logical contract** that
> the §7.8 corpus schemes (`local`, `nfs`, `s3`) implement, so discovery and media
> byte-reads work against any backing store without callers caring which one is in
> use. It is **domain-general** and **implementation-agnostic**: it names
> *capabilities*, not Go types or wire calls. Implementation lands in a follow-up
> dir2mcp code PR (dir2mcp #242) once this spec change is merged.

§7.8 defines *which* corpus locations exist; **CorpusFS** defines the small,
backend-neutral surface every such location MUST present. A conforming corpus
source is anything that can satisfy the three capabilities below; the §7.8 schemes
are the reference bindings (`local`/`nfs` ⇒ filesystem, `s3` ⇒ object store), and
adding a new backing store is adding a new CorpusFS binding, **not** a change to
any caller.

**Capabilities (normative).** A CorpusFS MUST provide exactly these three:

- **list** — enumerate the documents under the corpus root. Each entry MUST carry
  enough metadata to drive incremental indexing (§7.6, §7.8) without opening the
  body: a `rel_path` (§7.8 stable-`rel_path` rule), a `size`, a modification
  signal, and the backend's **cheap change signal** — `(size, mtime)` for
  `local`/`nfs`, the object **ETag** (plus `size`/`last_modified`) for `s3` (§7.8).
  Enumeration obeys the discovery, symlink, and ignore rules of §7.1 for
  filesystem schemes and the flat object-listing model for `s3` (§7.8).
- **stat** — return the same metadata as a `list` entry for a single `rel_path`,
  so a caller can refresh one document's change signal without a full
  re-enumeration. `stat` of a missing `rel_path` MUST be distinguishable from an
  error (it drives the deletion → tombstone path, §7.8).
- **open / range-read** — open a document's bytes for reading and support
  **random-access range reads** (read *N* bytes at offset *O*) — not only a
  whole-file stream. Range reads are required so media windowing (td-002), PDF
  per-page extraction, and `dir2mcp_open_media_clip` (bs-007) can fetch only the
  byte ranges they need; on `s3` a range read maps to a ranged `GET`, on
  `local`/`nfs` to a positioned file read. `content_hash` (§7.6) is computed over
  the **bytes returned by open**, identically across backends, so document
  identity is backend-independent (§7.8 relocation invariant).

**Invariants.**

- **Identity is backend-independent.** The `rel_path` set, `content_hash`, and
  therefore document/representation/chunk identity MUST be identical for the same
  logical corpus regardless of which CorpusFS backs it (§7.8). Relocating a corpus
  `local ⇄ nfs ⇄ s3` MUST NOT, by itself, force a reindex.
- **Root/prefix isolation applies to every capability.** A `list`, `stat`, or
  `open` for a `rel_path` (or object key) that resolves outside the configured
  root/prefix MUST be rejected (`PATH_OUTSIDE_ROOT`, bs-009), on every backend.
- **State stays local.** A CorpusFS exposes the corpus **content** only; it is
  never the home of the state directory (SQLite metadata, the embedded index,
  caches), which is always local
  ([df-000](../data-formats/df-000-base.md), §7.8). A CorpusFS is **read-only**
  with respect to dir2mcp — the pipeline never writes corpus content back through
  it.
- **Selection.** The active CorpusFS is chosen by `source.kind` (§7.8, bs-011);
  `local` is the default. No new config surface is introduced by this subsection —
  `source:` (bs-011) already declares the backing store.

## Examples

**Symlink under root (§7.1, follow enabled).** A symlink `docs/latest →
../releases/v3/` is followed because its target resolves under the corpus root; a
symlink pointing to `/etc/passwd` is not, regardless of the follow setting.

**Stable identity across relocation (§7.8).** A corpus at `local`
`/data/kb/policy.pdf` and the same corpus mirrored to `s3` `kb-bucket` under
prefix `corpora/kb/` both yield `rel_path = policy.pdf` and the same
`content_hash` — so moving the corpus `local ⇄ s3` does not force a reindex.

**Exact dedup group (§7.9).** `reports/q1.pdf` and `archive/2025/q1.pdf` share a
`content_hash`. With `dedup.exact: true` they form one group; `dedup.select: best`
keeps the larger/lexically-lower path canonical (chunked + embedded) and records
the other as an alias (discoverable via `list_files`, byte-served by `open_file`,
but contributing no retrieval hits). Removing the canonical promotes the alias and
re-indexes it.

## Changelog

- **0.3.0** (§7.7): the honest-coverage report also reports transcript
  coverage. 0.63.0 of the spec made a partial transcript record its own
  coverage, per representation; nothing summed it. `media.stt.on_partial_transcript`
  defaults to `warn`, which persists and indexes the partial transcript and
  leaves the document `status=ok`, so the default path aggregates nowhere and a
  corpus can report no skips, no errors and 100% indexed while missing hours of
  speech. The report MUST now name the count of incompletely covered
  transcripts, the summed decoded length against the summed recorded length
  (with unknown durations counted as files, excluded from the length, and their
  number reported), and a remediation. "Partial" is defined on the coverage
  record, not on document status. A record with no partial transcript MUST be
  reported positively (dir2mcp #972).
- **0.2.3** (§7.7): pinned the basis of the coverage report. A format class is
  present when the durable record holds a non-deleted extractable document in it
  regardless of `status` (an uncovered document with no other searchable
  representation is recorded as skipped/error, so a `status=ok`-only count cannot
  see it); the engine list covers every engine the policy makes eligible,
  including a capability-activated secondary engine, with its reason when
  unavailable. Clarification, no contract change (dir2mcp #395).
- **0.2.2** — §7.7: a lenient unsupported-format skip that leaves a
  document with no searchable representation is now a durable `status=skipped`
  (not `status=ok`), so the coverage gap survives the producing run (td-004 §B.2,
  dir2mcp #584).
- **0.2.1** — §7.7: reworded the pandoc remediation example to "install `pandoc`"
  now that the T2 engine is a binding, capability-activated engine (td-004 §B.1,
  dir2mcp #393); no contract change.
- **0.2.0** — §7.7: added the honest-coverage-report contract (list active
  extraction engines + availability; name uncovered corpus format classes and
  their remediation; never silent). Ties to td-004 §B.1/§B.2.
- **0.1.0** — Migrated from SPEC.md §7 (§7.1 Discovery, §7.2 Safety exclusions,
  §7.3 Type classification, §7.5 Chunking defaults, §7.6 Incremental indexing,
  §7.7 Per-document error handling, §7.8 Remote corpus sources, §7.9 Cross-file
  canonicalization, §7.10 CorpusFS). §7.4 (representation generation) was carved
  out to **td-004** and is referenced here by a pointer only. Cross-references
  rewired to stable doc IDs: §5/§5.1/§5.4 → df-003; §1.2 → df-000; §15.11
  open_media_clip → bs-007; §8.1.7 media windowing → td-002; §8.6.5 media variant
  selection → td-003; §9.2 retrieval de-dup → bs-003; §16/§16.1.1/§16.2 config →
  bs-011; §17 PATH_OUTSIDE_ROOT/security → bs-009; §6 vector-index identity →
  bs-008. Intra-§7 references (§7.1/§7.6/§7.8) kept as in-document anchors. Span
  shapes link to df-005. No normative requirement was added, dropped, or weakened.
