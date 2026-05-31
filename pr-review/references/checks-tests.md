# Tests Checks

Open this when the PR touches tests — detect no-ops, assert completeness, and verify CI actually runs the mechanism.

---

## Test Coverage Fundamentals

- Are there tests for the new behavior introduced by this PR?
- Is there regression coverage for the specific bug being fixed? Would the new test have **failed before the fix**? If not, the test proves nothing.
- Do tests cover relevant edge cases (empty input, null, boundary values, concurrent access)?
- Do mocks represent the real contract — not a stale, partial, or over-simplified shape?
- Are partial-failure cases tested (one source down, one write fails)?
- Are critical numbers asserted, not just existence? (`expect(result.count).toBe(3)` not `expect(result.count).toBeDefined()`)
- Do snapshots not mask changes? (Snapshot tests that auto-update on every run add zero coverage.)
- Are tests immune to date/time fragility? (No `new Date()` in assertions unless frozen.)
- Is coverage proportional to risk? A critical auth path with zero tests is a blocker regardless of overall coverage percentage.

**Key rules:**
- **Count blockers, not test count.** 46 tests with 7 blockers is not a ready PR. → lessons-ledger.md
- **Mock fidelity = test fidelity.** A no-op `orderBy()` in a fake DB means all ordering-dependent tests are lying — they pass even when ordering is broken. → lessons-ledger.md
- **Visual test claims without committed baselines = false coverage.** If the baseline PNGs aren't in the repo, the visual test has nothing to compare against. → lessons-ledger.md

---

## Test Assertion Completeness — Assert the Invariant

When a test checks a state transition (e.g., approve a request):

- Assert ALL fields that should change: `status`, `decidedByUid`, `decidedAt`, etc.
- A test that only checks `status === 'approved'` without checking `decidedByUid` would still pass if `decidedByUid` were accidentally removed from the write.
- **If removing a line from the production code doesn't break any test, that line has zero coverage.** Read the test assertions and ask: what could I delete from the handler and still pass?
- **Shadow-mode tests**: verify shadow tests exist for every RBAC-enabled route. If precedent workspaces all have shadow tests, a new workspace PR that lacks them is **P2** at minimum. → lessons-ledger.md

---

## Decorative Tests = False Confidence — **P1**

- Doctests, inline asserts, or ad-hoc test scripts mentioned in the PR body but NOT executed by CI are **decorative**. They give false confidence.
- Verify CI actually runs the test mechanism. If `python -m doctest` isn't in `ci.yml`, doctests are not tests.
- **"Keep in sync" invariants**: if a PR claims "Python mirrors TS" or "schema matches type", verify there is a CI step that enforces the sync automatically. A comment without automated enforcement rots silently.
- → lessons-ledger.md
