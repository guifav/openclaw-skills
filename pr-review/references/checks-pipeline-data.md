# Data Pipeline Checks

Open this when the PR touches data pipelines / materialized views.

---

## Pipeline Fix vs Runtime Fix

**The single most missed pattern in pipeline PRs.** A fix in the pipeline script does not fix the runtime path if the runtime reads from a different source.

### Trace Both Consumption Paths (P0)

When a PR fixes data in a materialized or aggregated table (e.g., BigQuery, Firestore, a Postgres materialized view), identify ALL consumers and split them:

- **Materialized-path consumers**: API routes, dashboards, and tools that query the fixed table. These benefit from the fix.
- **Runtime-path consumers**: engines, scoring systems, and live queries that read the RAW source (not the materialized output). These do NOT benefit.

**Action**: `grep -rn "<raw_source_table>"` across the entire repo. If the raw table name appears in runtime code (not just the pipeline script), the PR must either (a) fix the runtime path too, or (b) explicitly document the gap as a known limitation with a follow-up issue.

**Common trap**: pipeline builds `contact_interests_all` from `event_category_map`, but the matchmaking engine also reads `event_category_map` directly. Fix in pipeline ≠ fix in engine. → lessons-ledger.md

---

## Verify Business-Impact Claims Against Code Paths (P1)

When the PR body says "signals now score in matchmaking", "uplift in candidate counts", or similar, the reviewer MUST trace the actual runtime code path to confirm.

Steps:
1. Identify the claim (e.g., "these rules can finally fire for these contacts")
2. Find the runtime function that evaluates the claim (e.g., `getContactInterests()`, `findCandidates()`)
3. Check what data source that function reads — materialized table? raw table? API?
4. Confirm the PR actually fixes the data at THAT source

If the claim is about matchmaking and the PR only fixes a materialized table, but the matchmaking engine reads a different source: the claim is invalid. **P1 blocker.** → lessons-ledger.md

---

## Parallel / Sibling Data Sources

When the PR fixes a data quality issue in one source, the reviewer MUST check if sibling sources carry the same bug.

Steps:
1. Identify the bug pattern (e.g., "editorial vocabulary used where canonical is expected")
2. List all sources that feed the same dimension/field (e.g., event_category, newsletter categories, user-declared interests, CRM-declared values)
3. For each sibling source, check if the same bug exists
4. If a sibling has the same bug, flag it — even if the declared PR scope is one source only

**Critical sub-pattern — false comments about upstream canonicalization.** If a comment in the code says "source X already arrives canonical", VERIFY it. Read the upstream code. Check which table the upstream writes to vs which table the runtime reads from. If they are different tables, the comment is lying. **P1.**

> Concrete example: a comment said "behavioral categories already arrive canonical from upstream" — but the classifier script wrote to an `enriched` table, while the matchmaking engine read from a `raw` table. Same editorial vocabulary, same bug, masked by a false comment. → lessons-ledger.md

---

## Exhaustive Consumer Search (Mandatory for Data-Fix PRs)

**When a PR sanitizes, filters, or validates a field: `grep -rn` the field name across the ENTIRE repo before approving.**

- List every file that reads/writes/joins on the field. Verify EACH consumer is covered by the fix or explicitly deferred with a follow-up issue.
- **A fix that covers 5 of 8 consumers is WORSE than no fix** — it creates false confidence that the problem is solved.
- Check not just the same query, but other queries, other routes, other scripts, and library functions that use the same table/field.
- **The reviewer who approves "with coverage" owns the consumers they missed.** This is the single most common review failure. → lessons-ledger.md

---

## Drop / Filter Observability (Mandatory When Rows Are Discarded)

When a filter silently drops rows (invalid identifier, malformed data, etc.):

- Is there a log/warn when rows are dropped? The log must include a count, not raw payload (PII risk).
- Is the log structured — not swallowed by `.catch(() => [])`?
- Is there a way to detect when drop rate exceeds a threshold? 100% drop = empty UI with no alarm. **P0.**

If the PR adds a filter without logging, that is a **P0 blocker** — silent data loss is worse than noisy bad data. → lessons-ledger.md

---

## Status Changes That Affect External Systems (Mandatory)

When a PR changes which rows receive a given status, and the OLD status triggered an external action (CRM writeback, email, webhook, automation flow):

- Check if the status change silently disables that external action. **P0.**
- Concretely: if `auto_approved` rows feed into `auto_approved_rows → automation flow → an external CRM`, and the PR now sets some of those rows to `needs_review` instead, those rows no longer reach the CRM. This is a silent degradation of an external integration.
- The PR body must document this side-effect explicitly. If not documented, flag as blocker. → lessons-ledger.md

---

## New Status / State Values — Full Lifecycle Required (Mandatory)

When a PR introduces a new status value (e.g., `needs_review`, `suspended`, `archived`), trace the FULL lifecycle:

1. **Write**: where is the new status written? (extractor, API route, cron job)
2. **Read/Display**: where is the new status shown to users? (UI filter, admin panel, API response)
3. **Transition**: where can the status be changed? (approve/reject routes, admin actions)
4. **Filter**: where is the status filtered out? (downstream consumers, pipelines, scoring)

If ANY step is missing, the new status creates a **data black hole** — rows written but never visible, actionable, or resolvable. **P0 blocker.**

**Common trap**: PR writes `status='needs_review'` but the review UI only shows `pending` rows and the approve/reject API only accepts `pending` → quarantined rows are invisible and unactionable. → lessons-ledger.md

---

## Symptom vs Root Cause — Track Both (P2)

When the PR is a containment fix (defensive filter against bad data from upstream):

- Is there an open issue against the upstream root cause?
- Does the PR body say explicitly that this is containment, not a definitive fix?
- Does the code comment anchor the filter to the issue, to prevent future "DRY cleanup" that removes the protection?

A containment fix without an upstream issue is incomplete. The reviewer must ensure the upstream gap is tracked.

---

## Empty Catch Blocks in Pipeline Paths (P0)

`catch {}` or `catch { // proceed without it }` in a data pipeline path is a **blocker**, not a recommendation.

When a worker catches an error and continues without the data, the output is silently degraded:
- HTML dashboard regenerated without original layout (object-storage read failed)
- AI prompt built without source data (file parse failed)
- Refresh marked "completed" when the result is fundamentally different from the original

Fix: either fail the job (preferred for data integrity) or annotate warnings on the output doc so the UI can distinguish "clean refresh" from "degraded refresh". → lessons-ledger.md
