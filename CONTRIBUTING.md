# Contributing to OpenClaw Skills

Thanks for your interest in contributing! This repository is a collection of
[OpenClaw](https://clawhub.ai) skills covering the full-stack development
lifecycle. Each skill is a self-contained folder that can be loaded into
OpenClaw and (optionally) published to ClawHub.

This guide explains how the repo is laid out, the anatomy of a skill, and how
to add or change one.

By participating, you agree to abide by our [Code of Conduct](./CODE_OF_CONDUCT.md).

---

## Repository layout

Every top-level directory (other than tooling files) is a single skill:

```
openclaw-skills/
├── stack-scaffold/        # one skill
├── feature-forge/         # one skill
├── pr-review/             # one skill
├── video-editor/          # one skill
├── _template/             # copy this to start a new skill
├── examples/              # shared sample output files
├── README.md              # overview + skill catalog
├── CONTRIBUTING.md        # this file
├── CODE_OF_CONDUCT.md
└── LICENSE
```

There is no build step. A skill is just files that OpenClaw reads.

---

## Anatomy of a skill

A skill folder **must** contain three files:

| File | Required | Purpose |
|------|----------|---------|
| `SKILL.md` | Yes | The skill itself — instructions the agent follows. Starts with YAML frontmatter. |
| `claw.json` | Yes | The manifest — metadata, permissions, requirements (see reference below). |
| `CHANGELOG.md` | Yes | Versioned history, [Keep a Changelog](https://keepachangelog.com/) style. |

Optional supporting folders:

| Folder | Purpose |
|--------|---------|
| `references/` | Load-on-demand deep-dive docs the `SKILL.md` points to. Keeps the main file slim. |
| `docs/` | Longer-form documentation for the skill. |

### `SKILL.md` frontmatter

```markdown
---
name: my-skill
description: >
  Use when <the situation that should trigger this skill>. Triggers:
  "<example user phrase>", "<another phrase>".
user-invocable: true
---

# My Skill

<the instructions the agent follows>

## When to Use
- ...

**When NOT to use:** ...
```

- `name` must match the folder name.
- `description` should describe **when** to use the skill and include trigger
  phrases — this is what the agent matches against.
- Keep `SKILL.md` focused. Push long command sets and deep references into
  `references/` and link to them.

### `claw.json` reference

```jsonc
{
  "name": "my-skill",                 // must match folder + SKILL.md name
  "version": "0.1.0",                 // SemVer
  "description": "One-line summary shown in listings.",
  "author": "your-handle",
  "homepage": "https://github.com/guifav/openclaw-skills",
  "license": "MIT",                   // keep MIT unless you have a reason not to
  "permissions": ["filesystem", "network"],
  "models": ["claude-*"],
  "tags": ["search", "keywords", "for", "discovery"],
  "minOpenClawVersion": "0.5.0",
  "entry": "SKILL.md",
  "skillDependencies": {              // optional
    "recommended": ["other-skill"]
  },
  "openclaw": {
    "requires": {
      "bins": ["ffmpeg", "python3"],  // CLI binaries the skill needs
      "env": []                       // required environment variables
    },
    "emoji": "🛠️",
    "os": ["darwin", "linux", "win32"]
  }
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | Matches the folder name. |
| `version` | Yes | Semantic version. New skills start at `0.1.0` or `1.0.0`. |
| `description` | Yes | One line; appears in catalogs. |
| `author` | Yes | Your handle. |
| `homepage` | Yes | This repo URL. |
| `license` | Yes | `MIT` to match the repo license. |
| `permissions` | Yes | Subset of `filesystem`, `network`. Request only what you use. |
| `models` | Yes | Supported model globs, e.g. `claude-*`. |
| `tags` | Yes | Lowercase keywords for discovery. |
| `minOpenClawVersion` | Yes | Minimum OpenClaw version. |
| `entry` | Yes | Always `SKILL.md`. |
| `skillDependencies` | No | `recommended` skills that pair well with this one. |
| `openclaw.requires.bins` | Yes | External CLI binaries the skill assumes exist. |
| `openclaw.requires.env` | Yes | Required env vars (use `[]` if none). |
| `openclaw.emoji` | Yes | A single emoji for the skill. |
| `openclaw.os` | Yes | Supported platforms. |

### `CHANGELOG.md` format

```markdown
# Changelog — my-skill

All notable changes to this skill will be documented in this file.

## [0.1.0] — 2026-06-11

### Added
- Initial release: <what it does>.
```

Use `### Added`, `### Changed`, `### Fixed`, `### Removed` as appropriate, and
date entries `YYYY-MM-DD`.

---

## Adding a new skill

1. **Copy the template:**
   ```bash
   cp -R _template my-skill
   ```
2. **Fill in the files** — replace every `<placeholder>` in `SKILL.md`,
   `claw.json`, and `CHANGELOG.md`. Make sure `name` is identical across the
   folder, `SKILL.md` frontmatter, and `claw.json`.
3. **Add references/docs** if needed under `my-skill/references/`.
4. **Register it in the README** — add a row to the relevant table in the
   "Skills Overview" section and, if appropriate, the workflow diagram and the
   publishing list.
5. **Open a PR** (see below).

## Changing an existing skill

- Bump the `version` in `claw.json` following SemVer:
  - **patch** (`1.0.1`) — fixes, wording, no behavior change.
  - **minor** (`1.1.0`) — new capabilities, backward compatible.
  - **major** (`2.0.0`) — breaking changes to behavior or interface.
- Add a corresponding entry to that skill's `CHANGELOG.md`.
- Update the README if the skill's purpose or requirements changed.

---

## Conventions

- **Commit messages** follow the existing history:
  `feat:`, `fix:`, `docs:`, `chore:`, `refactor:` — e.g.
  `feat(video-editor): add LUT export`.
- **SemVer** for all skill versions.
- **No `.DS_Store`** or other OS/editor cruft — these are gitignored; don't
  force-add them.
- **Request minimal permissions** in `claw.json`.
- **Keep `SKILL.md` slim**; move depth into `references/`.

---

## Pull request process

1. **Fork** the repo and create a branch from `main`:
   `git checkout -b feat/my-skill`.
2. Make your changes following the conventions above.
3. **Self-check before opening the PR:**
   - [ ] `name` matches across folder / `SKILL.md` / `claw.json`.
   - [ ] `claw.json` is valid JSON and has all required fields.
   - [ ] `CHANGELOG.md` has an entry for this change.
   - [ ] README updated if a skill was added or its purpose/requirements changed.
   - [ ] No `.DS_Store` or secrets committed.
4. **Open a PR against `main`** with a clear description of what the skill does
   (for new skills) or what changed (for edits).

Validate a manifest quickly with:

```bash
python3 -c "import json,sys; json.load(open(sys.argv[1])); print('valid')" my-skill/claw.json
```

---

## Questions

Open a [GitHub issue](https://github.com/guifav/openclaw-skills/issues) or start
a discussion. Thanks for contributing!
