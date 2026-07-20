# Step-by-step recipe, alignment math, and the verification protocol

Work in a job directory with checkpoints after every stage (a JSON per stage). Every stage must
be resumable: skip work whose artifact already exists on disk. Long stages run concurrently
(images 8-way, TTS 4-6-way) with per-item retry.

## 1. Research + period lock

One `openrouter/router` call → `{research}` (what actually happened, names, sequence) and a
**period lock** `{era, dress, architecture, objects}`. The period lock is injected into every
later prompt-writing system prompt — this is the anachronism firewall.

## 2. Screenplay — sized, then FROZEN

Target words ≈ `minutes × 140`. Ask for numbered NARRATOR lines (plus rare character lines),
each line one sentence, plain vivid documentary prose, no meta.
**After this stage the screenplay is IMMUTABLE** (law #1). Persist `rows[]` immediately.

## 3. Cast roster

For each recurring person/animal/object: `{name, visual}` where `visual` is one dense line
(age, build, hair, clothing, signature props). This exact text is restated inside every image
prompt that features them (style.md).

## 4-6. Voice

- Narrator: `eleven-v3` per line → `l000.wav …` (resumable).
- Character lines: `seed-audio-1.0` (voice described once, reused).
- Join: `workflow-utilities/merge-audio` with per-line `gaps` (house: 0.35 s between lines,
  0.9-1.1 s at paragraph turns) → `narration` master. Then `amix-audio` narration+music `[1.0,
  0.10]` LATER (step 11) — keep the un-mixed narration too; Scribe runs on narration, not the mix
  (music can only hurt STT).

## 7. Scribe the narration (chunked)

Scribe-v2 caps at 1200 s. For duration D:
- chunks of ≤800 s with 2 s overlap: `[0,800], [798,1598], …`
- scribe each chunk; add chunk start-offset to every word time
- keep-windows partition D without overlap (`[0,798), [798,1596), …`): keep a word iff its
  absolute start ∈ that chunk's window (dedupes the overlap safely even mid-word)
- concatenate → `words[]` (absolute, gapless, spans the whole track). Sanity: last word `end`
  ≈ track duration; total count plausible (~150-160 wpm of the screenplay).

## 8. Sentence↔word alignment (GLOBAL, not greedy)

Normalize both sides (`lowercase, strip non-alphanumerics`). Build the screenplay token stream
with row ownership, then run a global sequence alignment (difflib `SequenceMatcher`,
`autojunk=False`) between screenplay tokens and scribe words. From matching blocks map each row
to its first/last matched word → `row_start[i] = words[first].start`.
- Rows with no matched token (rare): interpolate between neighbors.
- Enforce monotonicity.
- Expect ≥99% token match; if far below, the narration and screenplay disagree — STOP and find out
  why (usually a rewritten screenplay: law #1 violated).
Greedy last-token matching (the old way) works ~99% of the time and silently misplaces the rest —
always use global alignment.

## 9-10. One prompt, one frame per sentence

- Prompts: batch 10 sentences per `openrouter/router` call. System prompt embeds the period lock
  + full cast roster + the rules in style.md. Output: `{ "<i>": "<one concrete picture>" }`.
- Frames: Seedream 5 Pro edit with the 6 style refs (style.md), 2048×1152, saved as `p{i:04d}.jpg`.
  Index `i` IS the sentence number — never reorder, never reallocate.

## 11-13. Score, mix, assemble

- Music ≤240 s/piece (loop or multi-piece for long films), instrumental, restrained.
- `amix-audio` `[narration, music] × [1.0, 0.10]`, `duration: "first"`.
- Visual timeline: sentence `i` is on screen from `row_start[i]` to `row_start[i+1]` (last: to
  audio end). **Snap starts to the frame grid** (`frame = round(start × 30)`, min 3 frames per
  image, cumulative integer frames) so 400 cuts accumulate zero rounding drift.
- Assembly options:
  a) fal-only: `image-to-video` per sentence window (30 fps, alternate zoom direction) →
     `ffmpeg-api/compose` with ms keyframes + the mixed master.
  b) local shortcut (faster, equally correct): ffmpeg concat of stills with exact frame
     durations + `-c:a copy` of the master. Either way the timing source is identical.

## 14. Subtitles — auto-subtitle, last, on the finished video

Exact call + limits in endpoints.md. If the film exceeds the endpoint's size limit: split at
sentence-start silences (cut 0.12 s before a sentence's first word so no word is clipped),
subtitle each part, rejoin with concat `-c copy`. Verify both seams afterwards (protocol below).

## 15. Verification protocol (mandatory)

1. Choose ≥4 timestamps: one early (~30 s), one mid, every join seam, one deep (>90%).
2. For each t: read the words spoken at t from `words[]`; extract the frame at t from the
   DELIVERED file; confirm (a) the burned subtitle shows those words, (b) the image depicts what
   the sentence says, (c) durations: video ≈ audio ≈ expected.
3. Check the previously-known-bad regions of a repaired film explicitly.
4. Confirm stream sanity: one h264 video + one aac audio, both ≈ full duration.
Only after all pass, present the file. If the user reports an issue anyway, extract frames at the
reported moments FIRST — diagnose with evidence, not assumptions.

## Cost yardstick (40-min film, 394 sentences)

Seedream frames ~$28 · narration TTS ~$3-5 · scribe/music/utilities ~$1-2 · LLM ~$1.
Roughly $0.85-1.0 per finished minute.
