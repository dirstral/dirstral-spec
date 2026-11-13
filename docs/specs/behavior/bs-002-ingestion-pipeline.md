# bs-002: Ingestion pipeline

- **ID:** bs-002
- **Version:** 0.6.0
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
- **SMIL backups are never indexed.** Discovery MUST skip a file whose name ends
  in `.smil.bak.<YYYYmmdd_HHMMSS>`, with or without a `.<n>` suffix. This is the
  backup name of the SMIL caption patch (td-003 §8.6.14). The rule applies to
  every scan, to watch mode and to every corpus source, whether or not
  write-back is enabled. The test uses the file name only and is
  case-sensitive. The SMIL itself (`<stem>.smil`) is not skipped. SPEC.md §7.1
  holds the full rule.

### 7.2 Safety exclusions (default)

- Exclude obvious secrets/credentials patterns (regexes applied to file
  **contents**):
  - AWS Access Key ID: `AKIA[0-9A-Z]{16}`
  - AWS/Secret assignment heuristic: `(?i)(?:aws(?:[_\s.]{0,20})?secret(?:[_\s.]*(?:access[_\s.]*)?key)?|secret[_\s.]*access[_\s.]*key)\s*[:=]\s*[0-9A-Za-z/+=]{20,}`
  - JWTs: `(?i)(?:authorization\s*[:=]\s*bearer\s+|(?:access|id|refresh)_token\s*[:=]\s*)[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}` (context-anchored)
  - Generic bearer token: `(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}` (broad; the allow list below releases known benign keys)
  - Hyphenated secret keys (`sk-...`, `sk-proj-...`): `(?:^|[^A-Za-z0-9_-])sk-[A-Za-z0-9-]{20,}`
  - Common API key formats: `(?:^|[^A-Za-z0-9_])(?:sk_[a-z0-9]{32}|api_[A-Za-z0-9]{32})`

  These patterns are the **defaults**; they live in configuration under
  `security.secret_patterns` (bs-011) and can be extended or overridden by users.

  **Allowed matches (normative).** `security.secret_allow_patterns` (bs-011)
  lists regular expressions for text that looks like a credential but is not
  one. The default list holds one pattern, for URL query parameters that carry
  no credential (a video-host redirect link, a pagination cursor):

  - `(?i)[?&;](?:redir_token|page_token|next_page_token|pagetoken|nextpagetoken)=[^\s&#;?=:<>"]*`

  The implementation finds every match of every secret pattern in the text. An
  allow pattern can cancel only a match of the **generic bearer-token default**
  (`(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}`); a match of any other secret
  pattern (an AWS key, an `sk-` key, an operator pattern) is never cancelled,
  also when an allow match covers it. A generic bearer-token match is
  **cancelled** when a match of an allow pattern in the same text covers it:
  the allow match starts at or before the start of the secret match and ends at
  or after its end. The text matches the secret set when at least one secret
  match is not cancelled. An allow pattern therefore releases only the text it
  covers: a document that holds a redirect link and a real credential is still
  withheld. The value of a default allow pattern stops at `&`, `#`, `;`, `?`,
  `=`, `:`, whitespace, a quote and an angle bracket, so it never runs into the
  next parameter or covers a second assignment
  (`?page_token=abc?GITHUB_TOKEN=...`). The allow list applies wherever the secret
  patterns apply: source bytes, derived text, and the text a tool returns. It
  has the list semantics of §7.1, and an empty list cancels nothing. Every allow
  pattern is valid RE2.

  Expected false positives and tuning notes (each note maps to its default rule):

  - **AWS Access Key ID** (`AKIA[0-9A-Z]{16}`): may match synthetic examples in
    docs/tests or random uppercase identifiers of the same shape.
  - **AWS/Secret assignment heuristic** (`(?i)(?:aws(?:[_\s.]{0,20})?secret(?:[_\s.]*(?:access[_\s.]*)?key)?|secret[_\s.]*access[_\s.]*key)\s*[:=]\s*[0-9A-Za-z/+=]{20,}`):
    reduces prose false positives (for example "AWS Secrets Manager") by
    requiring assignment-like context.
  - **JWTs** (`(?i)(?:authorization\s*[:=]\s*bearer\s+|(?:access|id|refresh)_token\s*[:=]\s*)[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}`):
    reduced false positives via auth/key context and minimum segment lengths; can
    still match synthetic token-like test strings with those contexts.
  - **Generic bearer token**
    (`(?i)token\s*[:=]\s*[A-Za-z0-9_.-]{20,}`): the rule matches any key
    that ends in `token` with a value of 20 or more key characters, so it
    catches the credential keys of any provider (`GITHUB_TOKEN`,
    `VERCEL_TOKEN`, `DOCKER_TOKEN`, `id_token`). It can match innocuous values
    named `token` (feature tokens, cache tokens, pagination cursors). Release a
    known benign key with `security.secret_allow_patterns`, not by a narrower
    rule: a rule that lists credential key names misses every provider that
    the list does not name. The default allow list releases the redirect and
    pagination parameters (`redir_token`, `page_token`, `next_page_token`,
    `pageToken`, `nextPageToken`) in a URL query string.
  - **Hyphenated secret keys** (`(?:^|[^A-Za-z0-9_-])sk-[A-Za-z0-9-]{20,}`): the
    left boundary requires `sk-` to start a word. Without it, the rule matched
    inside a transliterated URL slug
    (`.../krasnoyarsk-podpisanie-dogovorov-...`, where `sk-` closes a place
    name) and withheld the whole document. It can still match a slug segment
    that is exactly `sk` (for example a URL path `/sk-...` for a Slovak page)
    when 20 or more letters, digits or hyphens follow.
  - **Common API key formats**
    (`(?:^|[^A-Za-z0-9_])(?:sk_[a-z0-9]{32}|api_[A-Za-z0-9]{32})`): the left
    boundary requires the prefix to follow a character that is not a letter,
    a digit or `_`, so a prefix inside a longer identifier (`task_...`,
    `task_sk_...`) does not match. The rule can still match
    placeholders, test fixtures, or generated IDs that happen to share the
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
  - Bearer token: keep the rule broad, and release known benign keys
    (redirect or pagination parameters) with `security.secret_allow_patterns`.
    A rule that lists credential key names misses every provider that the list
    does not name.
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

  **Regression fixtures for the defaults (normative).** An implementation MUST
  test its default set against these samples. `<n chars>` stands for that many
  characters from `[A-Za-z0-9]`.

  - Must hit (each sample matches at least one default):
    - `GITHUB_TOKEN=<40 chars>`
    - `token: <24 chars>`
    - `PASSWORD_RESET_TOKEN=<24 chars>`
    - `VERCEL_TOKEN=<24 chars>`
    - `id_token: <30 chars>`
    - `OPENAI_API_KEY=sk-proj-<40 chars>`
    - `https://www.youtube.com/redirect?event=video_description&redir_token=<24 chars> GITHUB_TOKEN=<40 chars>`
      (a redirect link and a credential in one text: the allow match covers
      only the first)
    - `https://example.org/list?page_token=abc;GITHUB_TOKEN=<40 chars>` (the
      allow value stops at `;`)
    - `https://example.org/list?page_token=abc?GITHUB_TOKEN=<40 chars>` and
      `https://example.org/list?page_token=abc,GITHUB_TOKEN:<40 chars>` (the
      allow value stops at `?`, `=` and `:`)
    - `https://example.org/list?page_token=AKIA<16 chars from [0-9A-Z]>` (an
      allow match never cancels an AWS key match)
  - Must not hit (with the default allow list, no secret match is left
    uncancelled):
    - `https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbkJ3T2xYbE5fZ3lQdmZ4&q=https%3A%2F%2Fexample.org`
      (a redirect link pasted into a document)
    - `https://example.org/news/krasnoyarsk-podpisanie-dogovorov-o-sotrudnichestve`
      (a transliterated URL slug)
    - `https://www.googleapis.com/youtube/v3/search?part=snippet&pageToken=CAUQAQnextPageCursorAAA1`
      (a pagination cursor)

  **A change of the effective pattern set (normative).** A document withheld as
  `secret_excluded` (bs-007) was judged against one pattern set. The effective
  set changes when a spec version changes a default (0.93.0 did) or when an
  operator edits `security.secret_patterns` or `security.secret_allow_patterns`. On the first scan after such a
  change, the implementation MUST evaluate each withheld document again against
  the new set, also when its `content_hash` is unchanged (§7.6). A document that
  the new set does not match is no longer withheld for a secret: the scan
  treats it as a new document, so every other §7.1 gate (path rules,
  `.gitignore`, size and type limits) still applies, and only a document that
  passes them is indexed. To
  detect the change, an implementation SHOULD persist a fingerprint of the
  effective set (for example a hash of the ordered secret and allow lists) and compare it
  at each scan. A document already indexed that the new set matches MUST be
  withheld, and its representations and chunks retired, so a credential that a
  new pattern finds does not stay searchable. That direction reads every indexed
  document once, so an implementation MAY run it as a background pass after the
  scan. Until that pass has evaluated a document against the new set, no tool
  returns any part of that document: `search` leaves out its hits, `ask` leaves
  out its passages before it builds the prompt (so they never reach a
  generation provider), and `open_file` returns `FORBIDDEN`. Screening only the
  excerpt is not enough, because a credential can cross a chunk boundary and no
  single excerpt then matches. The pass is needed only when the new set can
  match more than the old one (a new or wider secret pattern, or a removed or
  narrower allow pattern). A change that can only match less (an added allow
  pattern, a narrower secret pattern) needs no pass for indexed documents, so
  it hides nothing. In addition, every tool MUST screen the text it is about to
  return against the effective set, and `ask` MUST do so before it builds the
  prompt.

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

**Video containers and shared extensions.** This is a clarification of the rule above: it
states what "extension + MIME sniff" above means for video. Video containers include
`.mp4 .mov .mkv .m4v .webm .avi .flv .wmv .mpg .mpeg .mxf .vob .3gp .m2ts`. An
implementation SHOULD classify each of these as `video`. Some extensions are
shared by code and media: `.ts` is TypeScript or an MPEG transport stream, and
`.mts` is a TypeScript module or an AVCHD transport stream. The extension
cannot decide such a file, so the implementation MUST decide it by content. A
file is an MPEG transport stream when it holds the sync byte `0x47` at a fixed
interval of 188 bytes, of 192 bytes for the BDAV variant (a 4-byte timestamp
before each 188-byte packet, so the sync byte is at offset 4 of each packet), or
of 204 bytes for the variant with 16 bytes of error correction after each
packet. Such a file is `video`. Other content is `code`. An implementation
SHOULD check several consecutive packets (for example the first five) before it
decides `video`, so one `0x47` byte in a text file does not decide it.

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
    generation, unless the type that the current version assigns to the file
    (§7.3) differs from the stored `doc_type`. A classification change (for
    example a `.ts` stored as `code` that §7.3 now decides is `video`) makes
    the next scan process the document again, also with an unchanged
    `content_hash`.
- Representation-level:
  - compute `rep_hash`; if unchanged → **skip** chunk rebuild
- Chunk-level:
  - compute `text_hash`; if unchanged → **skip** embedding
- Media identity: an audio/video `content_hash` is the whole-file hash under
  `ingest.media_identity: full` (default) or a size + three-slice hash under
  `sampled`, which also defers reading the bytes until a representation needs
  them. The normative rule, its trade-off and the derivation-pass read rule are
  in SPEC.md §7.6.

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
  (e.g. "install docling for `.tiff`; install `pandoc` for `.odt` (#393);
  install LibreOffice and `pandoc` for `.doc`; or
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
  `pandoc` or `libreoffice` under `auto`, and not only the primary engine of the cascade. An eligible
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
  total (summed `coverage.decoded_ms` against summed
  `coverage.duration_ms`, both qualified: a transcript `meta_json` carries its
  own top-level `duration_ms`, which is the media's length and not a decode
  result), so the report states
  how much speech the corpus is missing and not only how many files are
  affected. A transcript whose `coverage.duration_ms` is `0` (td-003 §8.6.13: the
  duration probe failed) counts toward the file total, is excluded from the
  length total, and the count of those excluded MUST be reported rather than
  summed as zero: an unknown length reported as no shortfall is the silence this
  section removes;
- a **remediation**, as the extraction report requires. It MUST name the STT
  **provider and model recorded on the transcript** (td-003 §8.6.7 derivation identity),
  which is what the record holds: a deployment MAY route one provider to several
  endpoints, and the endpoint that served a given window is not recorded, so a
  report MUST NOT claim to name it.

**A remediation MUST name an action that re-decodes.** "Re-index" is not one,
and this is stated because it is the obvious thing to write. td-003 §8.6.13 keys the
transcript cache on the media bytes and the STT derivation identity, and
requires a cache hit to restore the recorded coverage with the text. A repaired
endpoint changes neither key component, so an ordinary re-index restores the
same partial transcript, reports the same shortfall, and decodes nothing. The
report MUST therefore name the implementation's own way of forcing a re-decode;
an implementation that has none MUST say that the affected transcripts cannot be
repaired without clearing their cache entries, rather than name a re-index that
will not repair them.

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
  - A SMIL that the caption patch rewrote is an exception (SPEC.md §8.6.14).
    The patch invalidates the cached signal of that SMIL. The next scan MUST
    confirm it by `content_hash`, also when its `(size, mtime)` is unchanged.
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

- **0.6.0** (§7.2, §7.3, §7.7): mirrors spec 0.93.0. §7.2: a new `security.secret_allow_patterns` list cancels a secret match that an allow match covers; its default releases the redirect and pagination URL parameters (`redir_token`, `page_token`, `next_page_token`, `pageToken`, `nextPageToken`), so a pasted redirect link no longer withholds a document, while the generic bearer-token default stays broad. The `sk-` hyphenated-key default is now listed, and it and the `sk_`/`api_` default get a left boundary (a URL slug such as `...krasnoyarsk-...` no longer matches). New normative regression fixtures (must hit and must not hit). A change of the effective pattern set makes the next scan evaluate each `secret_excluded` document again, and withholds an indexed document that the new set matches. §7.3: a clarification that lists video containers and decides `.ts`/`.mts` by content (an MPEG transport stream is `video`). §7.7: the coverage report names `libreoffice` and the `.doc` remediation.
- **0.5.0** (§7.1, §7.8): mirrors spec 0.90.0: discovery skips the SMIL backups `<smil>.bak.<ts>[.<n>]` of write-back (SPEC.md §7.1, §8.6.14). The SMIL itself is still indexed. A SMIL that the caption patch rewrote is confirmed by `content_hash` on the next scan, also when its `(size, mtime)` is unchanged (§7.8).
- **0.4.0** (§7.6): mirrors spec 0.81.0: cross-reference to the `ingest.media_identity` rule (`full` | `sampled`) in SPEC.md §7.6.
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
