# Every fal endpoint in the pipeline — exact names, key inputs, limits

All calls are fal queue calls (`fal.subscribe` / `fal_client.subscribe`). Retry transient
failures 3-5× with backoff; the queue occasionally 5xxs under load.

## Direction (LLM / VLM)

**`openrouter/router`** — model `openai/gpt-5.6-terra` directs the whole film (research, period,
screenplay, cast, per-sentence visual prompts).
`{ model, system_prompt, prompt, temperature, max_tokens }` → `output`/`response`.
For JSON: instruct "Return ONLY JSON", then parse the first `{`…last `}` slice; retry ≤3 on parse
failure. Batch per-sentence work ~10 lines per call.

**`openrouter/router/vision`** — same shape + `image_urls`. Used for optional QA (style gate,
anachronism check) with `temperature: 0, max_tokens ≤ 40`.

## Images

**`bytedance/seedream/v5/pro/edit`** — the ONLY image model (see style.md).
`{ image_urls: [6 style refs], prompt, image_size: {width:2048,height:1152}, num_images: 1,
output_format: "jpeg" }` → `images[0].url`. ~$0.07/frame. 8-way concurrency is safe;
394 frames ≈ 45-60 min.

## Voice

**`fal-ai/elevenlabs/tts/eleven-v3`** — narrator. `{ text, voice }` → `audio.url`.
Generate PER SCREENPLAY LINE (resumable; a bad line re-records alone). Pick `voice` ids from your
own ElevenLabs voice library (one calm, articulate documentary narrator works best; keep the same
voice for the whole film).

**`bytedance/seed-audio-1.0`** — character dialogue with a described voice.
`{ prompt, output_format: "mp3", sample_rate: 44100, speed: ~1.05 }` → `audio.url`.
Prompt pattern: voice description + "Read EVERY word to the very end, in order, skip nothing,
never cut off." (guards the truncated-line failure). NOTE: it has no negative_prompt-style knobs.

## Timing (the single clock)

**`fal-ai/elevenlabs/speech-to-text/scribe-v2`**
`{ audio_url, language_code: "eng", diarize: false, tag_audio_events: false }`
→ `{ text, words: [{text, start, end, type: "word"|"spacing", speaker_id}] }` (absolute seconds).
**HARD LIMIT: 1200 s of audio.** For longer narration: split into ≤~800 s chunks with ~2 s
overlap, scribe each, add the chunk offset to every timestamp, then keep each word only if its
absolute start falls inside that chunk's non-overlapping keep-window. Filter `type=="word"`.

## Music

**`fal-ai/elevenlabs/music`** — `{ prompt, force_instrumental: true, music_length_ms }`.
**Max 240 000 ms per piece**; for a long film loop/tile or request multiple pieces. House prompt
shape: "Restrained cinematic documentary underscore fitting <era/topic>: period-leaning
instrumentation, low strings, soft percussion, patient build, dignified, no vocals."

## Assembly (workflow-utilities + ffmpeg-api)

**`fal-ai/workflow-utilities/merge-audio`** — `{ audio_urls, gaps }` (seconds of silence inserted
between consecutive clips) → single narration track.

**`fal-ai/workflow-utilities/amix-audio`** — `{ audio_urls: [narration, music],
weights: [1.0, 0.10], duration: "first" }` → mixed master. 0.10 music weight is the house level.

**`fal-ai/workflow-utilities/image-to-video`** — one still → gentle Ken Burns clip.
Request 30 fps and the sentence-window duration; alternate zoom-in/zoom-out per shot for life.
(Skippable: static frames concatenated locally also read well and encode faster.)

**`fal-ai/ffmpeg-api/compose`** — assembles tracks by keyframes:
video track = `[{url, timestamp(ms), duration(ms)}...]` (each clip at its sentence window),
audio track = the mixed master. Timestamps in ms.

**`fal-ai/workflow-utilities/auto-subtitle`** — THE subtitle step, run LAST on the finished video.
```
{ video_url, words_per_subtitle: 3, language: "en",
  font_name: "Comic Neue", font_size: 88, font_weight: "bold", font_color: "white",
  stroke_color: "black", stroke_width: 5, highlight_color: "orange",
  enable_animation: false, position: "bottom", y_offset: 60 }
```
→ `{ video.url, subtitle_count, words, transcription }`. It transcribes the video itself
(ElevenLabs) and burns karaoke-style 3-word sets — sync is guaranteed by construction.
LIMITS/QUIRKS:
- `highlight_color` accepts ONLY named colors (white/black/red/green/blue/yellow/orange/purple/
  pink/brown/gray/cyan/magenta). `#hex` → 422. Use `orange` for the house gold.
- Rejects long/large videos ("Unable to download or load video" on a 40-min/183 MB file).
  Split at sentence-start silences (cut ~0.12 s before a sentence's first word) into ~13-min
  parts, subtitle each, rejoin with ffmpeg concat `-c copy` (same-pipeline parts concat cleanly).
- `font_name` is any Google Font. House: Comic Neue.

