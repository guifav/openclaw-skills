# Documentation Checks

Open this when the PR touches docs — verify docs-vs-code contract accuracy and catch stale checkboxes outside the diff.

---

## Documentation Accuracy

- Do README/docs reflect current state, not aspirational or future state?
- **Verify every API contract in docs against the actual route handler.** API docs that describe the wrong response shape are worse than no docs — they mislead integrators. → lessons-ledger.md
- **Verify every enum/list in docs against the source of truth.** Enum list in docs missing a value that exists in `lib/types.ts` = silent contract violation. → lessons-ledger.md
- **Cross-check new docs against existing docs** for contradictions. A PR that fixes `docs/README` but leaves the root `README` stale has fixed nothing. → lessons-ledger.md
- Are new fields documented where the repo expects (API docs, `AGENTS.md`, type definitions)?
- Are contract changes reflected in API/feature docs?
- Are follow-up items clearly marked as follow-ups — not presented as complete?
- Are known risks explicit in the PR and/or docs?
- Do examples use realistic names, not placeholder values?
- Are links and file paths still correct?

### Issue/PR state references

- Check if an issue/PR number referenced in docs is still open/closed as the doc assumes. A doc that says an issue is "pending" when it was closed months ago is actively misleading. → lessons-ledger.md
- Line number references rot instantly — prefer file-path-only anchors.

### Concrete numbers must match CI

- Test counts, coverage percentages, and other concrete numbers in docs must match what CI actually produces. A stale "46 tests passing" claim undermines trust in all other doc claims.

### State changes must propagate beyond the PR diff

- **`grep -r` the entire repo** when a PR changes a concept (status name, field name, behavior). The diff shows only what changed; grep shows what was missed.
- A doc fix that covers the PR diff but leaves stale references in 3 other files in the repo is an incomplete fix. → lessons-ledger.md

---

## Asymmetric Comments Across Layers — **P2**

When the same fix is applied in multiple files (e.g., a Python pipeline script AND a TypeScript route):

- Verify WHY comments are symmetrical across all files containing the fix.
- If only one file has a WHY comment, the other becomes a target for "DRY cleanup" that removes the protection without understanding it. → lessons-ledger.md
- **Best practice**: explicitly couple the comments — "Do not remove without also removing the matching guard in `[other/file.ts]`."
- Asymmetric comments are low-severity individually but systematically produce hidden tech debt that removes load-bearing code in future refactors.
