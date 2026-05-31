# Error Handling & Observability Catalog

Open this when the PR touches error handling, fallback paths, logging, data pipelines, or any catch block — especially on critical or external-call paths.

---

## Boundary Error Handling — Pre-Try/Catch Zone

### Rule (P1)
Trace every external call (PG, Firestore, API, resolver) in the route handler. Any external call **before the first `try/catch`** is a naked 500 waiting to happen: no log, no `Retry-After`, no structured error code.

Common blind spots: viewer-ID resolution, identity cache writes, Firestore reads for enrichment.

```bash
# Find all external calls in a route file and check their position relative to try/catch
grep -n 'await\|fetch\|Firestore\|query\|resolve' app/api/<route>/route.ts
grep -n 'try {' app/api/<route>/route.ts
```

If any `await externalCall()` appears at the top level of the handler (before the opening `try {`), that is a **P1 blocker**.

**Best-effort side-effects** (cache writes, telemetry, identity-cache updates) must each have their own `try/catch` and must NOT propagate to the response. A cache write failure must not crash the request.

→ lessons-ledger.md

---

## Empty Catch Blocks in Data-Pipeline Paths

### Rule (P0)
`catch {}` or `catch { // proceed without it }` in a data-pipeline path is a **P0 blocker**, not a recommendation.

When a worker catches an error and continues, the output is silently degraded:
- HTML dashboard regenerated without original layout (object-storage read failed silently)
- AI prompt built without source data (file parse failed silently)
- Refresh marked "completed" when the result is fundamentally different from the original

**Fix:** either fail the job (preferred for data integrity) or annotate warnings on the output document so the UI can distinguish "clean refresh" from "degraded refresh."

`catch (() => {})` on a meta-operation (logging the failure itself) is doubly dangerous — silent failure of the failure reporter.

```bash
grep -n 'catch {}' workers/ scripts/ lib/pipeline*.ts
grep -n 'catch.*proceed\|catch.*continue\|catch.*ignore' workers/ scripts/
```

→ lessons-ledger.md

---

## Preexisting `.catch(() => default)` — Active Tech Debt

### Rule (P1 when touched by the PR; P2 otherwise)
`.catch(() => [])` and `.catch(() => null)` swallow **all** errors silently: syntax errors, timeouts, auth failures, rate limits. They return "no data" indistinguishably from "empty data."

When the PR diff touches a function that has this pattern, flag it — a bug in new code (a bad regex, a new Firestore rule) would return an empty array with no signal.

**Replace with:**
```ts
.catch((e) => {
  console.warn(JSON.stringify({ severity: 'WARNING', component: 'queryFoo', error: String(e) }));
  return [];
})
```

```bash
grep -n '\.catch(() => \[\])\|\.catch(() => null)\|\.catch(() => {})' lib/ app/api/
```

→ lessons-ledger.md

---

## Drop/Filter Observability

### Rule (P0)
When a filter silently discards rows (invalid external ID, malformed data, out-of-range value), the absence of data is indistinguishable from legitimate empty results. A regression that corrupts 100% of upstream data produces an empty UI with zero alarm.

**A filter without logging is a blocker.**

Required observable behavior:
1. A `warn` log fires when rows are dropped — with a **count** (not raw payload, to avoid PII).
2. The log is structured JSON (not swallowed by `.catch(() => [])`).
3. There is a mechanism to detect when drop rate exceeds a threshold (100% drop = empty table with no metric spike).

```bash
grep -n 'filter\|drop\|discard\|invalid' lib/pipeline*.ts scripts/
grep -n 'log\|warn\|metric' lib/pipeline*.ts scripts/
# Verify every filter has a corresponding log
```

→ lessons-ledger.md

---

## Silent Fallback Observability

### Rule (P1)
When a feature has a fallback path (hardcoded default, cached version, degraded mode), the fallback must be **observable**:

1. A log/warn fires **when the fallback activates** — not only on the happy path.
2. The UI shows a distinguishable indicator ("using fallback version", degraded badge) if the operator needs to know.
3. **Invert the logging condition:** log when fallback fires, not when the real version fires.

**The danger case:** operator publishes a new version; the UI shows "v3 active"; production silently uses the fallback. Nobody knows.

```ts
// WRONG — only logs when happy path succeeds
if (version !== null) {
  log({ event: 'version_loaded', version });
}

// CORRECT — logs when fallback fires
if (version === null) {
  log({ severity: 'WARNING', event: 'fallback_active', reason: 'version_null' });
}
```

A cache that silences repeated fallback warnings (e.g., 60s TTL) must still log at least once per cache cycle.

```bash
grep -n 'fallback\|version.*null\|null.*version' app/api/<route>/route.ts lib/
```

→ lessons-ledger.md

---

## Error Handling — Structural Completeness

### External errors caught at the right level (P1)
- Partial failures must not crash the route when they shouldn't (degraded mode is valid).
- Critical failures must crash when they should — `Promise.allSettled` that maps rejected → empty array makes an outage look like healthy empty state.
- Logs must have sufficient context (route, viewer, resource ID) but must not leak secrets or sensitive data.

### HTTP semantics (P2)
- Errors must return appropriate HTTP status codes (400 vs 422 vs 500 vs 503).
- 503s must include a `Retry-After` header consistent with the retry budget.

### "Best-effort" paths (P1)
A "best-effort" operation without a recovery path is a **silent failure**. Verify:
- The retry CTA exists in the UI.
- Or the operation is truly fire-and-forget with a documented acceptable failure rate.

```bash
grep -n 'allSettled\|Promise\.allSettled' app/api/ lib/
grep -n '"best.effort"\|bestEffort\|fire.and.forget' app/api/ lib/
```

### Resilience (P2)
- External dependencies (analytics service, PG, Firestore) must be able to fail independently without cascading.
- "No data" and "source unavailable" must be distinguishable in both the API response and the UI.
- Defaults must be safe — a fallback that shows `£0` as a real value is worse than showing an error.
- Timeouts must stay within the infra budget (`timeout × (maxRetries + 1) < platform request timeout`, e.g. Cloud Run).
- Incidents must be diagnosable from structured logs alone.

```bash
grep -n 'timeout\|maxRetries\|retry' lib/ app/api/
```
