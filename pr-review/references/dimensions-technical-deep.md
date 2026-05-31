# PR Review — Deep Technical Dimensions

_Reference checklist: items the technical reviewer (you) MUST evaluate, going beyond surface-level code reading. Many require database access, runtime analysis, or codebase-wide search._

---

## 1. Architecture and codebase idioms
- Does the pattern match how THIS codebase does similar things, or just "generic best practices"?
- Should the new helper live in a different file based on existing conventions?
- Is this the Nth variation of the same pattern — time to generalize?
- Naming inconsistency between similar concepts (degraded vs degradedSources) — intentional or drift?
- Conventions for function granularity in shared files (one per SELECT or grouped by entity?)
- Dependency direction consistency
- **Driver/client config validated against the REAL server, not assumed.** A recognizable pattern (`ssl: { rejectUnauthorized: false }`, retry policy defaults, pool size copied from another service) only works if the server accepts it. PG with `ssl: {...}` forces a TLS handshake; a server with `pg_settings.ssl = off` rejects. Validate the handshake against the prod host before approving driver/pool changes — 1 minute saves multi-day outages. Same applies to TLS versions, auth scopes, timeouts, pool sizes (a read replica vs primary have different `max_connections`).
- **Multi-host audit coverage.** When a workspace uses 2+ hosts (replica + analytics, OLTP + OLAP), `audit:schema --live` or equivalent must connect to all of them. A single-host audit gives false confidence for the host it doesn't reach.
- **Overlay/impersonation/feature-flag coverage:** when a feature swaps a profile field (impersonation overlay, tenant switch, role downgrade), grep TWO things: (a) every consumer of the swapped field, (b) every short-circuit/bypass that reads the OTHER fields. Bypasses based on the un-swapped fields (`profile.role === 'superadmin'`) silently ignore the overlay. Canonical helpers (`decideXxxScope`) are obvious; secondary paths (`resolveXxxOwnerFilter`, ad-hoc query builders, util functions consuming the full profile) are easy to miss. Listing both greps side-by-side catches the gap.
- **Architecture as a gate BEFORE technical review.** When evaluating a PR that introduces a new datastore, new table, new schema location, or any data-home choice — ask architectural questions FIRST, before evaluating tests/correctness/decision-matrix/etc. "Is this in the right architectural location?" supersedes "is the code correct?". Approving a PR with green CI, live parity, and a decision matrix but the wrong architectural home costs hours of rework when the owner rejects it. The signal: if a PR review ends with "approve without reservations" on a non-trivial architectural change, treat it as a red flag — re-verify the architectural criterion before final approval. Examples: a materialized view placed in an analytics-replica host instead of the data warehouse (rejected after approval); a deprecated table re-introduced by a JOIN that the PR was supposed to remove elsewhere.
- **Naming convention as deploy taxonomy.** When a repo has 3+ jobs/scripts/components with the same prefix (`sync-*`, `enrich-*`, `taxonomy-*`), the prefix carries deployment semantics (container image, secrets bundle, allowed dependencies, runtime). Validate that the proposed name matches the semantics: e.g. `sync-*` reads from an external source (Postgres/MySQL/object storage), `enrich-*` reads the warehouse → writes the warehouse. A ticket proposing `sync-tasks` may need to become `enrich-task-aggregates` because the actual work is warehouse→warehouse. Wrong prefix = wrong container image = unnecessary credentials + bloat.

## 2. SQL semantic correctness
- JOIN type: does it maintain expected cardinality (1 row per entity)?
- ORDER BY ties: when end_date has duplicates, which row is "current"? Need a tiebreaker?
- Case-sensitivity of enum/picklist values: 'Activated' vs 'ACTIVATED' vs 'active'
- LEFT() in PG: bytes or characters? Multibyte (accents, emojis) can cut mid-codepoint
- 0 vs NULL vs absent semantics in numeric columns
- LIMIT 1 subquery: which row does it pick? Order is implicit = unstable
- NULL handling: `typeof x === 'boolean'` when the driver maps NULL to null vs undefined
- Index existence for new WHERE clauses (seq scan risk)
- Timezone in date::text casts — with or without Z? Does Node assume UTC or local?
- **PG driver type coercion:** `pg` returns `numeric` columns as `string` by default (no parser/cast). A `typeof value !== 'number'` guard silently rejects valid data. Fix: `::double precision` cast in SQL or accept string coercion in JS.
- **Bare quoted columns in subqueries:** `SELECT "Id" FROM crm."Contact"` without an alias is the most common SQL pattern but the hardest to audit. Any tool/parser that only handles `schema."Table"."Column"` misses these.
- **Schema validation ≠ value validation.** `information_schema.columns` confirms the column exists, not that the string literal in your filter appears in actual rows. For every enum/string-literal filter (`Status IN ('Activated', ...)`, `Type = 'Renewal'`), run `SELECT DISTINCT "column", COUNT(*) FROM table GROUP BY 1` against prod. Schema check is structure; SELECT DISTINCT is content. Both required when the filter is a string literal.
- **Polymorphic foreign-key IDs (e.g. a CRM's `Task.WhatId`, `Task.WhoId`, `Event.WhatId`):** a polymorphic key can point at multiple object types — an Account, an Opportunity, a Case, or a custom object — typically distinguished by an ID prefix. `COALESCE(t.WhatId, c.AccountId)` assumes WhatId is always an Account or null — wrong. Use an explicit prefix guard: `CASE WHEN t.WhatId LIKE '001%' THEN t.WhatId ELSE c.AccountId END`. (Some CRMs encode the object type in the ID prefix — e.g. 001=Account, 003=Contact, 006=Opportunity, 500=Case.) Whatever the system, learn its prefix scheme before writing the guard.
- **Enum filter implicit product decisions.** Every value NOT included in `IN (...)` is a product decision. Excluding `New Business` / `Reactivation` / NULL from an opportunity-type filter is a choice with downstream impact. Require a JSDoc explaining WHY each excluded value is excluded; without it, drift is guaranteed when the source adds a new picklist value.
- **UNION ALL with declared precedence between N paths — validate the GLOBAL property, not just per-input.** When paths have precedence (`AccountId > WhatId > Contact`), the trap is a `WHERE` clause that excludes only the current `$1`: a row whose AccountId points at ANOTHER account but whose Contact links to `$1` leaks into both `$1`'s path 3 AND the other account's path 1. A per-input dedup check (zero duplicates for the same `$1`) WON'T catch this. Global check: the same row shouldn't appear in different `$X` / `$Y` inputs. Fix pattern: Path N's WHERE excludes rows already claimed by Paths 1..N-1 **for any account**, not just the current parameter. Test guard: `expect(cte).not.toMatch(/<> \$1/)` blocks a revert to the buggy lazy form.
- **`CREATE OR REPLACE TABLE` silent empty failure.** A successful CREATE that produces an empty table means the query returned 0 rows — but the warehouse job marks SUCCESS, the cron marks `ok`, and consumers see empty counters. Always add post-write COUNT validation: `if row_count == 0: emit_event(status='failed') + exit(1)`. Logging/alerting triggers only on the post-write check, not the CREATE itself.
- **Bug regression that re-introduces the root cause it removed.** A PR migrating off a broken view `v_X` (which referenced a missing table `T`) may RE-introduce `T` in another JOIN. The reviewer must grep the migrated PR for ALL tables in the original broken view, not just the view name. Pattern: when removing a broken view, list every table it referenced and verify none reappears in the replacement queries. Example: `v_contact_360` referenced `dim_company` (missing in a region), the PR removed the view but the new `churn` query added `LEFT JOIN dim_company`. Same 404 at runtime, different code path.
- **Schema drift between sibling tables that should look alike.** Tables joined via UNION ALL or treated uniformly in code may have inconsistent column names. Example: one table uses `data_criacao/data_alteracao` (a legacy native naming) while sibling tables use `created_at/updated_at`. UNION ALL silently misaligns columns by position when alias names match but the underlying columns differ. Always verify column existence per-table via `INFORMATION_SCHEMA.COLUMNS`, even (especially) when tables look similar by name. Run for each table the UNION touches: `SELECT column_name FROM information_schema.columns WHERE table_name = '...'`.
- **Auto-populated FKs:** some CRMs populate a direct FK (e.g. `Task.AccountId`) automatically when a related lookup resolves — but only for a fraction of rows (say ~56% of Tasks). Rows reached via a different lookup, or without the related link, leave it NULL. If a query uses a polymorphic `CASE WhatId LIKE prefix THEN ... ELSE Contact.AccountId END`, it is likely missing the direct-FK path. Use the direct FK as primary (it's indexed!) + the polymorphic fallback for the remaining rows. The same caution applies to any "populated automatically when X" field — never assume it's always populated.

## 3. TypeScript gotchas
- Index signature `[key: string]: unknown` escaping typed fields (typos become silent unknown)
- Related optional fields not expressed in the type (previousMembership without previousMembershipEndDate is useless)
- Optional in the consumer but always present in the producer — forces unnecessary checks
- 0 as a legitimate value hidden by `> 0` checks
- undefined vs false vs true semantic significance in boolean optionals
- Type assertions (`as { message?: string }`) vs proper validation
- `Promise.allSettled` tuple inference quality
- Missing discriminated unions
- **Error matcher anchoring: `startsWith` vs `includes`.** Detecting a specific error class via a message substring is fragile. `msg.includes('Not found: Table')` matches genuine 404s AND IAM-403 wrapped errors that mention the phrase in stack traces (e.g. an SDK retry wrapping a 403 into a higher-level error with prior attempts in the message). Use `msg.startsWith('Not found: Table ')` (with trailing space — anchored at the start) + a `code` check (404 / '404' / 'NOT_FOUND'). The combined anchor + code check keeps the silent fallback defensible. Test guard: an explicit case for an IAM-403-with-phrase-in-stack returning false.

## 4. React/Next.js specifics
- Components defined in the same file as the page — re-render on every parent render? React.memo?
- useState not reset when props change (errored persists when photoUrl changes)
- Native `<img>` bypasses Next.js Image — lost lazy loading, aspect ratio, CLS
- Server vs client component boundary — 'use client' tree implications
- useAuth at page level — re-renders the entire tree on auth change
- Constants outside the component vs inside — convention?
- Array index as key — stable only if order never changes
- Bundle size impact of new imports

## 5. Async, concurrency, lifecycle
- `Promise.allSettled` without an aggregate timeout — the sum of sources can exceed the budget
- No AbortController — a client disconnect doesn't cancel server-side queries
- A document-store `get()` that isn't abortable — wastes quota even if nobody reads the result
- PG pool behavior under concurrent load — driver error sync vs async?
- **A→B→A rapid-switch race in useEffect.** `let cancelled = false; ... .then(if cancelled) return` protects unmount and A→B, but NOT A→B→A. The in-flight fetch of A can land after A is remounted with a different identity and overwrite the fresh state. For any effect that fetches keyed by a dynamic identity (accountId, contactId, search term), prefer `AbortController` over a `cancelled` flag. Pattern: `const ctrl = new AbortController(); fetch(url, {signal: ctrl.signal}); return () => ctrl.abort();`.
- **External-dependency SDK default retry vs orchestrator-level fallback.** Many SDKs (warehouse clients, document stores, LLM clients) retry 3+ times by default on transient errors. With an edge timeout of ~100s and an orchestrator-level fallback (~200ms live alternative), SDK retry (~24s wall-clock before it surfaces) is harmful: by the time the SDK surfaces, the user has already timed out. Pattern: disable in-SDK retries (e.g. `{retryOptions: {autoRetry: false}}`, `{maxRetries: 0}`) when the orchestrator decides the fallback policy. Document this in a module-level docstring.
- **Module-level singleton + lazy-init.** External clients (warehouse, LLM, document store) hide overhead in first-call setup. Across hot-reload + repeated route-handler invocations, each call allocates a fresh client = connection pooling broken. Pattern: `let _client: T | null = null; function getClient() { if (_client) return _client; _client = new T(...); return _client; }`. A client instantiated inside a request handler = bug.
- **A statement-timeout band-aid can mask pool exhaustion.** When raising `PG_STATEMENT_TIMEOUT_MS=10000 → 30000` fixes one query timeout but a different error reappears, observe carefully: `canceling statement due to statement timeout` = single-query slow (statement budget); `timeout exceeded when trying to connect` = pool exhaustion (concurrent connections). Same error surface in app code, different fix: raise `PG_POOL_MAX`, not the statement timeout. Validate the host's `max_connections` before deciding pool size.

## 6. Performance (eyeball)
- Duplicate iteration (filter then reduce when one pass suffices)
- A PG pool of 5 as a bottleneck with 4 parallel queries per request
- `new Intl.NumberFormat` per render call — should be a module-level const
- Response payload size growth with new fields — does compression handle it?
- Conditional spread micro-optimization that hurts readability

## 7. Coupling and encapsulation
- Query returning more columns than any caller needs — split into Detail vs Summary?
- Exporting internals for testing — convention?
- File growing past the team limit — split by entity?
- High-coupling symptom: a rename touches route + card + panel + test
- Conceptual duplication across the server/client boundary (BIO_MAX_CHARS vs BIO_DISPLAY_CHARS)
- **Workspace-wide invariants honored in ALL routes, not just the one in this PR.** When a workspace establishes an invariant (universe filter `view ∩ replica ∩ RBAC`, currency conversion, scope check), every endpoint must enforce it. Reviewing one route in isolation misses gaps in sibling routes. Method: list ALL routes of the workspace (including ones not touched by the PR), trace the invariant path in each. A route missing the path is a blocker regardless of whether it's in the current diff.
- **Cache invariants that depend on external state.** A cache validates write-path freshness (TTL, explicit invalidate). But what about invariants the cache PRESUMES that can evolve while the cache is alive? A per-record cache assumes the record is still in the universe; if the record churns out, the cache serves zombie data until TTL. The read path must re-validate external invariants, OR invalidation must be wired to every event that changes them.

## 8. API/contract design
- String literals without type narrowing (degradedSources: string[] vs a union)
- Excessive positional props (14 in one panel component) — object pattern?
- Argument order suggesting the wrong merge direction
- Optional in the consumer but required in the producer
- Error path: always throw vs a Result type
- **JSDoc/comments that reference private functions as if public.** If a comment says "exposed via formatMoney" but formatMoney is a private helper in another file, the comment lies about the API surface. Verify every cross-reference in comments.
- **`Intl.NumberFormat` doesn't validate all ISO 4217 codes.** Valid-looking 3-letter codes (XYZ, ABC) render without error. Only structurally malformed codes (1–2 chars, symbols) throw RangeError. A `catch` block won't detect all currency drift.

## 8b. Parser/tool correctness (when the PR adds a code analysis tool)
- **Does the parser handle the MOST COMMON pattern?** If the tool audits SQL column names but misses `SELECT "Column" FROM schema."Table"` (bare quoted, no alias), it misses the exact pattern that caused the original bugs.
- **Scope of analysis: per-file vs per-block?** A global alias map per file breaks when two queries reuse the same alias for different tables. Each SQL block needs its own scope.
- **False negatives are worse than false positives for a guard tool.** A guard that says "all clear" when it can't see 40% of the references gives false confidence. Document coverage gaps prominently.

## 9. Runtime validation vs type trust
- A generic `queryPG<T>` marks but doesn't validate — a SQL change silently breaks the type
- Falsy checks hiding legitimate zero values
- `new Date('garbage')` returns Invalid Date silently
- A document-store `serverTimestamp()` racing with a read
- No schema validation (e.g. Zod) on an API response in the client

## 10. Error handling philosophy
- Inline string identification vs a structured logger
- Asymmetric logging (some sources log errors, others don't)
- Throw vs return — who catches? `Promise.allSettled` changes the stack
- null return vs throw distinction — the caller can't distinguish
- No error codes/discriminators — string matching is fragile
- **Failure mode rendered as ambiguous data in the UI.** When the backend falls back to a sentinel (`0`, `[]`, `null`) on failure, trace it to the UI. If the user sees "$0 YTD" or "No tasks" and can't distinguish that from a real zero/empty, the fallback is producing operationally wrong decisions silently. Required signal in the response: `<field>Unavailable: true` OR a `degradedSources` tag the UI honors. Required UI: "Unavailable" / banner / chip, never a bare zero.
- **Empty-state vs degraded-state interaction in UI rendering.** When a list endpoint returns 200 with `items: []` + `degradedSources: [...]`, the UI must render the degraded banner BEFORE the empty-state short-circuit. Otherwise the operator sees "No items" and the banner stays hidden. Copy must adapt: "List unavailable" instead of "No items".
- **Orchestrator silent fallback hides bad-permission errors.** If a fetcher throws on a 403 (IAM missing) and the orchestrator has a live-fallback rescue (warehouse → live DB), it serves correct data and shows NO banner — the operator can't distinguish "fast path working" from "100% on the slow path because the warehouse never works". Required: every fallback branch emits a structured log event (`query_failed`, `stale_live_rescued`, `missing_live_rescued`, `error_live_rescued`). A central logging/monitoring dashboard then surfaces the silent degrade. Without these logs, on-call learns about the IAM gap weeks later when an alert finally triggers from some unrelated symptom.
- **A freshness check should not discard already-fetched data on its own failure.** If the aggregate query succeeds (rows in hand) but the freshness/metadata query then fails (network blip, quota, table-not-found on a sidecar), DON'T throw out the rows. Return them with `isStale: true, isMissing: false` and let the orchestrator decide silent-stale vs live fallback. Pattern: wrap freshness in a `readSafe` that catches and degrades freshness, not data.

## 11. Logging and observability
- `console.error` direct vs the team's structured logger
- PII in logs (a raw record ID identifies an individual)
- No traceId/requestId for cross-service debugging
- Log cardinality (per-record-ID labels pollute metrics)
- degradedSources as a natural metric candidate
- **`console.error('[prefix]', obj)` breaks structured-log parsing.** A structured logging backend only promotes the message to `jsonPayload` when the line is pure JSON. Prefixed strings (`[account-management] error: { message: '...' }`) come through as `textPayload`, so filters by `jsonPayload.component=` find nothing. Every error path in route handlers must emit `console.error(JSON.stringify({severity, component, event, account_id, message}))`.
- **Source/event naming inherited via copy-paste.** When a new route inherits a `degradedSources` tag from a sibling route, verify the tag still names the source accurately. A tag copied across routes can describe the original source while the new route fails on a different query. Review behavior AND taxonomy.
- **Zero-traffic features hide errors.** A degraded route with `error_rate > 5%` triggers the usual alerts, but a feature with 5 requests/day and a 100% error rate produces no signal until someone reports it manually. For features dependent on a critical upstream (DB, external API), the alert should fire on an **absolute error count > 0 over a 24h window**, not just on rate.
- **JSON structured logging in success AND failure branches.** Many enrichment/sync scripts log only on errors (`log.error('...')`) or only on success (a final report). For log-based alerts, every run must emit a single-line JSON event: `{"event": "...", "status": "ok|failed|running", "duration_ms": ..., "rows_written": ..., "bytes_billed": ...}`. Failed status with `error_message` truncated to ~500 chars. Without a uniform shape, log-based metrics can't differentiate "didn't run" from "ran with 0 rows" from "ran with an error".
- **`bytes_billed` as a cost signal in logs.** When warehouse jobs emit `bytes_processed` and `bytes_billed` per run, divide by 1MB to get a human-readable cost dashboard. A regression in query shape doubling `bytes_billed` = a $$ cost regression invisible without the metric. Recommended: emit MB explicitly alongside the raw bytes.
- **Centralized monitoring beats per-feature alert policies.** Don't create one logging alert + notification channel per new cron/job. Use existing project-level dashboards (Grafana, your cloud provider's monitoring, etc.) that auto-discover jobs by convention. Adding a job to the central dashboard's `CATEGORY_MAP` (a 1-line PR) is simpler than a 3-step monitoring-policy creation. Pattern: when proposing alerting, first check whether the project already has a central monitoring tool — add the job there instead.

## 12. Test design quality
- Partial mock shapes — prod returns extra fields the test doesn't cover
- Missing interaction tests (what if source A AND a subscription both fail?)
- Tests coupled to the exact response shape — any rename breaks N tests
- No property-based tests for pure functions
- Mock chain fragility (`db.collection().where().orderBy().limit().get()`)

## 13. Magic numbers
- Difference between server truncation (600) and client display (400) — why the 200 gap?
- LIMIT 50 for subscription history — overkill?
- `consecutiveYears >= 2` threshold — why not 1?
- Palette size (8 colors) — why not 4?

## 14. Anti-patterns implicitly adopted
- Index signature as a disguised `any`
- Interface bloat (8 optional columns added to the same type)
- String-concatenated error messages vs structured
- String overloading (a single field carries identity AND status)
- Tooltip revealing internal column names

## 15. Backwards compatibility
- Wire-breaking renames (`previousTier` -> `previousMembership`)
- New optional fields are safe additions
- Does downstream-consumer type narrowing still compile?
- Do test fixtures need updating?

## 16. IAM and permission grants
- **Cloud IAM grants come in pairs.** Granting a data-read role on a dataset is necessary but not sufficient — the service account also needs a "job/query runner" role on the project that runs the query (the billing project). Without it, every job-create call returns 403 even with full data-read access. The same pattern recurs across services: a pub/sub publisher needs the publisher role but also a viewer role on the topic for some SDKs; a secret consumer needs the secret-accessor role on the secret plus a service-account-user role when impersonating. Document BOTH grants in the PR body + repo runbook; don't let the client's docstring be the only complete source.
- **Permission audit cross-reference between code, docs, and PR body.** When a PR touches IAM, check 3 sources for consistency:
  - The client docstring (`lib/db/warehouse.ts:9` or equivalent)
  - The repo docs (a runbook section)
  - The PR-body post-merge manual section
  All three must list the SAME set of roles. An external reviewer routinely catches divergence: the code says "both job-runner + data-viewer", the docs say only "data-viewer". The operator follows the docs, and prod silently degrades.
- **Validate IAM grants exist as a post-merge smoke test.** Query the cloud's IAM policy for the project, filter the bindings to the service account's email, and list its attached roles. Compare with the required set listed in the docstring. A mismatch = a blocker before declaring the deploy complete.

## 17. Defensive deploy patterns
- **`relation does not exist` (PG `42P01`, warehouse "Not found: Table") as a silent fallback.** When app code depends on a table created by another process (a cron job, manual setup, a separate PR), the app must catch the "not found" error and degrade gracefully. Pattern:
  ```ts
  try {
    rows = await query(...);
  } catch (err) {
    if (isTableNotFoundError(err)) return { rows: [], freshness: { isMissing: true } };
    throw err;
  }
  ```
  This allows the PR to merge BEFORE the table exists. When the table appears, the app starts using it on the next request without a redeploy. Zero downtime + zero coupling between teams/repos.
- **Test-fragile when the local environment differs from CI.** A test that asserts `expect(SomeClientCtor).toHaveBeenCalledWith({ projectId, retryOptions })` may pass in CI (no credentials file) but fail locally (the file exists → the client gets a `keyFilename` arg). Fix: explicit `vi.mock('fs', ...)` or a fake `existsSync` in the test setup. Tests should not depend on host-machine state.

## 18. Migration and deprecation patterns
- **308 redirect for deprecated routes** with a `Deprecation: true` header and `Link: <new-url>; rel="successor-version"` — the standard HTTP migration. Preserves method + body for clients and signals to monitoring tools that the route is being sunset.
- **Hard caps without a parameter = silent truncation.** A route that returns `LIMIT 50` or `LIMIT 100` rows without a `?limit=` parameter caps client visibility. If the caller paginates assuming all data was returned in a single call, they miss rows. Same pattern: a response field with an array cap (`coAttendees: 30 max`). Either accept a `?limit=` parameter or include a `truncated: true` flag in the response.
- **A contract test for deprecated identifiers should validate existence, not just strings.** A test that grep-finds named views (`warehouse.enriched.v_*`) in code is good for catching reintroduction of those views, but doesn't catch when code references a DIFFERENT broken table (like `dim_company`) that was the root cause of the original problem. Augment with: a weekly job that lists every table reference in `app/api/**/*.ts` via regex and validates them against `INFORMATION_SCHEMA.TABLES`.

## 19. Edge case philosophy
- Does the code handle empty input, null, undefined, single-element collections, and maximum-size inputs — or only the happy-path demo case?
- Boundary values: 0, 1, and N-max are the three inputs that find most off-by-one bugs; check each explicitly.
- For numeric thresholds (`consecutiveYears >= 2`, `LIMIT 50`), ask: what happens at exactly the threshold? one below? one above?
- Document-store queries returning 0 docs vs. a query error: are these treated the same way? They shouldn't be.
- A warehouse query returning 0 rows on a `CREATE OR REPLACE TABLE` is an unhappy path the job marks as SUCCESS — see §2 `CREATE OR REPLACE TABLE` silent failure bullet.
- Unhappy-path coverage decision: which edge cases warrant a test vs. which are documented-as-out-of-scope? The reviewer must see that decision made explicitly — the absence of a test is not the same as a documented exclusion.
- Watch for: `array[0]` without a length guard, `Object.keys(obj)[0]` on a possibly-empty object, `.split(',')[1]` on a string that may not contain a comma.

## 20. DRY emergence
- When the THIRD copy of a pattern appears in the same PR, it's time to extract — not the second. Two call sites can be coincidence; three is a pattern.
- Distinguish true duplication (identical semantics, same invariants) from incidental similarity (same structure today, likely to diverge tomorrow). Don't extract a premature abstraction from two call sites that only look alike right now.
- Over-extraction is a smell too: a helper that adds a function call but no reusable logic or naming clarity is negative DRY.
- In Next.js API routes, repeated `try/catch → degradedSources push → return fallback` blocks are a DRY candidate; repeated `const session = await getServerSession()` lines are not (they must be co-located with the auth boundary).
- Shared SQL CTEs copy-pasted across two query files should live in a `buildXxxCTE()` builder, not be hand-synced — one will drift.
- Before extracting: verify all N copies have identical semantics. Run `grep -n 'patternSnippet' path/to/dir` to find all occurrences, not just the ones in the diff.

## 21. Style and idiom consistency
- Does the change match the idioms of the surrounding file (naming convention, error shape, import order, arrow-vs-function declarations)?
- A lone deviation is worth a comment: is it an intentional improvement or accidental drift?
- Check import ordering: the project uses a specific convention (external → internal → relative); a new import dropped in the middle silently breaks the pattern.
- Error shape: if the file returns `{ error: string }` objects, a new path that throws breaks the caller's `if (result.error)` contract.
- Naming: `getCrmContact` vs `fetchContact` vs `loadContact` — pick the verb the file already uses. Mixing verbs across sibling functions is a readability tax.
- Inline comments: if the file has no inline comments, don't add a wall of them; if it has JSDoc on every exported function, add JSDoc to the new one.
- `async/await` vs `.then()`: don't mix them in the same file without a reason.
- Trailing comma style, semicolon style — match the file, not personal preference.

## 22. Refactor opportunities (in-scope vs out)
- A refactor that rides along safely (rename a local variable, extract a small pure function) is acceptable in the same PR if the diff stays reviewable.
- A refactor that bloats the diff — large renames touching 15 files, restructuring an entire module — should be a separate PR or a follow-up issue. Mixed behavior-change + large rename makes `git blame` useless and review impossible.
- Flag the opportunity explicitly: "I see a chance to generalize `buildXxxFilter` across 3 routes — I'd suggest a follow-up issue rather than doing it in this PR."
- Never silently block a refactor from the PR; equally, never silently accept one that hides a behavior change. Surface the tradeoff.
- If the author mixed a refactor with a fix, ask them to split unless the refactor is trivially small — the fix needs to be atomic for bisection.
- Watch for: a "refactor" that actually changes runtime behavior (reordering `||` operands, changing a `>=` to `>`), which is a bug, not a refactor.

## 23. Diff hygiene
- Unrelated churn: reformatted lines that weren't functionally changed inflate the diff and hide real changes. Flag and ask to separate.
- Reverted formatting: `prettier` ran, then was reverted, leaving mixed indentation — a symptom of an editor/pre-commit hook mismatch.
- Stale debug artifacts: `console.log('DEBUG ...')`, `debugger`, `.only` / `fdescribe` / `fit` in test files, `JSON.stringify(result, null, 2)` in a route handler.
- Leftover TODO/FIXME: if the TODO was there before the PR, note it; if the author added a TODO to fix something they broke, it's a blocker.
- Accidental file-mode changes: `git diff --stat` showing `100644 → 100755` on a `.ts` file is always a mistake.
- Whitespace noise: `git diff --check` catches trailing whitespace and mixed tabs/spaces; require it to pass before approving.
- Committed `.env` or credential fragments — even partial API keys. Any `sk-`, `AIza`, `ya29.` substring in the diff is a hard stop.
- `.lock` file changes that don't match any `package.json` change: a sign of a manual `npm install` on a different Node version.

## 24. Pre-existing bugs surfaced by the diff
- The diff touched a file and exposed a latent bug that wasn't introduced by this PR — flag it anyway.
- Decision tree: (a) is the bug in code THIS PR changes? fix it here. (b) is it adjacent but not changed? open a follow-up issue and link it from the review comment. (c) is it critical/security? block the PR until it's fixed or explicitly accepted by the owner.
- Common surfacing pattern: a new test exercises a code path nobody tested before and finds the existing function returns the wrong output for an input that exists in prod.
- When a JOIN or query is restructured, check whether the previous version silently suppressed a bug that the new version now exposes to callers.
- Document pre-existing bugs clearly as "not introduced by this PR" — don't let the author think they're being blamed for old code.

## 25. Dependency versions and lockfiles
- New dependency: is it justified? Is there an existing package in the repo that covers the use case (`date-fns` vs `dayjs` both in `package.json` is a smell)?
- Version bump: intentional (`^18.2.0 → ^18.3.0`) or an accidental major bump (`^18 → ^19`) triggered by a lockfile regeneration?
- Lockfile-only change (no `package.json` change): acceptable only after `npm ci` on a fresh environment; otherwise it's uncontrolled drift.
- Transitive risk: a minor dep bump can pull in a new transitive dep with a different license. Check `npm ls <newpkg>` if in doubt.
- Supply-chain sanity: for any new package with < 100k weekly downloads or an unknown maintainer, review the source briefly before approving.
- License: if the stack ships as a commercial product, `GPL-3.0` or `AGPL` transitives are blockers; `MIT`/`Apache-2.0`/`ISC` are fine.
- No accidental `npm install --save-dev` vs `--save` mismatch (a runtime dep landing in `devDependencies`).

## 26. Pluggability and extension points
- Does the design make the LIKELY NEXT change easy, or does it hard-code an assumption that will require restructuring when the roadmap item ships?
- Concrete examples: `if (org === 'acme')` hard-coded will need removal when a second org onboards; a hard-coded `LIMIT 50` will need a parameter when pagination is requested.
- Over-engineered extensibility nobody asked for is also a smell: a plugin registry for a function called from exactly one place adds cognitive load with zero current value.
- Heuristic: if the next roadmap item (visible in the issue tracker or known from context) directly contradicts an assumption in this PR, flag it — "this will need to change when issue #N ships."
- Extension points should be at the RIGHT layer: adding a `tenantId` parameter to a query helper is correct; adding it to a React prop 3 layers deep is premature.
- Hard-coded enum lists (`Type IN ('Renewal', 'New Business')`) must be documented and ideally driven by a constant — otherwise every new picklist value in the source system requires a code change.

## 27. "Compiles but wrong"
- TypeScript compiles and tests pass, but the semantics are incorrect. This is the hardest class of bug to catch in review.
- Off-by-one: `>= 2` vs `> 2`, `< length` vs `<= length - 1`, `LIMIT 50` returning 50 but the caller expects "all".
- Inverted boolean: `if (!isActive)` doing the work that `if (isActive)` should do — passes all tests written by the author because the author inverted the test too.
- Wrong field with the same type: `account.billingCountry` vs `account.shippingCountry` — both `string`, both non-null, TypeScript cannot distinguish.
- Swapped arguments of the same type: `formatCurrency(amount, currency)` called as `formatCurrency(currency, amount)` — both `string`, compiles fine.
- Semantic negation: `degradedSources.length === 0` used to mean "all sources OK" when the intent was "at least one source degraded".
- Silent truncation: `LEFT(bio, 600)` returns a valid string even if bio was 601 chars — the type is still `string`, the value is silently shorter.
- Test for this class: ask "could I write a test that passes with this code but fails with the correct code?" If yes, the existing tests don't cover it.

## 28. Cognitive load
- Can a reviewer hold the entire function in their head while reading it? If not, the function is too complex regardless of what it does.
- Nesting depth > 3 (if → if → for → if) is a strong signal to extract or early-return.
- Number of things to track simultaneously: a function with 4 local variables, 2 flags, and 3 early-returns requires holding 9 pieces of state — a tired engineer debugging at 2am will miss one.
- A 3-level fallback chain (`warehouse cache → warehouse live → live DB`) is operationally necessary but cognitively expensive: it MUST have a single readable diagram or comment, not just emerge implicitly from nested try/catch.
- Long ternary chains (`a ? b : c ? d : e ? f : g`) are hard to follow and should be a named helper or a switch.
- Flag any function longer than ~60 lines that isn't a pure data transformer — it's almost always doing two things.
- Ask: "Would a junior engineer be able to debug this in 3 months with only the code and the tests?" If no, it needs a comment or a refactor.

## 29. Author decisions worth surfacing
- Choices the author made silently that the OWNER should ratify explicitly before the PR merges.
- Default values: `const DEFAULT_LIMIT = 50` — is 50 the right product default? Who decided?
- Excluded enum cases: `Status NOT IN ('Draft', 'Cancelled')` — are Draft and Cancelled intentionally excluded, or just forgotten?
- Chosen library: `import { parseISO } from 'date-fns'` in a file that previously used `dayjs` — was this intentional or a copy-paste?
- Hardcoded org/env assumption: `if (process.env.NODE_ENV === 'production')` guarding a data write — does this skip the write in staging and QA?
- API version pin: a call to an external CRM's API `v57.0` — was v58 or v59 considered? Is there a deprecation notice?
- Make these explicit in review comments as questions to the owner, not blockers: "The author chose X — confirm this is intentional before merging."

## 30. Knowledge gaps and reviewer intuition
- Name explicitly what you could NOT verify in the review: "No DB access to check DISTINCT values of the opportunity-type column", "Couldn't run locally — trusted the test output", "Unfamiliar with the CRM's sharing rules for the case-safe contact ID."
- Stating what you didn't check is MORE useful than implying you checked everything — the owner knows which gaps matter and can fill them.
- When something "feels off" but you can't prove it, say so explicitly: "The `COALESCE` here feels like it may hide a NULL that should be surfaced — couldn't confirm without running against prod data. Worth a spot-check."
- Suppressing intuition signals because you "can't prove it" is a mistake — a well-framed "this smells like X, am I wrong?" takes 10 seconds to write and can prevent a production incident.
- Document domain-specific gaps: CRM picklist semantics, internal API contracts, workflow-automation trigger conditions, warehouse slot/quota behavior — these are things only the domain owner can verify. Ask by name, don't assume.
