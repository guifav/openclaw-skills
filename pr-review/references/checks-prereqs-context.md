# Checks — Prerequisites and Context

Load the contract the code is held to, and confirm the PR is reviewable, before judging a single line of code.

---

## Why this must come first

Skipping prerequisites costs more than it saves. Reviewing a PR with conflicts produces a verdict against code that cannot land. Reviewing without reading the repo's agent-instruction file (`AGENTS.md`) means you will miss repo-specific conventions and declare compliant code non-compliant or vice-versa. Reviewing from the PR description instead of the full diff means you are auditing intent, not implementation.

Two things genuinely block the review from starting: an unmergeable (conflicted) PR and an unread diff — see the reviewability gate in §6. Everything else here (red or absent CI, a branch behind main, unread repo rules) does not stop you from reading the code and flagging blockers; it shapes the verdict and gates *approval*, not the review.

---

## 1. Mergeability and branch health

- **`mergeable` status must be clean.** A PR with conflicts is not reviewable. Return it to the author to rebase.
- **Is the branch up to date with main?** If main has advanced since the PR branch was cut, the PR may silently revert merged work. First refresh your view of main: `git fetch origin main`. Then list what main has that the PR head is missing: `git log FETCH_HEAD..origin/main` (commits reachable from `origin/main` but not from the PR head). Note the direction — the inverse range `git log origin/main..FETCH_HEAD` lists the PR's *own* commits (useful to confirm what the PR adds), not what it is behind on.
- If the branch is behind, require a rebase before review. Do not attempt to mentally "subtract" main changes.

---

## 2. CI status

- **Non-green CI blocks approval, not the review itself.** A red or pending build means you must not *approve* — but you can and should still read the diff and flag blockers. Don't bounce the PR unread; a failing build is itself a finding to report alongside whatever the code reveals. Record in the verdict that CI was not green on the reviewed SHA.
- When CI is red, separate concerns: distinguish failures the PR introduced from a pre-broken baseline. If the baseline is already broken, say so explicitly — don't attribute baseline failures to the PR, and don't credit the PR for a baseline that was already green.
- **"No checks reported" is NOT a green signal.** Absence of CI ≠ passing CI — never treat a PR with no checks as if the build passed. When no checks ran, run the validation locally against the pinned SHA (`npm test`, `npm run lint`, `npm run build`, `git diff --check`) and review against those results, noting in the verdict that CI was absent and you validated locally.
- Check CI against the exact SHA you pinned (see `live-head-protocol.md`), not against a previous run.
- Warnings that are new (not pre-existing) are flagged; pre-existing warnings are noted.
- **Tooling checks:**
  - `npm test` passes
  - `npm run lint` passes
  - `npm run build` passes
  - `git diff --check` is clean (no trailing whitespace, no conflict markers)
- **Lockfile and dependency hygiene:** if `package-lock.json` or equivalent changed, verify there is a documented reason (explicit dep bump, security patch). Unexplained lockfile changes are a flag.

---

## 3. Repo rules discovery

Before judging any code, load the following in order. A PR that violates an approved design doc or a documented repo convention is non-compliant even if the code is otherwise clean.

**Required reads (always):**
- `AGENTS.md` — the primary agent-instruction contract for this repo: auth patterns, rate-limit rules, coding conventions, data sources, datastore guardrails, test expectations, driver quirks (e.g. "convert datastore timestamps with `.toDate()?.toISOString()` before returning to the client")
- Tool-specific agent-instruction files — some teams keep an additional per-agent `*.md` at the repo root alongside `AGENTS.md`; read whichever your stack maintains and treat it as the same kind of contract
- `CONTRIBUTING*` — human contribution conventions, if present

**Package scripts (always):**
- Read `package.json` scripts section. Know what `lint`, `test`, `build`, `typecheck` actually run. A CI step that calls `npm run lint:strict` is different from `npm run lint`.

**PR-specific reads (when referenced in the PR body):**
- Any `architecture.md`, `design-doc.md`, `ADR-*`, or spec file the PR cites or implements. When a PR implements a design doc, every numbered decision (D1–D5, C1–C8, or equivalent) must be cross-checked against the implementation. Divergence from an approved design is a **blocker**, not a nit — the design was approved as a binding contract.
- Open issues referenced in the PR body — understand what the PR claims to close, and verify those issues are still open (an issue already closed by another PR is a signal to re-read scope).

---

## 4. Load project context from the agent-instruction file

Extract and internalize before proceeding:

- **Auth patterns** — what middleware enforces authentication, where it lives, what headers/tokens it validates.
- **Rate-limit rules** — per-route, per-user, or global; which routes are exempt and why.
- **Coding conventions** — naming patterns (e.g. a prefix marking the origin system of a field, like `crm_*` for fields synced from an external CRM vs `enriched_*` for derived values), file organization, import rules, layer responsibilities.
- **Data sources** — which databases are read-only vs writable from the app layer, which tables are canonical vs materialized.
- **Guardrails** — e.g. "PG numeric columns arrive as JS strings via node-postgres; cast at query boundary", "Firestore FieldValue.delete() is no-op in set(..., {merge: false})", "serverless scale-down kills fire-and-forget — await side effects".
- **Test expectations** — required coverage patterns (shadow-mode tests for RBAC-enabled routes, snapshot discipline, doctest execution in CI).

If the agent-instruction file conflicts with what the PR implements, that conflict is itself a finding — it is either a non-compliance bug or a signal that the instruction file needs updating, which is a separate action the PR should document.

---

## 5. Read the full diff

- Read **every file** in the diff. Not just the PR description. Not just the files mentioned in the summary comment.
- If the diff is large (>500 lines), read file by file. Do not skim. Do not trust the PR description to accurately summarize what changed.
- Note files touched but not in the diff (e.g. lockfile, auto-generated files) — these are not code to review but are signals about tooling.
- After reading the diff, summarize mentally: what layers are touched, what data flows changed, what external integrations are affected. This sets the scope for the technical review steps that follow.

---

## 6. Reviewability gate

Two tiers. The first blocks the review itself; the second blocks only the *approval*, while the review proceeds and folds the gap into the verdict.

**Blocks review entirely — bounce to author, write no technical findings:**

| Check | Required state |
|---|---|
| `mergeable` | clean / no conflicts — a conflicted diff is not what will land, so findings against it are meaningless |
| Full diff read | every file accessible and read, no skimming |

**Blocks approval, but review still proceeds (record as a finding, don't bounce):**

| Check | Required state | If not satisfied |
|---|---|---|
| CI on pinned SHA | all required checks green | review the code, flag blockers, record CI red/absent in the verdict; do not approve. "No checks" ≠ green — validate locally (see §2) |
| Branch up to date | within acceptable distance from main | note the drift and the silent-revert risk; recommend a rebase before merge |

**Always loaded before judging code (not a bounce condition, but required for a sound review):**

| Check | Required state |
|---|---|
| `AGENTS.md` loaded | extracted auth, rate-limit, conventions, guardrails, test expectations |
| Design docs loaded | all docs cited in PR body read and decisions extracted |

If a "blocks review entirely" row is not satisfied, return the PR to the author with a specific description of what must be resolved before review resumes. For the approval-gating rows, proceed with the review and state plainly in the verdict that approval is withheld until they are resolved.
