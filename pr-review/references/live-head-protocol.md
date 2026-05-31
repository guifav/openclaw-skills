# Live Head Protocol

Pin the exact commit that will merge before reading a single line of diff — reviewing stale code is worse than not reviewing at all.

---

## Why this matters

A PR head can move at any moment: the author rebases, force-pushes a fix, or CI triggers an automatic rebase. If you load a diff from your local branch checkout and the author has since pushed, you are auditing code that will never ship. Any verdict you issue — approve or block — is issued against a phantom.

---

## Step 1 — Confirm a clean working tree

Before fetching anything, verify your working tree is clean:

```bash
git status
```

Uncommitted local modifications contaminate what `git diff` reports against the PR. **Do not stash or discard a dirty tree to "clean it up"** — those changes are work-in-progress, and stashing or discarding them without an explicit mandate from the user can destroy unsaved work. Instead, preserve the WIP and review from an isolated workspace that never touches it: a detached worktree (`git worktree add --detach <path> FETCH_HEAD`, reviewed there, then `git worktree remove`) or a fresh clone. Only stash or discard when the user explicitly tells you to.

---

## Step 2 — Fetch the PR head by ref, not by branch name

Do not use your local branch checkout. GitHub exposes a stable ref for every PR that resolves to exactly what is proposed, including for forks:

```bash
git fetch origin refs/pull/<n>/head
```

For forks this is the only reliable approach — the contributor's branch is not in your remote. `refs/pull/<n>/head` always works regardless of fork or same-repo PR.

---

## Step 3 — Pin the reviewed SHA

Capture the exact commit you will review:

```bash
gh pr view <n> --json headRefOid -q .headRefOid
```

Or, after the fetch above:

```bash
git log -1 FETCH_HEAD
```

Record this SHA. It goes into the verdict (see "Record the SHA" below).

---

## Step 4 — Diff against the merge base, not the branch tip

Never do a raw branch compare. Diff against the actual merge base so you see only the proposed delta, not any main-branch changes that accumulated after the PR was opened:

```bash
git diff $(git merge-base origin/main FETCH_HEAD)...FETCH_HEAD
```

This is the diff to review. Everything else — the PR description, CI output, comments — is context; the merge-base diff is the artifact.

---

## Step 5 — Re-pin if the head moves during review

If the author pushes or force-pushes while you are mid-review:

- Re-fetch: `git fetch origin refs/pull/<n>/head`
- Compare the new SHA against the one you pinned in Step 3.
- Any section of your review that you had already completed must be re-validated against the new diff. A finding that was true at SHA A may be resolved at SHA B, or a new one may have appeared.
- Do not issue a verdict until you have re-cleared anything that overlaps the changed files.

---

## Step 6 — Record the reviewed SHA in the verdict

The review template includes a line:

```
reviewed at head SHA <sha>
```

Fill it with the exact SHA from Step 3. This creates an auditable record: if a regression is later found, it is possible to determine whether the reviewed commit contained the offending code.

---

## Edge cases

**PR from a fork:** `refs/pull/<n>/head` resolves correctly. Do not attempt to checkout the contributor's fork branch — use the fetch above.

**Local branch that diverged from PR head:** If you have a local branch tracking this PR, your local tip and the PR head may differ (especially after a rebase). Trust `FETCH_HEAD`, not your local checkout. To avoid confusion, do not checkout the branch — work from `FETCH_HEAD` references only.

**`mergeable: false` / conflicted PR:** A PR with conflicts is not reviewable. Do not begin the technical review. Return to the author: rebase against main, resolve conflicts, then request re-review. The merge-base diff is meaningless when there are unresolved conflicts, because the diff you see is not what will land.

**Stacked / sequential branches:** When Branch 2 depends on Branch 1 not yet in main, fetch both and diff Branch 2 against the tip of Branch 1, not against main. Document the dependency in the verdict. Verify Branch 1 is actually in main before issuing any approval on Branch 2.
