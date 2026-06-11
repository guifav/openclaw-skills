---
name: pr-review
description: >
  Reviews pull requests, diffs, and proposed changes as the senior engineer who gets paged when it breaks in production. 
  **Pragmatic version:** optimized for velocity with guaranteed minimum review coverage. 
  One automated reviewer (Claude Code) + lightweight checklist for author + GitHub Actions gating. 
  Multi-model (Codex + Kimi) is optional for production-critical changes only.
user-invocable: true
---

# PR Review (Pragmatic Edition)

Review as the senior engineer who gets paged when this breaks in production. This skill balances rigor with velocity. The default process is lightweight; multi-model deep review is reserved for high-risk changes.

## Process Overview

```
PR opened
  ├── Author fills lightweight checklist (5 items)
  ├── GitHub Actions runs (tests + lint + smoke)
  ├── 1 automated reviewer (Claude Code) OR manual review if tools fail
  ├── Human approves (if all green)
  └── Merge
```

**Golden rule:** A PR is NEVER merged with zero review. If automated tools fail, fallback to manual checklist review. If that also fails, document exception and create follow-up issue.

---

## 1. Author Checklist (pre-review)

The PR author must fill this before requesting review. Copy into PR body:

```markdown
## Author Checklist
- [ ] Tests pass locally (`pytest -q` or equivalent)
- [ ] No hardcoded credentials / secrets / API keys
- [ ] No static literals / snapshot data in generated output
- [ ] Issue referenced in title (`Refs #N` or `Closes #N`)
- [ ] Change tested in branch workflow (if applicable)
```

If any item is unchecked, review is blocked until author clarifies.

---

## 2. Review by PR Scope (differentiated rigor)

Not every PR needs the same depth. Route by scope:

| Scope | Examples | Review Required | Tool |
|---|---|---|---|
| **Hotfix** | Bug in production | Post-merge review (within 24h) | Self-review + checklist |
| **Trivial** | <10 lines, typo, rename | Self-review + checklist | None |
| **Refactor** | Extract function, rename | 1 reviewer, focus on regression | Claude Code |
| **Feature** | New functionality | 1 reviewer + smoke test | Claude Code |
| **Infra** | Workflow, deploy, secrets | 1 reviewer + staging test | Claude Code |
| **Production-critical** | Auth, billing, data pipeline | **Multi-model** (Claude + Codex + Kimi) | All three |

**How to classify:** If the change touches auth, payment, user data, or the critical path of a production cron/job, it's production-critical.

---

## 3. Automated Review (Claude Code)

Default reviewer for non-trivial PRs. Run via CLI:

```bash
claude -p --dangerously-skip-permissions \
  "Review PR #{N} in {repo}. Focus on: correctness, security (no secrets), 
  no static literals in output, test coverage, and whether the change 
  addresses the linked issue. Output: APPROVED or CHANGES_REQUIRED with 
  specific findings (P0/P1/P2/P3)."
```

**If Claude Code fails (timeout >120s):**
1. Retry once after 20s
2. If still failing → execute manual review (Section 5)
3. Document in PR comment: "Automated review unavailable. Manual review executed."

---

## 4. Multi-Model Deep Review (production-critical only)

Reserved for: auth changes, payment flows, data pipeline changes, schema migrations, critical cron modifications.

Run in parallel:
- **Claude Code:** Architecture + correctness
- **Codex:** Probe local (simulate payload, validate behavior)
- **Kimi:** Static analysis + data consistency

**Merge rule:** At least 2 of 3 must approve with zero P0/P1 findings. If 2 fail technically, document exception and escalate to human.

**When NOT to use multi-model:**
- PRs <50 lines
- Documentation changes
- CSS/HTML-only changes
- Dependency bumps with known changelog

---

## 5. Manual Review Fallback (when all tools fail)

If no automated reviewer is available, execute this manual checklist:

### 5.1 Diff Sanity Check
```bash
gh pr view {N} --json files --jq '.files[].path' | head -20
gh pr diff {N} --stat
```

### 5.2 Quick Review Questions
1. Does the diff touch only what the PR claims to touch?
2. Are there any hardcoded values that should be configurable?
3. Are error paths handled (not silently swallowed)?
4. Are there tests for the new behavior?
5. Does the change respect existing patterns in the repo?

### 5.3 Document
Comment on the PR:
```
Manual review executed (automated tools unavailable).
- Checked: diff scope, error handling, test coverage, patterns
- Verdict: [APPROVED / CHANGES_REQUIRED]
- Risks: [any concerns]
```

---

## 6. Severity Labels

Use consistently across all review types:

- **P0 — Blocker:** Data corruption, security hole, outage, silent feature death. Must fix before merge.
- **P1 — Blocker:** Correctness/contract defect (missing RBAC, non-atomic write, broken API contract). Must fix before merge.
- **P2 — Recommended:** Real improvement, non-blocking. Can be fast-follow.
- **P3 — Nit:** Style, naming, optional.

**Never soften a blocker into a "consideration."**

---

## 7. Exception Merge Protocol

When a PR MUST be merged before review is complete (hotfix, incident response):

1. Add label `exception-merge`
2. Document in PR body:
   ```
   ## Exception Merge
   - Reason: [why review is skipped/deferred]
   - Risk accepted: [what could go wrong]
   - Mitigation: [how we'll catch issues]
   - Follow-up: [issue # for post-merge review]
   ```
3. Create follow-up issue immediately after merge
4. Post-merge review must be completed within 24h

---

## 8. GitHub Configuration (enable gating)

Branch protection rules for `main`:

```
[x] Require a pull request before merging
    [x] Require approvals: 1
    [x] Dismiss stale PR approvals when new commits are pushed
    [x] Require review from CODEOWNERS (if configured)
[x] Require status checks to pass before merging
    [x] Require branches to be up to date before merging
    [x] Status checks: pytest, build, lint
[x] Restrict who can push to matching branches
    [x] Include administrators
[x] Allow force pushes: NO
[x] Allow deletions: NO
```

**Why this matters:** Without branch protection, `reviewDecision: REVIEW_REQUIRED` is cosmetic. The data from tech-team shows 12 consecutive PRs merged with zero reviews because nothing physically blocked the merge.

---

## 9. Review Output Template

```markdown
## Verdict: [Approve / Changes Required / Exception Merge]
(reviewed at head SHA <sha>)

### Blockers (P0/P1)
(numbered, actionable, with path:line)

### Recommended (P2)

### Nits (P3)

### What's good

### Verified
(exactly what was checked: tests, diff, smoke, head SHA)
```

---

## 10. Tool Reliability Notes

Based on empirical data from tech-team (2026-06-04 to 2026-06-09):

| Tool | Failure Rate | Typical Error | Mitigation |
|---|---|---|---|
| Codex CLI | ~40% | Timeout >120s | Retry once, then fallback |
| Kimi API | ~30% | 401 auth error | Check credentials, fallback |
| Claude Code | ~20% | Timeout on large PRs | Split large PRs, retry |

**Key insight:** A process that requires all 3 tools is guaranteed to fail ~70% of the time (union of failure rates). Design for 1 reliable reviewer + fallback, not 3 simultaneous reviewers.

---

## Reference Files

| File | When to Use |
|---|---|
| `references/historical-analysis-tech-team-2026-06-09.md` | Why this skill was rewritten |
| `references/dimensions-technical-deep.md` | Deep technical catalog (production-critical only) |
| `references/dimensions-developer-owner.md` | Handoff to human owner |
| `references/checks-data-sql.md` | SQL/schema changes |
| `references/checks-security-rbac.md` | Auth/RBAC changes |
| `references/checks-error-observability.md` | Error handling changes |
| `references/checks-ai-prompt.md` | LLM/AI prompt changes |
