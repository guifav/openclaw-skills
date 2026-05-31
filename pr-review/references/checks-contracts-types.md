# Contracts & Types Checks

Open this when the PR touches types, interfaces, or API contracts — discriminated unions, type redeclaration drift, design-doc compliance, and lifecycle completeness.

---

## Correctness & Regressions

- Does the code implement the behavior described in the PR? Does it resolve the declared issue?
- Are there regressions in existing flows?
- Do fallbacks work as promised? Are fallbacks deterministic?
- Are null, empty, and absent cases handled? `null` vs `undefined` vs `0` vs `false` vs `""` must be differentiated correctly.
- Is backward compatibility maintained with old payloads?
- Are conditional branches actually covered by tests?
- Do UI and API fail safely in degraded paths?
- **`FieldValue.delete()` in `set(..., { merge: false })` is a no-op.** Only works with `{ merge: true }` or `update()`. → lessons-ledger.md
- **`Intl.NumberFormat` accepts structurally valid but non-existent ISO 4217 codes** (`XYZ`) without throwing — only malformed codes throw `RangeError`. → lessons-ledger.md

---

## Code Contracts

- Do TypeScript interfaces reflect the real returned shape — not aspirational or approximate shapes?
- Do field names communicate correct semantics? (`previousTier` showing contract name, not tier = **P1**) → lessons-ledger.md
- Is there mismatch between backend and frontend shapes?
- Are new fields propagated correctly end-to-end?
- Are renamed fields updated in ALL consumers — not just the route that changed?
- Are enums, unions, and literals centralized? No per-route redefinitions.
- Did the public route contract change? Is it documented?
- Does "partial data" look different from "zero data" in the type? A `number | null` field that is always `null` "by default" is a silent lie.
- **`[key: string]: unknown` in an interface nullifies type-checking** — index signature swallows access to non-existent fields without error. → lessons-ledger.md
- **Illegal states via loose types.** `coverage: number | null` + `coverageBasis: string` permits `coverage=null` with a non-null basis. Use discriminated unions. → lessons-ledger.md

### Type-level enforcement > comment promises — **P1**

- If a field "must never have value X", enforce it via type, not comment.
- Path that "will never have field Y" → use `{ [Y]?: never }`, not omission. `never` collapses to `undefined` while maintaining assignability.
- `@ts-expect-error` as a contract pin: if a future refactor loosens the guarantee, the expected error disappears and the build breaks — that is the signal. → lessons-ledger.md
- `RbacRawDecision = 'allow' | 'deny'` vs `RbacDecision` (which includes `shadow_deny`) — the raw type leaking into public return is a security bypass vector. → lessons-ledger.md
- Comments lie; types compile.

---

## Internal Integration

- Do API routes use the correct helpers? Does changing one helper accidentally alter behavior in other routes?
- Does the frontend consume new fields correctly?
- Do child components receive consistent props?
- Do test mocks match the real contract — not a stale or partial shape?
- Do pure functions remain pure? Are side effects in the right layer?
- Are imports following repo patterns?
- **Dangerous logic duplication** — same field-parsing logic in 3+ routes without a shared helper. → lessons-ledger.md
- **Naming convention classifiers** (e.g. `crm_*` → an external CRM, `internal_*` → your organization) must be centralized. Per-route regex diverges silently. → lessons-ledger.md

---

## Design Doc Compliance — **P1 if violated**

- If the PR implements a previously approved design doc, cross-check EVERY numbered decision (D1-D5, C1-C8, etc.) against the implementation.
- Env var names, function locations (`lib/env.ts` vs inline), rollout strategy, and schema shapes must match the approved text.
- **Divergence from approved design is a blocker, not a nit** — the design was approved as a contract.
- Silently skipping items the design doc marks as "Branch N+1" is also a blocker — deferred work must be explicitly deferred, not invisible.
- `parseRbacMode` must live where the design doc says (`lib/env.ts`), not inline in the domain module. → lessons-ledger.md
- Per-feature env var names must match the documented convention — `RBAC_MODE_CONTACT` (singular) vs `RBAC_MODE_CONTACTS` (plural) breaks granular rollout. → lessons-ledger.md

---

## Cross-Cutting Helper Extraction

- If 3+ routes implement the same pattern (degraded response format, error classification, retry headers), verify the PR extracts to a shared helper.
- If a helper already exists (`lib/degraded.ts`), verify the PR uses it instead of reimplementing.
- Verify the helper covers ALL call sites, not just the ones the PR touches.

---

## Firestore Timestamp vs ISO String Contract — **P1**

When a TypeScript interface declares a field as `string` (ISO), but the Firestore write uses `FieldValue.serverTimestamp()`:

- The GET response returns a Firestore Timestamp object, not a string.
- This breaks: JSON serialization, string comparisons, sort operations, and the API contract.
- Check the repo's agent-instruction file (e.g. `AGENTS.md`): if it says "convert with `.toDate()?.toISOString()` when returning to the client", any route that returns `snap.data()` directly with serverTimestamp fields is a **P1 blocker**.
- `a.submittedAt < b.submittedAt` is undefined behavior on Timestamp objects.
- → lessons-ledger.md

---

## Precedent Comparison — **P1 if diverging without justification**

When a PR implements a new workspace that mirrors existing ones, systematically compare each pattern:

- **Atomicity**: does the precedent use `WriteBatch`/transactions? Does this PR? → lessons-ledger.md
- **Timestamp handling**: how does the precedent convert `serverTimestamp` to ISO?
- **RBAC**: does the precedent have shadow-mode tests? Does this PR?
- **SQL**: does the precedent filter inside CTEs? Does this PR?
- **Error handling**: does the precedent have `Retry-After` on 503? Does this PR?
- Divergence from precedent without documented justification is a **P1 blocker**.

---

## Type Redeclaration Across Boundaries — **P1**

- When the client (UI) redeclares response interfaces (`SummaryResponse`, `StaleResponse`) instead of importing from `lib/types` or `lib/pipeline/types`, any backend change that doesn't update the client creates silent drift.
- **Verify: are API response types imported, not redeclared?** → lessons-ledger.md

---

## New Status/State Values — Full Lifecycle Trace — **P0**

When a PR introduces a new status value (e.g., `needs_review`, `suspended`, `archived`), trace the FULL lifecycle. Any missing step = data black hole = **P0 silent feature-death**:

1. **Write**: where is the new status written? (extractor, API route, cron job)
2. **Read/Display**: where is the new status shown to users? (UI filter, admin panel, API response)
3. **Transition**: where can the status be changed? (approve/reject routes, admin actions)
4. **Filter**: where is the status filtered out? (downstream consumers, pipelines, scoring)

**Critical trap**: a downstream filter that *excludes* the new status is NOT a positive — it IS the problem. Do not frame it as "good, the filter handles it." That filter makes the rows invisible and unactionable.

**Also check**: does the new status silently disable external integrations? If `auto_approved` rows fed a downstream writeback to an external CRM, and the PR now routes those rows to `needs_review`, the writeback is silently disabled for high-confidence rows. The PR body must document this.

→ lessons-ledger.md

---

## Counter/Metric Semantic Changes — **P1**

- When a PR changes what a counter counts (via alias exceptions, filter changes, or vocabulary normalization), the counter's meaning changes silently. Historical comparisons become invalid.
- **Example**: `non_canonical` counter drops overnight after adding alias exceptions — not because canonicalization improved, but because the metric definition changed. Masks real drift.
- Require: (a) rename the counter, (b) split into two (`non_canonical` + `alias_hits`), or (c) explicit documentation in stats output ("non-canonical excl. known aliases").
- → lessons-ledger.md
