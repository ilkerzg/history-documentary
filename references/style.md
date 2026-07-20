# Image style contract — softfacet Fable (Seedream 5 Pro)

Every frame is painted by **`bytedance/seedream/v5/pro/edit`** in edit mode: the six reference
images below are passed as `image_urls`, and the prompt orders the model to copy ONLY their art
style. The references are pre-hosted on fal so any environment can fetch them; Seedream renders the
style directly from them.

## Style reference URLs (pass all six as `image_urls`)

```
https://v3b.fal.media/files/b/0aa2e209/pj5t5w_zdYerL642ZIu3L_0.jpg
https://v3b.fal.media/files/b/0aa2e209/lumW1OiN7651Qhy1NTHhB_1.jpg
https://v3b.fal.media/files/b/0aa2e209/eHQftkDZ02g956Cdn7HRi_2.jpg
https://v3b.fal.media/files/b/0aa2e209/VOZh-BMRO5TPwf0NkK7SE_3.jpg
https://v3b.fal.media/files/b/0aa2e209/OftE9cA3sSJIV2GucMfF2_4.jpg
https://v3b.fal.media/files/b/0aa2e209/_NMJ1OLdlWt3xJHXezxHZ_5.jpg
```

## Prompt template (verbatim — this exact wording survived many bad variants)

Final prompt = `LEAD + <one concrete visual sentence> + ANCH`

LEAD:
```
Use the reference images ONLY for ART STYLE, never for their subject or characters. FLAT 2D
CARTOON art style, mandatory and overriding realism: thick uniform solid-black outlines around
every shape, simple round-cornered geometric heads with small minimal facial features, flat cel
shading with hard-edged colour blocks and NO gradients, bold highly-saturated flat fills. NOT a
photograph, NOT a painting, NOT a 3D render. Render this subject in that exact style:
```

ANCH (appended after the subject):
```
, softly rounded geometric facet construction, round-cornered polygon heads with simple defined
faces, flat cel shading, clean bold dark outlines, bold saturated palette (sky blue #4FB3E8,
orange #F59B2D, red #E2603F, green #6FBF44, cobalt #2E6FA3, brown #8A5A33) plus warm skin tones,
faces never blank, HIGH saturation, NOT pastel, NOT dark, NO yellow wash, flat, no text
```

## Call shape

```
bytedance/seedream/v5/pro/edit
{ image_urls: [<the 6 refs>], prompt: LEAD + subject + ANCH,
  image_size: { width: 2048, height: 1152 }, num_images: 1, output_format: "jpeg" }
```
2048×1152 = exact 16:9 at 2K. Cost ≈ $0.07/frame.

## Character consistency (no edit-chains needed)

Keep a cast roster of canonical visual descriptions (one line each, e.g. "ODYSSEUS: weathered
middle-aged Ithacan king, dark curly hair, short dark beard, scar above right knee, bronze
sword…"). When a sentence's picture involves a recurring character, the visual prompt MUST begin
by restating that exact description. Restated-look + fixed style refs keeps characters visually
identical across hundreds of independent frames — proven across 394 frames of a 40-minute production.

## Subject-sentence rules

- One vivid present-tense sentence; concrete and physical (never "symbolizes", "represents").
- Period-true: embed the film's period lock (era, dress, architecture) into the prompt-writing
  system prompt so anachronisms never reach the image model.
- "no text" stays in ANCH; never ask for captions/labels inside the image.
