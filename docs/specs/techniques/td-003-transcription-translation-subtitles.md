# td-003: Transcription, translation & subtitles

- **ID:** td-003
- **Version:** 0.3.0
- **Status:** Draft
- **Supersedes:** —
- **Superseded-by:** —
- **Source:** SPEC.md §8.6

## Scope

This document defines the normative contract for the media surface that absorbs
the retired `livevtt archive_transcriber` (dir2mcp #251): transcript
representation and timing, source-language detection and optional translation,
subtitle export (VTT/SRT always; TTML/SMIL optional), subtitle sidecar
ingestion, multi-rendition selection, the degenerate-output quality gate,
representation derivation identity / re-derivation, optional speaker
diarization, word-level timing, bilingual broadcast packaging, and the optional
two-phase batch ingest with progress reporting and a resumable run manifest.

**Status: Planned.** Implementation lands in follow-up dir2mcp code PRs once this
spec change is merged. The contract is **domain-general**: it carries **no**
language- or broadcaster-specific behavior (no built-in language list, no default
target language, no station-specific rules). It refines the audio/STT path
(td-004, §7.4.C) and adds translation and subtitle surfaces. All behavior is
deterministic so citations and exports are stable across re-indexing.

The transcript `meta_json` and diarization speaker fields are stored per the
df-003 SQLite schema; the timed provenance coordinate is the df-005 `Span`
(`kind: "time"`).

> Internal `§8.6.x` references below point to the correspondingly numbered
> subsection of this document. External `§N` references have been rewired to
> stable doc IDs (see Changelog).

## Specification (normative)

### 8.6.1 Transcript representation and timing

* A transcript is a `transcript` representation (df-003 §5.2), `index_kind=text`,
  organized into **time-spanned segments** (`time` spans, `start_ms`/`end_ms`,
  df-005).
* **Per-segment timestamps MUST** be stored when the provider returns them.
* **Per-word timestamps MAY** be stored when available, in the segment span's
  `extra_json` as a `words` array of `{t, d, w}` (`t` = start ms, `d` = duration
  ms, `w` = word). Word timing is **metadata only**: it MUST NOT create extra
  chunks and MUST NOT change the chunk `text`. Provider-response normalization
  into this shape and the optional surfacing of word granularity in spans and
  citations are defined in §8.6.9; word-level timing is always optional and
  graceful-degrade.
* **No-timestamp fallback.** When a provider returns no timing, the transcript
  falls back to text-size chunking (td-004 §7.4.C) and the segments MUST be
  flagged `timing: "estimated"` (in `meta_json` and/or span `extra_json`), so
  consumers know the spans are not provider-authoritative.
* **Deterministic windowing.** Segment/window boundaries MUST be deterministic so
  `time`-span citations are stable across re-indexing (consistent with td-002
  windowing).
* **Transcript chunk window.** A **transcript segment** is the unit the
  transcript arrives in: one breath group of about eight seconds from an STT
  provider, or one authored cue from a subtitle sidecar (§8.6.4). Either is too
  fine a retrieval unit for a spoken recording: a sentence splits across two
  chunks, and a retrieved set of ten chunks covers eighty seconds of a three-hour
  recording. An implementation **MUST** therefore merge consecutive transcript
  segments into **chunk windows** under three rules, two of them
  operator-configurable (`media.transcript_chunk_sec`,
  `media.transcript_chunk_gap_sec`, bs-011):
  * a window closes when adding the next segment would make it longer than
    `transcript_chunk_sec`;
  * a window closes when the silence before the next segment is longer than
    `transcript_chunk_gap_sec`, so a window does not span a pause;
  * a window closes at a **speaker change** (§8.6.8), which is already a chunk
    boundary. A window that crossed one would attribute a chunk to a speaker who
    did not say half of it.

  The window's span keeps the **first segment's `start_ms` and the last
  segment's `end_ms`**, and the window's `text` is the member segments' text
  joined by a single space. This is the transcript **chunk** unit: it is the unit
  a `time`-span citation names and the unit retrieval scores. Merging is
  deterministic given the same segments and the same values, so the determinism
  rule above still holds.

  `media.transcript_chunk_sec: 0` **disables** merging and restores one chunk per
  transcript segment. Word timing (§8.6.9) is unaffected: merging removes chunk
  boundaries and concatenates text, so the `words` array of a merged window is
  the concatenation of its members' arrays, and the §8.6.9 rule that word timing
  MUST NOT add chunks or change text is not weakened.

  **Subtitle export MUST NOT inherit the window** (§8.6.3). A forty-second cue is
  unreadable, and re-cutting a sidecar's authored cues would destroy editorial
  work that a round-trip through the implementation has no right to touch. A
  merged chunk therefore **MUST record the boundaries of the segments it merged**
  (each member's start, duration and text length, in order), and export cuts the
  merged text back into exactly those segments. A chunk with no such record, or
  one whose record does not describe its text, is exported whole: an ugly cue is
  recoverable, text cut at the wrong offset is not.

  **Upgrade note.** An implementation that previously emitted one chunk per
  segment produces coarser chunks and different `time`-span citation boundaries
  for already-indexed media after its next re-index. Subtitle output does not
  change. The text and the timing of the underlying speech do not change either,
  only the boundaries the chunks are cut at. Operators who need the old spans
  MUST pin `transcript_chunk_sec: 0`.

### 8.6.2 Language: detection and optional translation

* **Source language is AUTO-DETECTED by default**; an operator MAY pin it
  (`media.language` / per-provider `stt_language`, bs-011 §16.2).
* **Translation is OPT-IN and off by default** (`media.translate.enabled: false`).
* **Target language(s)** are configurable (`media.translate.target_langs`) with
  **NO default**. Enabling translation with an **empty** target list is
  `CONFIG_INVALID`.
* **Transcripts are keyed per language.** A transcript representation is
  identified per language using a **`TranscriptLangSuffix`** convention (the
  source-language transcript and each translated transcript are distinct
  representations of the same document). A translated transcript MUST record its
  `source_language` plus the **translation provider/model** that produced it
  (df-003 §5.2; §8.6.7).

### 8.6.3 Subtitle export

* **VTT and SRT MUST always be available** for any transcribed media: they are
  **derived from the transcript segment spans** (no re-transcription required).
  "Segment" here means the **transcript segment** (an STT breath group, or a
  sidecar's authored cue), not the merged retrieval chunk window of §8.6.1. A cue
  must stay readable on screen and an authored cue must survive a round trip, so
  subtitle export is **not** affected by `media.transcript_chunk_sec`: a merged
  chunk is cut back at the segment boundaries recorded on it.
* **TTML and SMIL are OPTIONAL and off by default**
  (`media.subtitles.ttml.enabled: false`). Producing them MAY require additional
  codec/track metadata (e.g. via `ffprobe`); when that metadata is absent the
  export MUST **fail open** (omit TTML/SMIL, do not fail the request). The
  **bilingual** TTML/SMIL packaging contract (cross-language cue alignment, SMIL
  track metadata) is defined in §8.6.10.
* The **exported language is selectable** (any language for which a transcript
  exists, §8.6.2). Requesting an export for a language with no transcript is
  `INVALID_FIELD`.

### 8.6.4 Sidecar ingestion

* A subtitle **sidecar** (`.vtt`, `.srt`, `.ttml`) sitting next to a media file
  MUST be ingested **as the transcript** for that media **instead of** running STT
  — an authored transcript is authoritative over a machine transcription.
* Sidecar ingestion is **mtime-gated** (bs-002 §7.6): a sidecar newer than the
  cached transcript triggers re-ingest; `--force` overrides the gate.
* **Multiple sidecars** for one media file (e.g. `clip.en.vtt`, `clip.fr.vtt`)
  produce **per-language transcripts** (§8.6.2 keying).

### 8.6.5 Variant / multi-rendition selection

* When a corpus contains multiple **renditions of the same media** (e.g. several
  bitrates/resolutions of one recording), they MUST be **grouped by normalized
  name** (`media.variants.group: true`).
* The pipeline transcribes the **canonical/best** rendition **once**
  (`media.variants.select: best`), **deterministically**, and MUST NOT duplicate
  chunks or embeddings across renditions of the same logical media.
* This is the media-specific special case of **cross-file canonicalization**
  (bs-002 §7.9); `media.variants` and `dedup` share the `best|first` selection
  vocabulary.

### 8.6.6 Output quality gates

* STT, OCR, and translation output MUST pass **degenerate-output checks** before
  being indexed. Minimum checks:
  * **empty** output;
  * **repetition / looping** (the classic STT failure mode);
  * **low density vs. duration** (far too little text for the media length).
  * Implementations **SHOULD** additionally flag a **detected language ≠ pinned
    language** mismatch.
* A failed gate is a **non-fatal per-document error** (bs-002 §7.7): the document
  is marked `status=error` with the appropriate code — `TRANSCRIBE_FAILED`,
  `OCR_FAILED`, or the new `TRANSLATE_FAILED` (df-008 §14.4) — and indexing
  continues.
* The checks MUST be **deterministic** (the same output is judged the same way
  every run).

### 8.6.7 Representation provenance and re-derivation

* Every **derived** representation (extracted markdown, transcript, translated
  transcript, annotation) MUST record the **provider + model (+ model version)**
  that produced it (df-003 §5.2).
* A representation's **derivation identity** is
  `{capability, provider, model, version, language}`. On load, if the configured
  derivation identity for a capability differs from the one recorded on a
  representation, that representation is **stale** and MUST be **re-derived,
  re-chunked, and re-embedded**. This is the runtime analogue of the
  embed-identity → reindex rule (td-001 §8.1.4), but **scoped to a single
  representation** rather than the whole index.
* **Sidecar-sourced transcripts are NOT model-derived** (§8.6.4): they have no
  STT provider/model derivation identity and MUST NOT be invalidated by an STT
  model change. (A change to the sidecar file itself still re-ingests via the
  mtime gate, §8.6.4.)

### 8.6.8 Speaker diarization (optional)

> **Status: Planned.** This subsection defines an **optional** contract for
> **speaker-attributed transcripts** (dir2mcp #266). It is **OFF by default** and
> **provider-dependent**: speaker attribution requires a **diarization-capable
> STT backend** (e.g. a self-hosted WhisperX / pyannote-backed endpoint, td-001
> §8.5). The contract is **domain-general** — no built-in speaker roster, no
> language- or broadcaster-specific behavior. Implementation lands in a follow-up
> dir2mcp code PR once this spec change is merged.

Diarization attributes each transcript segment to a **speaker**. It refines the
transcript representation (§8.6.1) **without changing chunk `text`** — speaker
attribution is **metadata only**: it never edits, reorders, or re-times
transcript content. An implementation MAY, however, introduce a **chunk boundary
at a speaker change** so that every emitted chunk carries a single `speaker` (the
one-`speaker`-per-span model of df-005 / bs-003 §9.3); this speaker-aligned split
is the only boundary effect diarization may have, and it applies **only when
diarization is active**. A transcript with no speaker attribution MUST chunk
**identically** to the non-diarized path.

* **Off by default; opt-in.** Diarization is enabled via
  `media.diarize.enabled: true` (bs-011 §16.2). When disabled (the default),
  transcripts carry no speaker attribution and behave exactly as today.
* **Provider-dependent (capability-gated).** Diarization requires a
  diarization-capable STT backend. If `media.diarize.enabled: true` but no
  configured STT provider advertises the diarization capability, startup MUST fail
  `CONFIG_INVALID` with remediation (no silent partial behavior). Consistent with
  capability-driven activation, an implementation MAY additionally **auto-enable**
  diarization when the active STT backend advertises the capability *and* the
  operator has not set `media.diarize.enabled: false`; the tri-state opt-out
  (`false`) always forces it off.
* **Storage.** Per-segment speaker attribution is stored on the segment `time`
  span's `extra_json.speaker` (a stable per-transcript identifier, e.g. `"S1"`),
  with an optional human-readable `extra_json.speaker_label` (df-003 §5.4;
  df-005). The transcript representation records `diarized: true`, the
  `diarize_provider`/`diarize_model`, and the distinct `speakers` set in its
  `meta_json` (df-003 §5.2).
* **Stable, deterministic identifiers.** Speaker identifiers MUST be **stable and
  deterministic across re-indexing** of the same media with the same diarization
  identity, so `speaker`-scoped citations and filters are reproducible. Mapping a
  raw diarization label to a friendly name (`speaker_label`) is optional and MUST
  NOT change the underlying `speaker` identifier.
* **Sidecar speakers.** A subtitle sidecar (§8.6.4) that carries voice/speaker
  markup (e.g. WebVTT `<v Speaker>` cues) MAY populate `speaker`/`speaker_label`
  directly; such a transcript is **not** model-derived for diarization (no
  `diarize_provider`/`diarize_model`), mirroring §8.6.7.
* **Derivation identity.** When diarization is active, the diarization
  provider/model is part of the transcript's derivation identity (§8.6.7): a
  change to the diarization backend invalidates and re-derives the transcript like
  any other capability change.
* **Retrieval and citation surface.** Speaker is **additive** at retrieval time:
  * `dir2mcp_search` MAY accept an optional `speaker` filter (the search tool
    surface, SPEC §15.2) that restricts time-spanned transcript hits to segments
    attributed to that speaker; a corpus without diarized transcripts simply
    returns no speaker-filtered hits.
  * A hit `span` of kind `time` MAY surface `speaker`/`speaker_label` (bs-003
    §9.2), and human-readable transcript citations MAY append the speaker, e.g.
    `[interview.mp4@t=02:13-02:41 › S2]` (bs-003 §9.3). The base citation form is
    unchanged when no speaker is present.
* **Export.** Subtitle export (§8.6.3) MAY carry speaker as voice markup when the
  target format supports it (WebVTT `<v>`, TTML voice); formats that cannot
  represent it omit it (fail open, never fail the export).
* **Degenerate output.** Diarization that yields a single speaker for clearly
  multi-speaker audio, or an implausible speaker count, MAY be flagged by the
  output quality gate (§8.6.6) but MUST NOT fail the transcript: a
  diarization-quality concern degrades to a flat (un-attributed) transcript rather
  than `TRANSCRIBE_FAILED`.

### 8.6.9 Word-level timing: capture, normalization, and surfacing

> **Status: Planned.** This subsection refines §8.6.1's per-word timing rule
> (dir2mcp #252) by defining (a) how a provider's word-level response is
> normalized into the `words` array and (b) how word granularity is **optionally
> surfaced** in spans and citations. Word-level timing is **always optional and
> graceful-degrade**: a transcript with only segment timing remains fully
> conformant. Implementation lands in a follow-up dir2mcp code PR.

Per-segment timing is the conformance baseline (§8.6.1); per-word timing is a
finer, **optional** refinement layered on top of it.

* **Granularity is recorded, not assumed.** A transcript declares its finest
  available granularity in `meta_json` via the `words` flag (df-003 §5.2):
  `words: true` iff at least one segment carries a populated `extra_json.words`
  array. Consumers MUST treat absent/`false` as "segment granularity only" and
  degrade gracefully — never error because word timing is missing.
* **Provider normalization (OpenAI-compatible / verbose-JSON).** When an STT
  backend returns word-level timing — e.g. a self-hosted faster-whisper /
  whisper.cpp `/v1/audio/transcriptions` endpoint (td-001 §8.5) responding in the
  OpenAI `verbose_json` shape with a `words` array of `{word, start, end}`
  (seconds) — the implementation MUST normalize it into the §8.6.1 `words` shape
  `{t, d, w}` (`t` = start **ms**, `d` = duration **ms**, `w` = word) on the
  owning segment span's `extra_json`. Seconds-to-ms conversion MUST be
  deterministic (round half-up). A response that carries only segment timing
  normalizes to segment spans with no `words` arrays (the existing path,
  unchanged). This parser is a **sibling** to the existing provider segment
  parser, not a replacement: the Mistral `[mm:ss]` segment path is unaffected.
* **Word arrays do not change chunking.** Reaffirming §8.6.1: `words` is metadata
  on the segment span only. It MUST NOT create extra chunks, MUST NOT alter chunk
  `text`, and MUST NOT change segment boundaries. The chunker behaves identically
  whether or not word timing is present, so a transcript chunks the **same** with
  word timing added or removed (deterministic, citation-stable).
* **Optional word-level span surfacing.** A `time` span (df-005; bs-003 §9.2) MAY
  OPTIONALLY narrow its `start_ms`/`end_ms` to **word boundaries** drawn from the
  segment's `words` array (for tighter highlighting/deep-linking), provided the
  narrowed span stays **within** the owning segment's bounds. When word timing is
  absent, the span uses segment bounds (the default). This narrowing is a
  presentation refinement: it MUST NOT add or drop hits and MUST NOT change which
  chunk a span belongs to. It is consistent with the word-level deep-linking
  already permitted for `dir2mcp_open_media_clip` (the open-media-clip tool
  surface, SPEC §15.11).
* **Citation form is unchanged.** Word-level surfacing reuses the transcript
  citation form `[path@t=<start>-<end>]` (bs-003 §9.3); the only difference is
  that `<start>`/`<end>` MAY be word-snapped. No new citation syntax is
  introduced, and a consumer that ignores word timing renders the segment-level
  citation identically.

### 8.6.10 Bilingual subtitle export (TTML + SMIL)

> **Status: Planned.** This subsection refines §8.6.3's optional TTML/SMIL surface
> (dir2mcp #255) to define **bilingual** packaging precisely. It is **OPTIONAL and
> OFF by default** (`media.subtitles.ttml.enabled: false`), broadcaster-neutral
> (no station-specific rules), and requires the translation surface (§8.6.2) only
> for the bilingual case. Implementation lands in a follow-up dir2mcp code PR.

VTT and SRT are always available and monolingual per export (§8.6.3). TTML and
SMIL are the **optional broadcast-packaging** surface and MAY carry **two
languages in one document**.

* **Off by default.** With `media.subtitles.ttml.enabled: false` (the default),
  no TTML/SMIL is produced and behavior is exactly as in §8.6.3. Enabling the
  surface is purely additive — VTT/SRT remain unaffected.
* **Bilingual TTML.** When enabled with a primary and a secondary language (each
  identified per §8.6.2 keying), TTML export MUST emit, per cue, the primary- and
  secondary-language text aligned to the **same** time region, with each text run
  language-tagged (`xml:lang`). Both languages MUST map back to the **same
  transcript segment span** (§8.6.1) so a TTML cue is traceable to its
  `start_ms`/`end_ms` and chunk. Monolingual TTML (one configured language) is
  also valid.
* **Cross-language cue alignment.** Source and translated transcripts (§8.6.2) are
  distinct representations whose segment boundaries MAY differ slightly. The
  exporter MUST align the secondary language to the primary segment's time region
  within a **configurable tolerance** (`media.subtitles.ttml.align_tolerance_ms`,
  default `2500`); a secondary segment whose start is within tolerance of a
  primary cue is merged into that cue. Alignment MUST be **deterministic** (same
  inputs → identical cues). A secondary segment with no primary cue within
  tolerance is emitted as its own secondary-only cue rather than dropped.
* **SMIL packaging.** When SMIL is produced it describes the media presentation:
  the media reference plus probed track metadata — container/codec, bitrate, and
  video **width/height** when applicable — and references the companion subtitle
  document(s). Track metadata is obtained via `ffprobe` (reusing `internal/avutil`,
  df-000 §1) and MAY be cached. SMIL is emitted alongside TTML under the same
  enable flag.
* **Fail-open on missing metadata.** Consistent with §8.6.3, when required codec/
  track metadata is absent or `ffprobe` is unavailable, the export MUST **fail
  open**: omit SMIL (and any metadata-dependent TTML attributes) and still emit the
  text-bearing subtitle output, never failing the request.
* **Language selection.** The exported primary/secondary languages are selectable
  among languages for which a transcript exists (§8.6.2). Requesting an export for
  a language with no transcript is `INVALID_FIELD` (§8.6.3). The bilingual case
  requires translation (§8.6.2) to be enabled for the secondary language; if it is
  not, the export degrades to monolingual rather than failing.
* **Speaker markup.** Per §8.6.8, TTML export MAY carry speaker as voice markup
  when present; formats/cues that cannot represent it omit it (fail open).

### 8.6.11 Two-phase batch transcription, progress, and run manifest

> **Status: Planned.** This subsection defines the **batch ergonomics** contract
> for large-archive ingests (dir2mcp #260): an optional two-phase pass split,
> progress reporting, and a resumable run manifest. It is **implementation-agnostic
> but precise**, additive, and changes no per-document representation, chunk, or
> citation. Worker-pool / multi-GPU distribution is explicitly **out of scope**
> (covered separately). Implementation lands in a follow-up dir2mcp code PR.

* **Optional two-phase ingest.** An implementation MAY run media ingest as **two
  ordered passes** over the corpus — a **transcription pass** (STT/sidecar →
  transcript, §8.6.1/§8.6.4) followed by a **derivation pass** (translation §8.6.2
  and subtitle export §8.6.3/§8.6.10). Two-phase mode is **opt-in**; the default
  single-pass per-document pipeline is unchanged. The two-phase split MUST be
  **observably equivalent** to single-pass for the resulting representations,
  chunks, embeddings, and citations — it changes **ordering and reporting only**,
  never output. Either pass MUST be independently **resumable** (a pass picks up
  where it left off using existing identity/cache state, bs-002 §7.6 / §8.6.7), so
  an interrupted transcription pass does not force re-transcription of completed
  assets.
* **Progress semantics.** Progress reporting is **optional and side-channel**: it
  MUST NOT alter representations, chunks, embeddings, citations, ordering of
  results, or error semantics. Progress is reported against a **total unit count**
  established at pass start (e.g. assets, or asset-seconds of media) and is
  **monotonic** within a pass — completed/failed/skipped units only increase. A
  unit resolved from cache (no work performed) counts as **completed** so a resumed
  run reports faithful totals. Progress output is for human/operator consumption
  and is not part of the MCP wire contract.
* **Run manifest (JSONL).** When enabled, a batch run MUST write a **manifest** as
  newline-delimited JSON (one record per asset) for auditability and resume. Each
  record MUST be **self-describing and deterministic** in field set, and MUST
  record at least:
  * **asset identity** — the corpus-relative path (`rel_path`, stable across source
    schemes per bs-002 §7.8) and the resolved `content_hash` (bs-002 §7.6);
  * **outcome** — a terminal `status` (`completed` | `skipped` | `error`), and for
    `error` the canonical code (df-008 §14.4, e.g. `TRANSCRIBE_FAILED` /
    `TRANSLATE_FAILED` / `OCR_FAILED`) so a manifest is a faithful record of bs-002
    §7.7 per-document outcomes;
  * **media duration** (`duration_ms`, when known) and **processing time** for the
    asset;
  * **outputs produced** — the derived representations and any export artifacts
    (e.g. transcript language(s), translated language(s), subtitle formats emitted).
* **Manifest as resume index.** A manifest MAY be consumed by a subsequent run to
  **skip** assets already terminal in a compatible derivation identity (§8.6.7) and
  to re-attempt `error` assets. The manifest is **advisory for resume** — it MUST
  NOT override the authoritative identity/cache and mtime gates (bs-002 §7.6;
  §8.6.4; §8.6.7); when the manifest and the live state disagree, the live state
  wins (the manifest can only avoid redundant work, never suppress required
  re-derivation).
* **Determinism.** Asset processing order within a pass MUST be deterministic so
  manifests and progress are reproducible across runs of an unchanged corpus.

### 8.6.13 Windowed decode and partial-transcript coverage

> §8.6.12 (multi-track audio, `media.stt.tracks`) is **not yet migrated** into
> this document; it lives in SPEC.md only. The numbering here follows SPEC.md so
> the two do not diverge.

A recording longer than a provider will accept in one request is decoded in
**several windows** and the results merged into one transcript (the §8.6.1 timing
rules are unchanged: the merged transcript is still time-spanned segments in
absolute time). Windowing is derived from the media and the provider, not
configured, and a recording that fits in one request is still one request.

Windowing introduces a failure mode a single request does not have: **some
windows decode and some do not**. A 73-minute recording scheduled as eight
windows, of which one decoded, yields a transcript of the first ten minutes.
Merged, that transcript is indistinguishable from a complete one: the document
is `ok`, its chunks are returned by `search` and `ask`, and nothing says the
other 88% of the audio was never transcribed. An editor then reads "not found"
for minute eleven onward and cannot tell it from "not said". That is the silence
the honest-coverage contract (bs-002 §7.7) exists to forbid, and it applies to a
transcript exactly as it applies to an unreadable format.

* **Coverage MUST be recorded.** When a transcript is produced by a decode of
  **two or more** windows, the implementation **MUST** record a `coverage` object
  on the `transcript` representation's `meta_json` (df-003 §5.2):
  * `windows_attempted`: integer ≥ 2, the scheduled windows.
  * `windows_decoded`: integer ≥ 0, those that yielded transcript content.
  * `ranges`: the decoded time ranges as `{start_ms, end_ms}` objects in
    **absolute** recording time. They MUST be **coalesced** (adjacent or
    overlapping windows merge into one range), **non-overlapping**, and in
    **ascending** `start_ms` order, so a fully decoded recording records exactly
    one range and a consumer reads the gaps directly.
  * `decoded_ms`: the summed length of `ranges`.
  * `duration_ms`: the recording's length, when known (0 when the duration
    probe failed).

  A decode that took **one** request records nothing: absence is "no assertion"
  (df-003 §5.2), never "complete". Recording the object for a **fully** decoded
  multi-window transcript is REQUIRED, not optional. `windows_decoded ==
  windows_attempted` with one full-length range is a positive statement of
  completeness, and an implementation that recorded coverage only on failure
  would make absence ambiguous again.

* **Partial-transcript floor.** The decoded fraction is
  `decoded_ms / duration_ms` when `duration_ms > 0`, else
  `windows_decoded / windows_attempted`. When it falls **below**
  `media.stt.min_coverage` (a fraction in `[0,1]`, default `0`, bs-011), the item
  trips the **partial-transcript floor** and the response is governed by
  **`media.stt.on_partial_transcript`** (`warn | skip`, default `warn`), the same
  shape as the SPEC.md §8.2.1 language floor:
  * **`warn`** (default, **fail-open**): persist the transcript and emit a
    warning naming the decoded fraction. The coverage object is recorded either
    way; `warn` adds the operator-visible signal. The §8.6.6 quality gate remains
    the backstop for degenerate output.
  * **`skip`** (strict): **do not persist** the partial transcript. Record the
    item as `status=skipped` with `skip_reason="transcript_partial"` (df-007
    `stats.json`), so the gap surfaces in the `skip_reasons` honest-coverage
    aggregate rather than as a transcript that silently answers "no" for the
    audio it never saw. No transcript representation is produced for that track.

  `min_coverage: 0` (the default) means **the floor never trips**, so the shipped
  default behavior is unchanged except that coverage is now recorded. A partial
  transcript is genuinely useful to some operators; what is not acceptable is a
  partial transcript that does not say so. The floor is the opt-in for operators
  who would rather have a declared gap than a partial answer.

* **The floor is not the quality gate.** §8.6.6 screens the text that WAS
  decoded for degeneracy. This floor measures how much of the recording was
  decoded at all. Text that never existed cannot be screened, so one cannot
  substitute for the other.

* **All windows failed is still an error.** When **no** window decodes, the
  recording produced no transcript and the existing per-document rules apply
  unchanged (bs-002 §7.7): it is a transcription failure recorded as
  `documents.status=error`, not a coverage record. The distinction is the point: a
  failure is retried on the next run, a `skip` is a decision the daemon already
  made.

* **Coverage survives the transcript cache.** An implementation that caches
  transcripts (bs-002 §7.6; §8.6.7) MUST persist the coverage alongside the
  cached text and restore it on a cache hit. A cache hit that dropped the
  coverage would re-index the same partial transcript as a complete one on the
  next run, which is the defect this section exists to prevent. The coverage
  record MUST be written before the cached text it describes is published, and a
  coverage it cannot write MUST leave the text uncached: a cached transcript with
  no coverage claims to be whole. A **single-request** decode of bytes an earlier
  windowed decode cached MUST clear that entry's coverage record rather than
  inherit it.

* **A refusal retires what an earlier run indexed.** `skip` is evaluated on every
  run, not only the first. An operator who indexes a corpus with the floor off and
  then turns it on has transcript representations and chunks already persisted
  from the partial decode. Recording `status=skipped` while those stay live would
  produce the worst of both readings: a document that reports itself not indexed
  and still answers from the audio it never heard. On a refusal the implementation
  therefore MUST retire (tombstone) the refused track's transcript representations
  and their chunks, including any translations derived from that transcript
  (§8.6.2), before recording the skip. Representations from other sources
  (media chunks, recognition, a sidecar) are untouched: the refusal is scoped to
  the transcript it refused.

  The transcript **cache** entry is deliberately NOT purged. It is not a retrieval
  surface, and keeping it makes the refusal cheap and deterministic: the next run
  reads the same text and the same coverage, reaches the same verdict, and does
  not re-decode a recording the operator already declined.

* **A cache entry with no coverage record asserts nothing.** A transcript cached
  before an implementation recorded coverage has no coverage to restore, and
  nothing distinguishes it from a single-request decode. Such an entry records no
  `coverage` (§5.2 absence, "no assertion") and the floor **MUST NOT** be applied
  to it: an unknown fraction is not evidence of a low one, exactly as SPEC.md §8.2.1's
  floor does not apply when language coverage is undeclared. An implementation MAY
  re-decode to obtain a record; it MUST NOT refuse a transcript for lacking one.

## Changelog

- **0.3.0** — added **§8.6.13**: windowed decode and partial-transcript coverage.
  A transcript merged from several decode windows MUST record a `coverage` object
  (`windows_attempted`, `windows_decoded`, coalesced `ranges`, `decoded_ms`,
  `duration_ms`) on its `meta_json`, so a recording that decoded one window of
  eight is no longer indexed as a complete transcript. The optional floor
  (`media.stt.min_coverage`, default `0`; `media.stt.on_partial_transcript`,
  `warn|skip`, default `warn`) drops a transcript below the fraction as
  `skip_reason=transcript_partial`, retiring what an earlier run indexed from it.
  Coverage MUST survive the transcript cache, written before the text it
  describes; a cache entry with no record asserts nothing and the floor does not
  apply to it (dir2mcp #961).

- **0.2.0** — §8.6.1: added the **transcript chunk window**. Consecutive
  transcript segments (an STT breath group or a sidecar's authored cue) merge
  into retrieval chunks under a duration rule (`media.transcript_chunk_sec`,
  default 40 s), a silence rule (`media.transcript_chunk_gap_sec`, default 6 s)
  and a speaker-change rule; `0` restores one chunk per segment. A merged chunk
  records the boundaries it merged, and §8.6.3 export cuts them back, so subtitle
  output is unchanged and an authored sidecar cue survives a round trip
  (dir2mcp #955).

- **0.1.0** — Migrated from SPEC.md §8.6 (subsections 8.6.1–8.6.11), preserving
  every normative requirement verbatim. External cross-references rewired to
  stable doc IDs: §1→df-000, §5/§5.2/§5.4→df-003 (transcript `meta_json` +
  diarization speaker storage) with the `time` `Span` shape pointing to df-005,
  §7/§7.6/§7.7/§7.8/§7.9→bs-002, §7.4(.C)→td-004, §8.1.4/§8.5→td-001,
  §8.1.7→td-002, §9.2/§9.3→bs-003, §14.4→df-008, §16.2→bs-011. Intra-section
  `§8.6.x` references are retained as in-document subsection anchors. Cross-doc
  IDs are written as bare IDs (matching df-003/df-005), since several targets
  (bs-002, td-001, td-002) are not yet migrated.
- **Drift note:** the search and open-media-clip tool surfaces (`§15.2`,
  `§15.11`) are cited inline by tool name; their behavior is specified in
  [bs-007](../behavior/bs-007-tool-specifications.md) and their schemas in df-007.
