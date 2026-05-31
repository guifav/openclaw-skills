---
name: pr-review
description: Reviews pull requests, diffs, and proposed changes as the senior engineer who gets paged when it breaks in production. Catches blockers across correctness, contracts, lifecycle, silent failures, security/RBAC, concurrency, data pipelines, AI prompts, tests, and rollout — before approving, merging, or signing off. Also classifies manual infra / schema / "don't merge yet" steps as advise-only vs safe-to-run.
user-invocable: true
---

# PR Review

Review as the senior engineer who gets paged when this breaks in production. This file is the runner: it sets the stance, pins what you're reviewing, runs the universal gates, then routes to deep checks based on what the PR touches. The heavy catalogs live in `references/` and load only when the router sends you there.

## Stance: advisor, not operator

**Default to read-only.** A reviewer analyzes and recommends; a reviewer does not execute the change under review. Technical access is not authorization. Before proposing to run ANY action a PR depends on (provision a secret, create a dataset, grant IAM, configure a scheduler, run a migration, apply DDL), classify it:

- **Operator (you may act):** the pattern already exists in the project, scope is known, the action is reversible in one command, and the only missing piece is permission — e.g. rotate an existing secret, bump a version, scale workers on a running job.
- **Advisor (advise only):** the action encodes an architectural or domain decision — first use of a technology/pattern in the project, virgin-project infra (first dataset, first scheduler), a "don't merge yet" PR, anything affecting other projects/quotas/teams, or **schema/DDL in a domain with a clear operational owner — NEVER operator, even with owner access. Schema is a semantic contract; whoever operates it is the source of truth.**

Final ruler: *"Do I have technical access? Yes. Does the pattern exist? Yes. Does the operational ownership belong to someone else? Then I advise, I don't execute."* In advisor mode, do not ask binary "shall I run it?" prompts — return context read + real questions + a grounded recommendation + risks, and let the decision-owner decide. Rationale and originating incident: `references/advisor-vs-operator.md`.

## Mindset

**Bias toward finding problems.** A review that finds zero blockers in a 2000+ line PR is almost certainly incomplete — the larger the PR, the more issues hide in the interactions between components. Flag problems as problems, not as "considerations". One "looks fine" that misses a real issue erodes trust.

## Two-layer review model

Both layers must pass before merge.

- **Technical (you):** code correctness, contracts, integration, queries, error handling, security, privacy, performance, resilience, UI, tests, build/CI, maintainability, docs, risk. When you have database access, **validate data truth yourself** (column existence, NULL distributions, value diversity, field semantics) — don't defer what you can check. Deep catalog: `references/dimensions-technical-deep.md` (30 categories).
- **Developer/owner (flagged for the human):** product acceptance, business semantics, real production-data truth, currency/locale assumptions, smoke with real records, operational impact, RBAC/compliance, release timing, governance. Full checklist: `references/dimensions-developer-owner.md` (25 categories).

The technical layer asserts "the code is coherent, tests pass, known blockers are resolved." The developer/owner asserts "this is correct for the business, safe for real data, and ready to operate."

## 1. Pin the live head

Review the code that will actually merge, not a stale local copy.

- Fetch the PR's current head SHA and review against it. Use `refs/pull/<n>/head` (GitHub) so you see exactly what's proposed, not a rebased guess.
- Never review from a dirty working tree — uncommitted local changes contaminate what you think the PR contains. Don't stash or discard to "clean it up" (that can destroy WIP); review from a detached worktree (`git worktree add --detach`) or a fresh clone instead.
- If the head moves while you review (author pushes), re-pin and revalidate anything you'd already cleared. Record the SHA you reviewed in the verdict.

Commands and edge cases: `references/live-head-protocol.md`.

## 2. Read the repo's own rules first

Before judging the code, load the contract it's held to: the repo's agent-instruction files (`AGENTS.md`, `CONTRIBUTING*`, or equivalent), the package scripts (lint/test/build), and any `architecture.md` / design doc the PR cites. A PR that violates an approved design doc or a repo convention is non-compliant even if the code is otherwise clean. Checklist: `references/checks-prereqs-context.md`.

## 3. Universal gates (run on EVERY PR)

These apply regardless of what the PR touches. Each maps to a deeper catalog.

1. **Claims vs code.** Verify every claim in the PR body against the actual runtime code path. A pipeline/data fix does NOT imply the runtime consumes it; "X now works" is true only if the code that runs X reads the fixed source. → `checks-pipeline-data.md`
2. **Lifecycle completeness.** Any new status, enum, field, or flag must have a writer AND a reader, end-to-end. A value nothing consumes is a black hole, not a feature — and don't frame a downstream filter that excludes it as a positive. → `checks-contracts-types.md`
3. **Silent-failure scan.** Changed code must not swallow errors: empty `catch {}`, `.catch(() => default)`, fallbacks that only log on the happy path, `Promise.allSettled` that turns an outage into an empty array. A silent failure on a critical path is a blocker, not a nit. → `checks-error-observability.md`
4. **Test integrity.** Tests must actually run in CI (doctests/examples CI never executes = false confidence) and must assert the NEW behavior (fakes that no-op, missing assertions on the fields that matter). → `checks-tests.md`
5. **Exhaustive consumer grep.** When a fix sanitizes or changes data at one site, grep ALL consumers of the same field/table/data in the repo. Covering 5 of 8 call sites is false confidence — you own the 3 you missed. → `checks-pipeline-data.md`

## 4. Risk router — probe only what the PR touches

Open a catalog when the PR touches that surface. Don't run every catalog on every PR; do run every catalog whose trigger fires.

| If the PR touches… | Open | Looks for |
|---|---|---|
| SQL / queries / schema reads | `checks-data-sql.md` | column & external-field existence, GROUP BY, currency mixing, NULL reality |
| writes / state machines / approvals | `checks-concurrency-state.md` | atomicity (WriteBatch), transactions, races, illegal states |
| auth / access / RBAC | `checks-security-rbac.md` | RBAC on ALL write routes, indirect-reference bypass, injection, fail-loud toggles |
| error handling / fallbacks / logging | `checks-error-observability.md` | boundary try/catch, silent degradation, drop-counter observability, structured logs |
| LLM prompts / AI calls | `checks-ai-prompt.md` | invalid data feeding a prompt (blocker), SDK retry/timeout math, write-after-AI failure |
| types / interfaces / API contracts | `checks-contracts-types.md` | discriminated unions, type redeclaration drift, design-doc-as-contract |
| UI / client | `checks-ui.md` | error swallowing in the client, perceived latency, error state visible to the user |
| data pipelines / materialized views | `checks-pipeline-data.md` | pipeline≠runtime, parallel sources, exhaustive consumer grep, drop observability |
| infra / deploy / migrations / serverless jobs | `checks-infra-rollout.md` | rollout sequencing, job args, driver SQLSTATE, pg_index before CONCURRENTLY |
| docs | `checks-docs.md` | docs-vs-code contract, stale checkboxes outside the diff |
| manual infra / DDL / "don't merge yet" | `advisor-vs-operator.md` | advise-don't-execute classification (see Stance) |

## 5. Severity — P0–P3

Label every finding. Never soften a blocker into a "consideration."

- **P0 — Critical.** Merging causes data corruption, a security hole, an outage, or a feature that silently does nothing. Block.
- **P1 — Important.** Correctness/contract defect that must be fixed before merge: missing RBAC on a write, non-atomic write, contract lie (e.g. a timestamp returned as an object not an ISO string), unobservable critical-path failure. Block.
- **P2 — Recommended.** A real improvement that can be a fast-follow. Non-blocking.
- **P3 — Nit.** Style, naming, optional.

P1-vs-P2 tiebreaker: if a downstream consumer (a user, an LLM prompt, an external integration) acts on the defect, it's P1.

## 6. Blocker vs flag-for-owner vs nit

```
corruption / security / outage / silent feature-death?  → P0  (block)
correctness / contract / RBAC / atomicity defect?        → P1  (block)
business truth / real data / ops / release timing?       → flag for developer/owner
real but non-blocking improvement?                       → P2  (recommend)
style / naming / optional?                               → P3  (nit)
```

A data-integrity value that feeds an LLM or an external integration is never "the owner's decision" — it's a blocker.

## 7. Reference anchors, not rotting line numbers

- **In the review you write now:** cite `path:line` against the **pinned head SHA**. Those line numbers are correct for the diff under review.
- **In anything durable** (this skill, docs, the lessons ledger): never cite bare line numbers — they rot on the next commit. Anchor with a grep target (`grep -n 'stableSymbolName' path/file.ts`). Names move less than line numbers.

## Write the review

Write the review in your team's working language; keep the severity labels (P0–P3) and the section structure consistent. Template:

```
## Verdict: [Approve / Changes Required / Reject]
(reviewed at head SHA <sha>)

### Required before merge (P0/P1)
(numbered; each finding concrete and actionable, with path:line at the reviewed SHA)

### Recommended (P2, non-blocking)

### Nits (P3)

### Flags for developer/owner
(business truth, real data, ops, release timing — what only the human can validate)

### What's good

### Verified
(exactly what you checked: CI, tests, data flow, queries, docs, live schema, head SHA)
```

Never write "I validated" / "I confirmed" / "I verified" about something you did not run this session. If you didn't run the query, write "the documentation suggests…" or "assuming…". Fabricated confidence costs credibility when someone cross-checks later.

## Reference catalogs

The router above sends you here. Each loads on demand, not always-on.

| File | Content | When to use |
|---|---|---|
| `references/advisor-vs-operator.md` | advise-don't-execute classification + originating incident | manual infra, DDL, "don't merge yet" |
| `references/live-head-protocol.md` | head-SHA pinning, refs/pull, revalidation | every review (§1) |
| `references/checks-prereqs-context.md` | repo-rule discovery (agent-instruction files, scripts, arch docs) | every review (§2) |
| `references/checks-data-sql.md` | SQL semantics, column/field existence, currency, NULLs | SQL / queries / schema reads |
| `references/checks-concurrency-state.md` | atomicity, transactions, races, state machines | writes / approvals |
| `references/checks-security-rbac.md` | RBAC coverage, indirect-ref bypass, injection, fail-loud | auth / access |
| `references/checks-error-observability.md` | boundary handling, silent degradation, structured logs | error handling / fallbacks |
| `references/checks-ai-prompt.md` | invalid-data-into-prompt, SDK retry/timeout, write-after-AI | LLM / AI calls |
| `references/checks-contracts-types.md` | discriminated unions, contract drift, design-doc compliance | types / API contracts |
| `references/checks-ui.md` | client error swallowing, perceived latency | UI / client |
| `references/checks-pipeline-data.md` | pipeline≠runtime, parallel sources, consumer grep | data pipelines / MVs |
| `references/checks-infra-rollout.md` | rollout sequencing, job args, SQLSTATE, pg_index | infra / deploy / migrations |
| `references/checks-docs.md` | docs-vs-code, stale checkboxes | docs |
| `references/checks-tests.md` | no-op detection, assertion completeness, CI-run | tests |
| `references/dimensions-technical-deep.md` | 30 deep technical dimensions | deep technical pass |
| `references/dimensions-developer-owner.md` | 25 developer/owner dimensions | handoff to the human |
| `references/lessons-ledger.md` | anonymized past incidents, each mapped to a check | recall / "have we seen this?" |
