# Infra & Rollout Checks

Open this when the PR touches infra / deploy / migrations / container jobs.

---

## Sequential Branch Rollout

In multi-branch rollouts (Branch 1 skeleton → Branch 2 rules → Branch 3 wiring):

- Branch N-1 must already be in `main` before reviewing Branch N. **P1.**
- Branch N must be self-contained — it does not break if merged alone and Branch N+1 never lands.
- No follow-up is silently critical ("deferred" but actually blocks production safety).
- The skeleton branch must have zero behavior change — import it, and nothing happens until the next branch.

---

## New Validation Enabled by Default vs Current Deploy-Job Args (Mandatory)

When a PR introduces a new validation or gate that defaults to enabled:

1. Check what args the running deploy job actually invokes today. For example, on Cloud Run:
   ```
   gcloud run jobs describe <job> --format="value(spec.template.spec.template.spec.containers[0].args)"
   ```
2. Compare the PR's default behavior against those args.
3. If the validation will fire immediately on next deploy AND the data state is known to fail the threshold (e.g., PR body says "current divergence is 22%"), this is a **silent rollout footgun — P0**. The job goes red continuously until a separate unrelated change lands.

The PR body must declare explicit rollout coordination. "Escape hatch exists (`--skip-validation`)" is NOT sufficient mitigation if the escape hatch is not actually plumbed into the runner. Either the job args must be updated in the same PR, or the validation must default to warn-only mode until cutover.

> Example: a PR introduced `--snapshots-only` logic with a threshold check that defaulted to blocking. The running job invoked `--snapshots-only` WITHOUT `--skip-validation`. Merging would have turned the job red on every run. The escape hatch existed but was not wired to the runner. → lessons-ledger.md

---

## Driver Exception Class vs Postgres SQLSTATE (Mandatory)

When code catches a specific `psycopg2.errors.<Class>` to fall back to a degraded mode, verify the exception class matches the SQLSTATE the database actually raises.

**Critical mismatch:**
- `REFRESH MATERIALIZED VIEW CONCURRENTLY` without a UNIQUE index raises `OBJECT_NOT_IN_PREREQUISITE_STATE` (SQLSTATE **55000**), NOT `FEATURE_NOT_SUPPORTED` (SQLSTATE **0A000**).
- A `try/except FeatureNotSupported` does NOT catch it — the fallback silently never fires. **P0.**

**Other common SQLSTATE mismatches to check:**

| Caught class | SQLSTATE | Actual error to catch | SQLSTATE |
|---|---|---|---|
| `FeatureNotSupported` | 0A000 | `ObjectNotInPrerequisiteState` | 55000 |
| `UndefinedTable` | 42P01 | `UndefinedObject` | 42704 |
| `SerializationFailure` | 40001 | `DeadlockDetected` | 40P01 |
| `QueryCanceled` | 57014 | `AdminShutdown` | 57P01 |

**Validate before approval**: force the failure mode in a smoke test, or inspect Postgres docs/source, and confirm the SQLSTATE actually fires the fallback path. → lessons-ledger.md

---

## Verify pg_index Before Asserting CONCURRENTLY Works (Mandatory)

When the fix or remediation plan involves `REFRESH MATERIALIZED VIEW CONCURRENTLY`, query `pg_index` to confirm each materialized view has at least one unique index. Without it, CONCURRENTLY fails at runtime with SQLSTATE 55000.

**Run this query first:**
```sql
SELECT c.relname,
       COUNT(*) FILTER (WHERE i.indisunique) AS uniq
FROM pg_class c
LEFT JOIN pg_index i ON i.indrelid = c.oid
WHERE c.relkind = 'm' AND c.relname LIKE '<pattern>'
GROUP BY c.relname
```

If `uniq < 1` for any MV, the fix must either:
- (a) Drop `CONCURRENTLY` (note: this blocks readers during refresh), or
- (b) Add the unique index first, in a separate migration.

**This check is also required BEFORE writing claims in external comms** (emails to infra, issue bodies, PR descriptions, chat messages). Writing "I verified" or "confirmed" in an external comm when the corresponding SQL/command was NOT actually executed in the same session is fabricated confidence. When that message reaches the infra team, credibility is lost.

Rule: if you write "verified" about a pg_index check, the query above must have been run in the current session, with results visible. → lessons-ledger.md

---

## Driver and Platform Quirks at Boundaries

Each driver has quirks that TypeScript alone does not catch. Verify at every DB/API boundary:

- **PG `numeric` columns**: returned as JS `string` by `node-postgres`, not `number`. A `typeof value !== 'number'` guard rejects valid data. Cast at the query boundary: `::double precision`.
  - `grep -n 'numeric\|::' <query-file>` to locate cast points. → lessons-ledger.md
- **Firestore `FieldValue.delete()`**: silently ignored in `set(..., { merge: false })`. Only works with `{ merge: true }` or `update()`. A `merge: false` set with a delete sentinel leaves the field intact with no error.
  - `grep -n 'FieldValue.delete\|merge: false' <file>` → lessons-ledger.md
- **`Intl.NumberFormat` and ISO 4217**: accepts structurally valid but non-existent currency codes (`XYZ`) without throwing. Only malformed codes throw `RangeError`. Currency validation must go beyond `Intl.NumberFormat` constructor success. → lessons-ledger.md
- **Serverless scale-down kills fire-and-forget**: on platforms that scale to zero (e.g., Cloud Run), post-write side effects (cache invalidation, analytics) must be `await`ed, not background. ~20–50ms extra latency is acceptable; silent data inconsistency is not. **P1.**
  - `grep -n 'invalidat\|analytics\|telemetry' <route-file>` — confirm each is awaited. → lessons-ledger.md
- **Structured log parsing**: `console.warn('prefix:', JSON.stringify({...}))` breaks `jsonPayload` promotion in JSON-based log backends (e.g., Cloud Logging). The entire log line must be pure JSON for the backend to parse it as structured. Prefixed text forces `textPayload`.
  - Correct shape: `{severity, component, event, ...entry}` as a single JSON line, no prefix. → lessons-ledger.md

When in doubt: write a test that exercises the exact boundary behavior.

---

## Latency Budget

- Do external calls run in parallel when possible? Sequential calls where parallelism is safe are a **P2** improvement.
- Do existing timeouts cover new calls? Confirm new external calls fit within the request timeout of the serving platform and any reverse-proxy / CDN timeout in front of it.
- **SDK default retry multiplication**: when the PR configures a timeout for an external SDK, check the SDK's default `maxRetries`. `timeout × (maxRetries + 1)` must fit within the infra budget. If not, set `maxRetries: 0` explicitly.
  - Example: 45s timeout × (default 3 retries + 1) = 180s wall-clock > a 100s proxy limit. → lessons-ledger.md
- Are loops bounded? Do payloads have reasonable limits? Does truncation happen before sending large data?
- Does the PR obviously increase p95 of critical routes? If yes, flag as **P1** or **P2** depending on the route's SLA.
