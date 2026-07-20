---
name: history-documentary
description: Produce a fully narrated, illustrated, scored and word-synced-subtitled history documentary film (3-40+ min) on fal.ai from just a topic + duration, using a production-proven pipeline - Seedream 5 Pro frames in the softfacet Fable style, ElevenLabs v3 narration, Scribe v2 timing, and workflow-utilities assembly. Use when the user asks for a historical/documentary film, "tarih belgeseli", "X hakkında belgesel/film yap", a narrated long-form video on a historical topic, or wants to fix sync/subtitle/assembly problems in such a film.
---

# History Documentary (a production-proven pipeline)

Turn `topic + minutes` into a finished documentary film: research → period lock → screenplay →
cast → narration → word-accurate timing → one painted frame per sentence → music → assembly →
word-synced burned subtitles. Every step is a fal.ai call; no local ML, no local ffmpeg needed
(local ffmpeg is optional for the concat/verification shortcuts noted below).

This skill encodes the exact pipeline that produced and then FIXED a real 40-minute documentary.
The ordering rules and gotchas below are each the result of a real, painful failure. Do not
reorder the pipeline. Read `references/endpoints.md` for every endpoint's exact name, inputs and
limits; `references/style.md` for the mandatory image-style contract and reference URLs;
`references/pipeline.md` for the step-by-step recipe with alignment math and verification protocol.

## The five laws (violating any of these caused a real broken film)

1. **Freeze the screenplay before shot planning.** Never rewrite, trim, or renumber screenplay
   lines after any downstream artifact (shots, images, narration) exists. A real 40-minute film's
   images drifted onto the wrong sentences exactly because the screenplay was re-cut after the
   shot plan was made. If the screenplay must change, regenerate EVERYTHING downstream.

2. **One image per sentence, index-locked.** Image `i` illustrates screenplay sentence `i` and is
   shown exactly during sentence `i`'s spoken window. Never let an LLM freely allocate "beats"
   across lines for long films - the index lock makes visual drift structurally impossible.

3. **One clock: the finished narration audio.** All timing (cuts AND subtitles) derives from
   Scribe v2 word timestamps of the actual narration track. Never derive timing from estimated
   durations, per-line audio lengths + gaps arithmetic, or any second timeline.

4. **Subtitles are burned LAST, by `auto-subtitle`, on the finished video.** The endpoint
   transcribes the final video itself, so text can never drift from audio. Never build subtitle
   cues yourself; never burn subtitles from a parallel timeline (that caused both the drift and
   a 1-frame-per-cue "flicker" bug).

5. **Seedream 5 Pro is the only image hand.** `bytedance/seedream/v5/pro/edit` with the six
   style-reference images (see `references/style.md`).

## Pipeline at a glance (details in references/pipeline.md)

| # | Stage | fal endpoint |
|---|-------|--------------|
| 1 | Research + period lock | `openrouter/router` (model `openai/gpt-5.6-terra`) |
| 2 | Screenplay (~140 wpm × minutes, then FROZEN) | `openrouter/router` |
| 3 | Cast + canonical visual roster | `openrouter/router` |
| 4 | Narration TTS, per line | `fal-ai/elevenlabs/tts/eleven-v3` |
| 5 | Character lines (if any) | `bytedance/seed-audio-1.0` |
| 6 | Merge narration + gaps | `fal-ai/workflow-utilities/merge-audio` |
| 7 | Word timestamps (chunk ≤1200s!) | `fal-ai/elevenlabs/speech-to-text/scribe-v2` |
| 8 | Sentence↔word alignment | difflib-style global alignment (not greedy) |
| 9 | One visual prompt per sentence | `openrouter/router` (roster embedded) |
| 10 | One frame per sentence | `bytedance/seedream/v5/pro/edit` + style refs |
| 11 | Score | `fal-ai/elevenlabs/music` (≤240s pieces) |
| 12 | Mix narration+music | `fal-ai/workflow-utilities/amix-audio` weights `[1.0, 0.10]` |
| 13 | Motion + assembly | `fal-ai/workflow-utilities/image-to-video` (30fps) + `fal-ai/ffmpeg-api/compose` |
| 14 | Word-synced subtitles LAST | `fal-ai/workflow-utilities/auto-subtitle`, `words_per_subtitle: 3` |
| 15 | Verification | frame-vs-word spot checks (protocol in references/pipeline.md) |

## Non-negotiable verification

Never declare a film done without the spot-check protocol: pick ≥4 timestamps (start, middle,
any join seams, deep end), read what words are spoken there from the Scribe timeline, extract the
video frame at that second, and confirm image content + subtitle + audio all describe the same
moment. A film that "should be right by construction" still gets checked.

## Quick failure→cause map

- Subtitles drift mid-film → two timelines were used; redo timing from Scribe of the final audio.
- Subtitles blink/flash → hand-built cues with gaps; use `auto-subtitle` on the finished video.
- Images show the wrong scene → screenplay changed after shot planning, or beats weren't
  index-locked; re-prompt per sentence and re-render (images are cheap; drift is not).
- Scribe rejects audio → >1200s; split at silence with ~2s overlap, offset, dedupe by keep-window.
- auto-subtitle rejects video → too long/large; split at sentence-start silences, subtitle each
  part, rejoin with stream copy.
- `highlight_color` error → named colors only (use `orange` for the house gold).
