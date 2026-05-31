# Changelog — pr-review

All notable changes to this skill will be documented in this file.

## [1.0.0] — 2026-05-30

### Added
- Initial release.
- Slim runner `SKILL.md`: the advisor-vs-operator stance, the "bias toward finding problems" mindset, the two-layer (technical + developer/owner) review model, live-head pinning, repo-rule prerequisites, five universal gates, a risk router, the P0–P3 severity ladder, and a review-output template.
- 17 load-on-demand reference catalogs under `references/`:
  - `advisor-vs-operator.md` — advise-don't-execute classification for manual infra / DDL / "don't merge yet" PRs (technical access ≠ authorization).
  - `live-head-protocol.md` — head-SHA pinning, `refs/pull/<n>/head`, dirty-tree handling via a detached worktree or fresh clone (never blind stash/discard), and re-pin-on-push revalidation.
  - `checks-prereqs-context.md` — mergeability, CI status (gates approval, not the review itself; "no checks reported" ≠ green), repo-rule discovery, and a two-tier reviewability gate.
  - `checks-data-sql.md`, `checks-concurrency-state.md`, `checks-security-rbac.md`, `checks-error-observability.md`, `checks-ai-prompt.md`, `checks-contracts-types.md`, `checks-ui.md`, `checks-pipeline-data.md`, `checks-infra-rollout.md`, `checks-docs.md`, `checks-tests.md` — surface-specific deep checks opened by the risk router.
  - `dimensions-technical-deep.md` (30 technical dimensions) and `dimensions-developer-owner.md` (25 business/ops dimensions).
  - `lessons-ledger.md` — an anonymized incident-recall index; each lesson maps to the check that now guards it.
- Severity ladder P0–P3 with a P1-vs-P2 tiebreaker (if a downstream consumer acts on the defect, it's P1).
- Honesty guardrail: never claim "validated / verified / confirmed" for a command you did not run this session.
