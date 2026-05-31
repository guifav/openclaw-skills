# Lessons Ledger — Incident Recall Index

Reverse index of past review misses. Every check catalog forward-references this file
via `→ lessons-ledger.md`. You land here to answer one question:
**"have we seen this failure before?"**

- Entries are keyed by a short **descriptive slug** (not an internal ticket number), so
  they read the same in any repo.
- Each incident lists the check slug(s) that now guard it. The *detailed* trap lives in
  that check file; this ledger is the short recall hook + the mapping.

---

## The meta-pattern (read first)

The most expensive misses share one shape: **the reviewer approved with 0 blockers, then
a second (deeper or independent) pass found criticals.** Every time, the root cause was
the same — a check was skipped, not failed:

- did not trace the **runtime** path, only the changed file
- did not grep **all** consumers of a changed field/status
- did not compare against the **precedent** sibling route
- did not verify **atomicity / RBAC on every write route**
- framed a downstream gap as a *positive* instead of the actual *problem*
- published a retraction / a fabricated "I validated" **inside** the review

If you are about to approve with 0 blockers, re-run the universal gates against this list
before signing off.

---

## Slug → incidents (reverse lookup)

When you are working a specific check catalog, these are the incidents behind it.

| Check slug | Past incidents |
|---|---|
| `checks-data-sql` | consumed-vs-fetched; schema-audit-parser-blind-spots; incomplete-field-fix-coverage; atomicity-rbac-every-write |
| `checks-pipeline-data` | incomplete-field-fix-coverage; pipeline-is-not-runtime; new-status-no-lifecycle; silent-fallback-publish-validation |
| `checks-infra-rollout` | driver-quirks-at-the-boundary; shared-cache-delete-sentinel; fire-and-forget-under-scale-down; structured-logging-shape; rollout-sequencing-sqlstate |
| `checks-concurrency-state` | fire-and-forget-under-scale-down; atomicity-rbac-every-write |
| `checks-security-rbac` | config-fallback-rbac-type-leak; the-multi-miss-review; list-vs-per-resource-rbac; atomicity-rbac-every-write |
| `checks-error-observability` | audit-trail-contract-drift; incomplete-field-fix-coverage; silent-fallback-publish-validation; the-multi-miss-review |
| `checks-contracts-types` | audit-trail-contract-drift; consumed-vs-fetched; driver-quirks-at-the-boundary; centralized-degraded-contract; type-system-rigor; design-doc-precedes-write; config-fallback-rbac-type-leak; cache-design-baselines; shared-cache-delete-sentinel; new-status-no-lifecycle; the-multi-miss-review; atomicity-rbac-every-write |
| `checks-ai-prompt` | the-multi-miss-review; silent-fallback-publish-validation |
| `checks-ui` | the-multi-miss-review |
| `checks-tests` | best-effort-writes-and-mock-fidelity; stale-branch-auth-fake-visual; new-status-no-lifecycle; atomicity-rbac-every-write |
| `checks-docs` | docs-vs-code-contract; incomplete-field-fix-coverage |
| `advisor-vs-operator` | advisor-vs-operator |

---

## Incidents

### audit-trail-contract-drift
→ `checks-error-observability`, `checks-contracts-types`
- A `mark_sent` handler did not persist edits — the audit trail was silently broken.
- "Reopen output" was claimed in the History UI but the plumbing was missing (a dead capability).
- A getter returned `null` instead of throwing — the caller's `.catch()` never fires.
- A new enum value (`proposal`) was added but a downstream mapper still mapped to the old one (`pitch`).

### best-effort-writes-and-mock-fidelity
→ `checks-tests`, `checks-contracts-types`, `checks-security-rbac`
- A "best-effort" auto-create was approved with no recovery path.
- Prompt injection: one field (`anchorEvent`) was left unescaped while a sibling (`team_notes`) was escaped in the same file.
- A parent ID was accepted without validation; a linked-output ID was typed but had no write path.
- A create function returned a count, not the IDs.
- A fake DB's `orderBy()` was a no-op — every ordering test passed against the wrong order. **Mock fidelity = test fidelity.**

### stale-branch-auth-fake-visual
→ `checks-tests`, `checks-security-rbac`, `checks-prereqs-context`
- An out-of-date branch reverted already-merged work (branch-health gate).
- Auth "triple-guarded" with no shared secret; an `endsWith` domain check was open to a suffix attack.
- Visual testing claimed with no committed baseline images — nothing to diff against.
- E2E cleanup missed audit logs.

### docs-vs-code-contract
→ `checks-docs`
- API docs described the wrong response contracts vs the actual route handlers.
- An enum list in the docs was missing a value that exists in the type definition.
- An issue marked "pending" in docs was actually CLOSED in the tracker.
- The root README stayed stale after a nested README was fixed; a roadmap doc outside the diff had a stale checkbox. **State changes must propagate beyond the diff — `grep -r` the repo.**

### consumed-vs-fetched
→ `checks-data-sql`, `checks-contracts-types`
- A flag was fetched but never consumed in the filter.
- `amount_b ?? amount_a` silently mixed currencies.
- A "previous tier" field showed the contract name, not the tier.
- A column referenced in code did not exist in the database (a name mismatch the schema would have revealed).

### driver-quirks-at-the-boundary
→ `checks-infra-rollout`, `checks-contracts-types`
- Postgres `numeric` columns returned as strings by the driver — a `typeof !== 'number'` guard rejects valid data.
- `Intl.NumberFormat` does not validate all ISO 4217 codes — `XYZ` renders without error; only malformed codes throw.
- A doc comment referenced a private function as if it were a shared export — the comment lied about the API surface.

### schema-audit-parser-blind-spots
→ `checks-data-sql`
- A schema-audit parser missed bare quoted columns (`SELECT "Id" FROM schema."Table"`) — the exact pattern behind the original bugs.
- An alias map was global per file, not per SQL block — alias reuse across queries resolved to the wrong table.

### centralized-degraded-contract
→ `checks-contracts-types`
- A shared helper is required so every route returns the same degraded-response shape; without it, each route invents its own.
- A `code` discriminator on a `200`-with-degraded response lets automated clients pick a backoff strategy without parsing arrays.

### type-system-rigor
→ `checks-contracts-types`
- `[key: string]: unknown` in an interface nullifies type-checking — the index signature swallows access to non-existent fields with no error.
- `Omit<T, K> & { [K]: never }` marks fields absent on partial paths (`never` collapses to `undefined` while keeping assignability).
- `@ts-expect-error` as a contract pin — if a future refactor loosens the guarantee, the expected error vanishes and the build breaks (intentionally).

### design-doc-precedes-write
→ `checks-contracts-types`
- Writeback design docs (which store is writable, which is read-only, which receives data via sync) must precede write code — it prevents a writer aimed at the wrong destination.

### config-fallback-rbac-type-leak
→ `checks-security-rbac`, `checks-contracts-types`
- A security toggle with a silent fallback on an invalid value is a risk: `RBAC_MODE=enforced` vs an `enforcing` typo must fail loud, not default open.
- A raw decision type (`'allow' | 'deny'`) must be separated from the final decision type (which includes `shadow_deny`); leaking the raw type into a public return is a bypass vector.
- A config parser must live where the design doc says, not inline; per-feature env var names must match the documented convention (`RBAC_MODE_CONTACT` vs `RBAC_MODE_CONTACTS`).

### cache-design-baselines
→ `checks-contracts-types`
- A cache design doc must declare target metrics (p50, hit rate, cost) — without a baseline there is no way to measure success.

### shared-cache-delete-sentinel
→ `checks-contracts-types`, `checks-infra-rollout`
- A shared cache must not contain private data — exclude private notes before writing to it.
- `FieldValue.delete()` in `set(..., { merge: false })` is a no-op — the store ignores delete sentinels on non-merge writes.

### fire-and-forget-under-scale-down
→ `checks-concurrency-state`, `checks-infra-rollout`
- Stale-while-revalidate changed the *meaning* of `cached` / `generatedAt`: a semantic breaking change even though the type did not move.
- `invalidate()` must force a synchronous rebuild on the next read, not SWR — the write path needs correctness; only TTL expiry tolerates staleness.
- Serverless scale-down kills fire-and-forget — post-write invalidation must be awaited (~20–50ms is acceptable).
- Invalidation coverage must hit *every* write path in the design doc (PATCH/DELETE, uploads), not just the obvious ones.

### structured-logging-shape
→ `checks-infra-rollout`
- `console.warn('prefix:', JSON.stringify({...}))` is not a structured log — structured-logging backends only promote a JSON payload when the whole line is valid JSON; a text prefix forces a plain-text payload.
- Telemetry sites in the same project must share one shape (`severity` + `component` + `event` + spread) or observability queries fragment.

### incomplete-field-fix-coverage
→ `checks-data-sql`, `checks-pipeline-data`, `checks-error-observability`, `checks-docs`
- A field fix covered 5 of several query blocks but missed another query and the runtime engine module, which consume the same fields unfiltered — incomplete coverage, false confidence.
- A regex `^[a-zA-Z0-9]{15,18}$` accepted wrong lengths/prefixes; a known ID format with a fixed prefix needs an anchored pattern (`^003[a-zA-Z0-9]{12}([a-zA-Z0-9]{3})?$`).
- Rows dropped by the regex with no log/counter — an upstream regression corrupting 100% of data empties the UI silently.
- A WHY comment present in one language port but not the others — asymmetry invites a DRY cleanup that removes the protection.
- A pre-existing `.catch(() => [])` on an adjacent query swallowed syntax errors from the new regex with no signal.

### the-multi-miss-review
→ `checks-ai-prompt`, `checks-security-rbac`, `checks-error-observability`, `checks-ui`, `checks-contracts-types`, `checks-infra-rollout`
- A reference ID was accepted without checking ownership of the referenced doc — a viewer reads another owner's snapshot straight into the LLM prompt. **Secondary-resource RBAC.**
- A `target: 0` was hardcoded and flagged as an "owner decision" when it corrupts the prompt — invalid data feeding an AI is a **blocker, not a flag**.
- An auth resolver sat outside the try/catch on several routes — a backend failure returns a bare 500 with no log.
- A silent `catch {}` on an auto-pick swallowed a store outage.
- A UI action only `console.error`'d with no error state — the user never learns it failed.
- `Promise.allSettled` turned a 503 into an empty array — the pipeline looks healthy during an outage.
- The LLM SDK's `maxRetries` was left at default — 3 retries × 45s timeout = 135s wall-clock, past a 100s proxy limit.
- A persist failure after a successful LLM call — tokens spent, user gets a 500, output lost; a best-effort write propagated its error instead of being swallowed deliberately.
- `coverage: number | null` + `coverageBasis: string` allowed illegal states (should be a discriminated union); the UI redeclared response types instead of importing them (contract drift).
- A manager/admin saw only their own data because the summary route always pinned the owner ID — the contract said full access, the implementation restricted it.

### list-vs-per-resource-rbac
→ `checks-security-rbac`
- An allow-list helper returned `true` for `(undefined, [])` (orphan pass-through), but an allow-list of `[]` should mean deny-all. A user without an ID saw orphan items in the list but got 404 on the per-item routes — list-layer and per-resource RBAC disagreed end-to-end.

### atomicity-rbac-every-write
→ `checks-concurrency-state`, `checks-security-rbac`, `checks-data-sql`, `checks-contracts-types`, `checks-tests`
- An overlay PATCH wrote the overlay + audit as sequential writes (no `WriteBatch`) — a mid-failure leaves a partial audit with no signal. A sibling route used a batch; atomicity was not checked here.
- An approve/decline PATCH had no transaction — race between concurrent managers (state-machine concurrency).
- A POST create-route had no access check — any authenticated user could create a request. RBAC was not checked on **every** write route.
- A CTE lacked `GROUP BY` — runtime failure in Postgres; only the sibling getter was tested.
- An aggregate lacked an `end_date > today` filter — expired rows counted as active (not compared to the precedent pipeline).
- A request type was not a discriminated union — allows `status=approved` with `decidedBy=undefined`; those fields were not asserted in approval tests (could delete the write and tests still pass).
- A timestamp returned as an object instead of an ISO string — sort breaks, contract lies; flagged as "recommended" when it was a blocker (the repo rules were explicit).
- Shadow-mode had no test on the new route, though sibling precedents cover it.

### pipeline-is-not-runtime
→ `checks-pipeline-data`
- A fix corrected the materialized table, but the runtime engine read a raw source directly — the pipeline fix never delivered the promised uplift.
- The PR's business claim ("X can finally fire") was not confronted against the code — the engine did not use the corrected table.
- A parallel source carried the same bug, masked by a **false "already canonical" comment**.
- A new coverage check was introduced without considering existing data, which would break the build.

### new-status-no-lifecycle
→ `checks-pipeline-data`, `checks-contracts-types`, `checks-tests`
- A PR wrote `status='needs_review'` but no consumer reads it: the review UI filters `pending`, the approve/reject API accepts only `pending` — rows fall into a black hole. The reviewer verified the downstream filter excluded `needs_review` and *framed it as a positive*, when that was the problem.
- An external writeback was silently disabled for rows that used to carry the old status.
- A `non_canonical` counter changed semantics silently (aliases stop counting) — masks real drift.
- Doctests were decorative (CI does not run them); no cross-language sync check; a field-name typo passed as canonical with no log.

### silent-fallback-publish-validation
→ `checks-error-observability`, `checks-ai-prompt`, `checks-pipeline-data`
- Silent fallback = zero observability: a log that only fires on the happy (non-null) path never catches the dangerous case (everything in fallback). **Invert the condition.**
- A literal `catch {}` with no log = silent degradation: the worker proceeds without data and marks success — an empty catch on a critical path is a **blocker**, not a recommendation.
- Publish-time validation did not verify the template's required placeholders — an admin can publish a prompt that breaks the worker.
- A feature bypassed the registry, so admin edits did not reach one consumer (grep **all** consumers of the old prompt).

### rollout-sequencing-sqlstate
→ `checks-infra-rollout`
- A deploy job ran `--snapshots-only` **without** `--skip-validation` — merging before the prerequisite layer (a materialized-view refresh) turns the job red continuously. The escape hatch existed but was not wired to the runner.
- `psycopg2.errors.FeatureNotSupported` (0A000) does **not** catch `OBJECT_NOT_IN_PREREQUISITE_STATE` (55000), the real error when `REFRESH MATERIALIZED VIEW CONCURRENTLY` runs without a unique index. `pg_index` was not verified before approving the fallback logic.
- A material miss **outside** the PR: an external message to the infra team claimed "all 9 materialized views already have the unique index (validated)" when the index query had not been run. **"I validated" requires the query/command to have run in the same session.**

### advisor-vs-operator
→ `advisor-vs-operator`
- A dev opened a PR with several manual infra items (provision a secret, create a dataset + table in a virgin project, a cross-project IAM binding, a scheduled job). Technical access (an owner role) was mistaken for authorization to execute. Schema/DDL in a domain with a clear operational owner = **never** operator, even with owner access. When a PR says "don't merge yet," default to advisor mode. Full detail in `advisor-vs-operator.md`.
