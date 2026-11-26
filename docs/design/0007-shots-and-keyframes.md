# Design 0007: Shots and keyframes

**Status:** Normative text in SPEC §8.10 (spec 0.103.0). This note keeps the
motivation, the choices and the measurement.
**Extends:** SPEC §8.9 (recognition), §5 (metadata schema)
**Related:** [Design 0003](0003-multimodal-embeddings.md) (time windows over
video), [Design 0004](0004-recognize-capability.md) (recognition)

## 1. Summary

A video archive is described shot by shot: what each shot shows, what text is
on screen, who appears. Each of those per-shot outputs needs the same two
inputs: the shot boundaries and the frames to read. This design adds them as
**structure**: dir2mcp detects the shots of each video, stores their time
ranges, and derives keyframes from them. Shots produce no chunks. Later
per-shot outputs (captions, on-screen text) become annotations whose `time`
span is the shot's range.

## 2. Why shots are not recognition annotations

A recognition annotation (SPEC §8.9) is a **statement**: it MUST have
non-empty text, and it becomes one searchable chunk. A shot has no statement.
To index "shot 12, 3.2 s" as text would put noise into every search and
`ask`. So shots get their own small table (SPEC §5.7), and the statements
about a shot stay where they belong, in recognition-style annotations aligned
to it.

## 3. Why a built-in detector and a served one

dir2mcp builds with `CGO_ENABLED=0` (SPEC §6.5), so it cannot run a learned
model in its own process. Two detectors follow from that:

* **`ffmpeg`**, built in. It needs only the `ffmpeg` tool, no model and no
  GPU, so every installation can detect shots.
* **`serve`**, an HTTP backend that returns cut times. A learned detector
  runs there, in the same way as every other model that dir2mcp uses
  (transcription, OCR, chat).

The wire returns **cut times**, not shot ranges. Shot ranges invite gaps,
overlaps and zero-length shots that the daemon would have to repair; a set of
cut times has none of these, and SPEC §8.10.4 turns it into a cover of the
video with no gap and no overlap.

## 4. Measurement (2026-11)

**Corpus.** 71 films from a news and documentary archive sample, 50.6 hours,
decoded from 360-pixel renditions. Many programmes use an inset or
split-screen layout: a presenter on one side, footage on the other.

**Reference.** The cuts of TransNetV2, a learned shot detector, run earlier on
the same renditions: 13,709 cuts (271 per hour). The reference is not hand
checked, so the first numbers below measure **agreement**, not accuracy.

**Agreement of the `ffmpeg` scene score with the reference** (a cut matches
when the two are within 0.5 s; the numbers at 0.2 s are within 0.003):

| threshold | cuts | precision | recall | F1 |
|---|---|---|---|---|
| 0.15 | 14,735 | 0.896 | 0.963 | 0.928 |
| **0.2** | 13,425 | 0.957 | 0.937 | **0.947** |
| 0.25 | 12,530 | 0.978 | 0.894 | 0.934 |
| 0.3 | 11,715 | 0.986 | 0.843 | 0.909 |
| 0.4 | 9,551 | 0.992 | 0.691 | 0.814 |

The `scdet` filter gives the same result at its best threshold (8: F1 0.946).
The `ffmpeg` pass ran at 133 times real time on 4 CPU threads.

**Hand check of the disagreements.** At threshold 0.2, 803 reference cuts
have no `ffmpeg` cut and 578 `ffmpeg` cuts have no reference cut. A random 15
of each, viewed as two-second frame strips:

* all 15 reference-only cuts are **real**. Eight are cuts inside an inset or
  split-screen picture while the rest of the frame stays; seven are cuts
  between dark or low-contrast shots;
* about 10 of the 15 `ffmpeg`-only cuts are **false** (camera motion, a hand
  near the lens, a moving crowd, a webcam exposure change), and about 5 are
  real cuts that the reference missed or placed more than 0.5 s away.

**Tiles do not fix it.** The same films scored on a 3 by 3 grid of tiles
(the cut is where any tile, or k tiles, pass a threshold) never beat the
whole-frame score: the best tile rule equals it (F1 0.947), and every rule
that adds recall loses more precision (best tile at 0.3: recall 0.963,
precision 0.843). At the 803 missed cuts the best tile score has a median of
0.29 and a 75th percentile of 0.38, against 0.78 at the found cuts; motion
reaches the same range. A pixel difference cannot separate these cuts from
motion.

**Conclusion.** `ffmpeg` at 0.2 is a sound default with no model: about 94% of
the learned detector's cuts with about 96% agreement on its own cuts. A corpus
where inset layouts or dark footage matter uses a served learned detector.
The default threshold is 0.2.

## 5. Keyframes

Keyframes are derived, not stored: the middle of each shot, plus one more for
each `max_keyframe_gap_s` of a long shot, so a twenty-minute studio shot is
not described from one frame. The rule depends only on the shot and one
setting, so every step that reads keyframes reads the same frames, and a
cache of their images is optional.

## 6. Out of scope

* Shot-aligned windows for direct media embeddings (SPEC §8.1.7). The windows
  stay fixed-length; aligning them to shots is a possible follow-up.
* Per-shot facts (shot type, faces, camera motion) and their storage.
* Gradual transitions as their own spans. A dissolve is one cut at the time
  the detector reports.
