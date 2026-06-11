# Changelog — video-editor

All notable changes to this skill will be documented in this file.

## [1.0.0] — 2026-06-11

### Added
- Initial release, distilled from the workflow that edited the Claude Fable launch video (17 takes, 4 scenes, 25GB of raw footage, zero human time in a video editor).
- Slim runner `SKILL.md`: the "entire edit is code and reviewable artifacts" principle, the six-artifact chain (word-level transcripts → `edit.json` → ffmpeg cut → grade candidates → Remotion overlays → CLI render), take-selection heuristics (user notes override everything; later takes win; fewest fillers), the Figma MCP design round-trip, stakeholder-artifact mapping, right-sizing guidance, and a common-mistakes table.
- `references/pipeline-reference.md` — load-on-demand working command set: faster-whisper word-level transcription, transcript-similarity scene clustering (rapidfuzz + scipy), `edit.json` schema, frame-accurate ffmpeg cut/concat, candidate grades + Hald-CLUT/LUT packaging, HTML grading playground, Remotion project layout and phrase-triggered cues, Figma MCP loop, and optional hardening (proxies, loudness, QA, reshoot color matching).
