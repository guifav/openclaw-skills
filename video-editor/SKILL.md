---
name: video-editor
description: >
  Use when turning raw footage into a finished video without a GUI editor — a folder of unlabeled
  takes to pick from, clips to stitch together, flat/log footage to color grade, animated UI
  overlays that must appear in sync with spoken lines, or video graphics a design team iterates on
  in Figma. Triggers: "edit this video", "pick the best take", "stitch these clips", "grade this
  footage", "show a graphic when I say X", Whisper/ffmpeg/Remotion post-production.
user-invocable: true
---

# Video Editor

You are the video editor. There is no GUI editing session — **the entire edit is code and reviewable artifacts**. Every editorial decision becomes a file (transcripts, edit decisions, grade recipes, React components) that the user can inspect, diff, override, and re-render. This is the workflow that cut a real product-launch video from 25GB of raw takes with zero human time inside a video editor.

## When to Use

- A folder of raw takes (often unlabeled) that must become one polished video
- Best-take selection across multiple takes of the same scenes
- On-screen graphics/UI that must trigger on specific spoken words
- Color grading flat/neutral (Rec.709 or log) camera footage
- A design team (Figma, non-coders) iterating on the look of video graphics

**When NOT to use:** one-off format conversion or a single trim (a plain ffmpeg one-liner — no pipeline needed); heavy VFX/motion tracking/3D; footage with no speech (the transcript-driven techniques here have nothing to bite on).

## The Artifact Chain (core pipeline)

Six artifacts. Ship this chain before adding anything else:

```
footage/*.mp4
  → transcripts/<clip>.json   word-level transcription (Whisper, word_timestamps=True)
  → edit.json                 per scene: candidate takes, chosen take, why, in/out times
  → cut.mp4                   ffmpeg concat of the chosen segments (re-encode, frame-accurate)
  → grade/                    2–3 candidate looks as code + HTML knob playground
  → overlay/                  Remotion project — React components timed by transcript words
  → npx remotion render       final video
```

Each artifact feeds the next; a change re-runs only its downstream stages.

1. **Transcribe everything first, word-level.** One pass of faster-whisper with `word_timestamps=True` over every clip. This single artifact powers take clustering, take selection, cut points, overlay timing, and free captions. Without word timing, nothing downstream works.

2. **Cluster takes into scenes by transcript similarity — never ask the user to label clips.** Takes of the same scene read near-identical scripts; fuzzy text similarity separates them cleanly. Cross-check with file creation time (reshoots show up as a late block).

3. **Select the best take per scene with the heuristics below.** Decide in the open: every pick gets a one-line rationale.

4. **`edit.json` is the review gate.** Per scene: candidate takes, chosen take, reason, start/end timestamps. The producer reviews and overrides by editing this file — not by re-explaining preferences in chat. Never bury cut decisions inside ffmpeg commands where nobody can see or change them.

5. **Cut with ffmpeg, re-encoding.** Stream-copy (`-c copy`) only cuts on keyframes; re-encode (with `-ss` before `-i`) for frame accuracy. Prefer clean cut points at silences and scene boundaries; mid-take surgery is possible, but start clean.

6. **Grade as code, in candidates.** The user usually cannot articulate a grade but can pick one — always generate 2–3 candidate looks (ffmpeg filter chains or LUTs) from the flat source toward different finished looks. Ship an HTML playground with sliders so non-technical stakeholders can tweak exposure/contrast/saturation/temperature and send settings back.

7. **Overlays are React components (Remotion), never burned-in drawtext.** One composition reads the transcript JSON and triggers each graphic on the words being spoken — "when I say 'is Claude doing the right work', swap the component." Trigger by phrase, not hardcoded timecode, so timing survives a take swap. Expose durations, easings, and positions as props (knobs).

8. **Render via CLI.** `npx remotion render` composites graded footage + overlay components into the final file. The render is reproducible: same artifacts in, same video out.

## Take-Selection Heuristics

| Signal | Rule |
|---|---|
| User notes | Override everything — "the final scene was reshot" beats every score |
| Take order | Later takes are usually better (performer warmed up) |
| Filler words | Fewer "um/uh/like" wins |
| Restarts & pauses | Penalize repeated phrases and word gaps > ~1.2s |
| Completeness | The take must cover the scene's full script |
| Tie-break | Audio quality (loudness, noise floor) |

Record the winning reason per scene in `edit.json`.

## Design Team Loop (Figma round-trip)

Designers don't review code. Give them their native tool and keep ownership of the implementation:

1. Build the first pass yourself in Remotion — generate design assets from the script ("Claude-design" quality is fine as v1).
2. Export the design to Figma (a frame/component per overlay state).
3. Designers edit in Figma freely — layout, type, color, spacing. No constraints.
4. When they're done: pull the updated file via Figma MCP — *"the design has been updated in this Figma, update the video to match"* — and re-implement the React components to match it.
5. Re-render. Repeat until sign-off.

**Anti-pattern:** locking designers into a fixed SVG-filename + animation-preset contract. It feels robust but caps them at recoloring; real design iterations change layout and motion. You own the React; they own the look; Figma MCP is the bridge.

## Stakeholder Artifacts

Every non-technical collaborator gets an **editable artifact**, never just a conversation:

| Who | Artifact |
|---|---|
| Producer / director | `edit.json` + watermarked draft renders |
| Design team | The Figma file (round-trip above) |
| Grading opinions | HTML knob playground (sliders → exported settings) |
| "What did I say where?" | `transcripts/*.srt` |

## Right-Sizing

Start with the six-artifact chain — it ships a launch video. Add hardening **only when the job demands it**: proxies (footage too heavy to iterate on, ~25GB+), two-pass loudness normalization and a QA pass (broadcast or paid distribution), per-scene color matching (mixed lighting or reshoots), short audio fades at joins (audible clicks). Don't build a broadcast facility by default.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Asking the user which clip belongs to which scene | Transcripts + similarity clustering answer it |
| Hardcoding overlay timecodes | Resolve trigger phrases against the word JSON at render time |
| One take-it-or-leave-it grade | 2–3 candidates + the knob playground |
| ffmpeg `drawtext`/`overlay` for graphics | Remotion components — iterable, designable, animatable |
| `-c copy` cuts mid-clip | Re-encode for frame accuracy |
| Edit decisions living only in chat or shell history | `edit.json` with rationale, user-editable |
| Fixed asset contract for designers | Figma MCP round-trip; re-implement the components |
| Building 13 pipeline stages for a 2-minute video | Six artifacts first; harden on demand |

## References (load on demand)

- `references/pipeline-reference.md` — the working command set: faster-whisper snippet, scene-clustering recipe, `edit.json` schema, frame-accurate ffmpeg cut/concat, grade candidates + Hald-CLUT/LUT packaging, HTML playground skeleton, Remotion project layout and render flags, Figma MCP loop, optional hardening (proxies, loudness, QA).
