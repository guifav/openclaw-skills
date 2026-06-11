# Pipeline Reference — Video Editor

Working command set for the six-artifact chain. Everything here is code the user can re-run; treat `footage/` as read-only and keep all decision artifacts (JSON/YAML/LUTs/components) in git.

## 0. Project layout

```
edit-project/
├── footage/            # raw clips (read-only, gitignored)
├── transcripts/        # <clip>.json (word-level) + .srt + .txt
├── edit.json           # the edit decision list — the producer's review gate
├── cut/                # stitched first cut
├── grade/              # candidate looks, LUTs, playground.html
├── overlay/            # Remotion project (React)
└── out/                # renders (gitignored)
```

## 1. Word-level transcription (Whisper)

```python
# transcribe.py — run over every clip in footage/
from faster_whisper import WhisperModel   # pip install faster-whisper
import json, sys, pathlib

model = WhisperModel("large-v3", compute_type="int8")  # CPU-friendly; use GPU if available
for clip in pathlib.Path("footage").glob("*.mp4"):
    segments, _ = model.transcribe(str(clip), word_timestamps=True, vad_filter=True)
    words = [{"word": w.word.strip(), "start": round(w.start, 3), "end": round(w.end, 3)}
             for seg in segments for w in seg.words]
    out = pathlib.Path("transcripts") / f"{clip.stem}.json"
    out.write_text(json.dumps({"clip": clip.name, "words": words}, indent=2))
```

Extract audio first if transcription directly from video is slow:
`ffmpeg -i clip.mp4 -vn -ac 1 -ar 16000 -c:a pcm_s16le clip.wav`

## 2. Scene clustering + take scoring

Takes of the same scene read near-identical scripts. Cluster by transcript similarity — no manual labeling:

```python
# cluster.py (sketch) — pip install rapidfuzz scipy
from rapidfuzz import fuzz
from scipy.cluster.hierarchy import linkage, fcluster
import numpy as np

texts = {clip: " ".join(w["word"] for w in words).lower() for clip, words in transcripts.items()}
clips = list(texts)
n = len(clips)
dist = np.zeros((n, n))
for i in range(n):
    for j in range(i + 1, n):
        sim = fuzz.token_set_ratio(texts[clips[i]], texts[clips[j]])
        dist[i, j] = dist[j, i] = 100 - sim

Z = linkage(dist[np.triu_indices(n, 1)], method="average")
labels = fcluster(Z, t=NUM_SCENES, criterion="maxclust")   # NUM_SCENES from the script/user
```

- Within-cluster mean similarity < ~80 → put the clip in an `unassigned` bucket (slates, aborted takes) and ask.
- Order scenes by earliest file `creation_time` per cluster; reshoots appear as a late time block — use as a cross-check, and remember **user notes override scoring** ("the final scene was reshot" → prefer the late block for that scene).

Score takes per scene: filler-word count (`um|uh|like`), repeated bigrams (restarts), word gaps > 1.2s (dead air), completeness vs. the scene's consensus script, then take order (later wins ties).

## 3. edit.json — the edit decision list

```json
{
  "scenes": [
    {
      "scene": 1,
      "candidates": ["C0003", "C0007", "C0011"],
      "chosen": "C0011",
      "reason": "latest take, 0 fillers, full script coverage",
      "in": 4.62,
      "out": 41.08
    }
  ]
}
```

In/out = first scripted word − 0.5s pre-roll, last word + 1.0s post-roll, snapped into surrounding silence. The producer edits this file to override any pick; everything downstream re-runs from it.

## 4. Frame-accurate cut + concat (ffmpeg)

```bash
# Per scene — re-encode for frame accuracy (-ss BEFORE -i is fast and accurate when re-encoding)
ffmpeg -ss 4.62 -i footage/C0011.mp4 -t 36.46 \
  -c:v libx264 -preset slow -crf 16 -c:a aac -b:a 256k cut/scene1.mp4

# Stitch (identical codecs → concat demuxer)
printf "file '%s'\n" cut/scene*.mp4 > cut/list.txt
ffmpeg -f concat -safe 0 -i cut/list.txt -c copy cut/first_cut.mp4
```

For maximum quality through the pipeline, use a mezzanine instead of H.264:
`-c:v prores_ks -profile:v 3 -pix_fmt yuv422p10le -c:a pcm_s24le` → `.mov`.

## 5. Color grade — candidates as code

The source is flat/neutral (Rec.709 off the camera, or log). Write **2–3 candidate filter chains**, render 10s samples of each, let the user pick:

```bash
# look_a: warm punch    look_b: cool clean    look_c: filmic soft
ffmpeg -i cut/first_cut.mp4 -vf \
  "curves=master='0/0.02 0.5/0.52 1/0.98',eq=saturation=1.12:contrast=1.06,colorbalance=rs=0.02:bs=-0.02" \
  -t 10 grade/sample_look_a.mp4
```

**Package the chosen look as a LUT** so it's a swappable artifact (and editable in Resolve/Photoshop):

```bash
ffmpeg -f lavfi -i haldclutsrc=8 -frames:v 1 grade/identity.png
ffmpeg -i grade/identity.png -vf "<chosen filter chain>" -frames:v 1 grade/look.png
# Apply:
ffmpeg -i cut/first_cut.mp4 -i grade/look.png \
  -filter_complex "[0:v][1:v]haldclut=interp=tetrahedral[v]" -map "[v]" -map 0:a \
  -c:v libx264 -crf 16 -c:a copy cut/graded.mp4
```

Only bake point operations (curves/eq/colorbalance) into a LUT; spatial ops (vignette, grain) go in a separate final pass.

**HTML knob playground** (for non-technical stakeholders): one HTML file with the draft cut in a `<video>`, range sliders for brightness/contrast/saturation/temperature driving a CSS `filter` preview, and an "export settings" button that downloads the slider values as JSON. CSS filters only *approximate* ffmpeg — it's a communication tool; you translate the exported values into the real filter chain.

## 6. Overlays — Remotion project

```bash
cd overlay && npx create-video@latest .   # or: npm create video@latest
```

Structure:

```
overlay/src/
├── index.ts          # registers <Composition id="Overlay" ...> sized/fps'd to the cut
├── Overlay.tsx       # maps cues → components, frame math from useCurrentFrame()
├── cues.ts           # resolved cues: {phrase, asset/component, startSec, endSec, knobs}
└── components/       # one React component per graphic (your design, then Figma-synced)
```

- **Resolve cues from words, not timecodes.** Fuzzy-match each trigger phrase against the *final-timeline* word JSON (remap word times: `t_final = (t_word − scene_in) + scene_offset`). Hard-error if a phrase doesn't match — never silently drop a graphic.
- Animate with `spring()` / `interpolate()`; expose durations, easings, and positions as props (knobs the user can adjust without touching layout code).
- Transparent-background composition rendered over the graded cut, either by importing the video into the composition (`<OffthreadVideo src>`) or rendering an alpha overlay track:

```bash
npx remotion render src/index.ts Overlay out/final.mp4 --props=cues.json
# alpha track variant: --codec=prores --prores-profile=4444 → composite with ffmpeg overlay
```

## 7. Figma round-trip (design team)

1. First pass: generate the design assets yourself (from the script) and implement them as the Remotion components.
2. Export the overlay designs to Figma — one frame per overlay state — via the Figma MCP/API so the design team edits in their native tool.
3. They iterate freely (layout, type, color, motion intent in notes).
4. Pull changes: *"the design has been updated in this Figma — update the video to match."* Read the file via Figma MCP (`get_design_context` / `get_screenshot`), re-implement the React components to match, re-render.
5. Also hand them the grading playground HTML if they want a say in the look.

Do **not** replace this loop with a fixed SVG-export contract — it limits designers to recoloring inside your preset animations.

## 8. Optional hardening (only when the job demands it)

| Need | Add |
|---|---|
| Footage too heavy to iterate (≥ ~25GB) | 540p proxies: `ffmpeg -i in.mp4 -vf scale=-2:540 -c:v libx264 -preset veryfast -crf 28 proxy.mp4`; cut decisions on proxies, conform from originals |
| Broadcast / paid distribution | Two-pass `loudnorm` (measure → apply, target −16 LUFS / −1.5 dBTP); QA pass: `blackdetect`, A/V duration delta < 1 frame, bt709 color tags, a still per cue to confirm each graphic fired |
| Reshoot color drift | Per-scene gray-world channel gains (clamped ±10%) via `colorchannelmixer` before the LUT — verify visually |
| Clicks at joins | 15ms `afade` in/out baked at the per-scene cut |
| Captions | Free from the word JSON → `.srt` |
