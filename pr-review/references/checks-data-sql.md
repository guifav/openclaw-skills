# Data & SQL Checks

Open this when the PR touches SQL / queries / schema reads.

---

## Column / CRM-Field Existence

- **Validate column names against the live schema** for every table the PR queries. `information_schema.columns` takes 10 seconds — run it. **P0** if skipped: the column may not exist.
- PG is case-sensitive with double quotes. `"EndDate"` != `"End_Date"` — the wrong name silently returns NULL or breaks the query entirely.
  - `grep -n 'EndDate\|End_Date' <file>` then cross-check against live schema.
- External CRM field names must match the exact CRM API name. Column does not exist in PG = data silently absent.
- **Field fetched but not consumed = dead code or broken contract.** If a new column is SELECTed but the filter still uses the old approach, the canonical field is wasted. **P1**.
  - Example trap: `IsClosed` SELECTed but filter still does `stage !== 'Closed Won'`. → lessons-ledger.md

---

## Query Semantics

### JOIN Cardinality & GROUP BY

- Do JOINs avoid row multiplication? A one-to-many JOIN without aggregation inflates counts silently.
- **CTEs with aggregate functions (`COUNT`, `SUM`, `MODE`) without `GROUP BY` fail at runtime in PostgreSQL.** This is a **P0 runtime error blocker**.
- When a CTE aggregates a table (e.g., `GROUP BY account_id`) and the outer query filters by a parameter (e.g., `WHERE a."Id" = $1`), verify the CTE also includes the filter — otherwise the CTE computes aggregates for ALL rows and discards most of them at the outer level. **P1** performance blocker on large tables.
  - Cross-check: if `getAccountsByOwner` pushes the filter into CTEs, does `getAccountById` do the same? → lessons-ledger.md

### Filters, LIMIT, and ORDER BY

- Are `LIMIT`, `ORDER BY`, and filters adequate?
- Do filters use strong semantic fields (`IsClosed`) instead of fragile labels (`stage !== 'Closed Won'`)?
- Is the query parameterized and safe? No dangerous SQL interpolation.

### Expiry / Date Filters

- Check that aggregations involving time-bounded data include date guards. Example: `renewal_agg` without `end_date > today` counts expired subscriptions as overdue in health scores. **P1**. → lessons-ledger.md

---

## Currency & Units Mixing

- **Currency fallback in financial aggregates is always a blocker. P0.**
  - `amount_gbp ?? amount` silently mixes GBP and raw currency when `amount_gbp` is NULL. A single non-GBP opportunity poisons the aggregate with no error. → lessons-ledger.md
- Do aggregations avoid mixing currencies, units, or scales?
- Are monetary values formatted per their source?
- Confirm with the dev/owner whether a currency assumption (e.g., "GBP company-wide") is actually valid in production data.

---

## NULL vs 0 vs Absent vs False

- Are `undefined`, `null`, `0`, `false`, and empty string differentiated correctly?
- Does "partial data" look different from "zero data"?
- Does a failed query fallback mask the error as real data? A `.catch(() => default)` that returns zero-data on failure hides errors as legitimate zeros. **P1**.
- What does NULL mean for each field: doesn't exist, not synced, no permission, or error? This must be explicit — not assumed.

---

## Identifier Regex (Domain-Specific)

- **Never accept a generic identifier regex when the domain has a known format. P1.**
- Example: a CRM's Contact IDs always carry a fixed prefix (e.g., `003`) and a fixed length (exactly 15 or 18 chars).
  - `^[a-zA-Z0-9]{15,18}$` is too loose — it accepts 16/17 chars (which don't exist in the CRM) and any prefix (a URL fragment could pass).
  - Correct regex: `^003[a-zA-Z0-9]{12}([a-zA-Z0-9]{3})?$`
- Test the regex against **realistic negative examples from the actual bug**, not just obvious junk. Could a fragment of the bad data (e.g., part of an HTML URL) pass?
- When the fix validates at one call site, `grep -rn` for all other consumers of the same identifier field — the same loose regex may exist elsewhere. → lessons-ledger.md

---

## Drop / Filter Observability

When a filter silently discards rows (invalid identifier, malformed data):

- Is there a log/warn when rows are dropped, with a count (not raw payload, for PII)?
- Is the log structured — not swallowed by `.catch(() => [])`?
- Is there a way to detect when drop rate exceeds a threshold? 100% drop = empty UI with no alarm. **P0**.
- Adding a filter without logging is a **P0 blocker** — silent data loss is worse than noisy bad data. → lessons-ledger.md

---

## Live Validation (Mandatory When PR Touches SQL/Data)

- Run `information_schema.columns` against the real database for every table the PR queries.
- Smoke test representative records with the actual query.
- Run the actual query for a known record and confirm the result makes sense.
- **"Production-proven" doesn't mean correct.** Case sensitivity, refactored context, or renamed columns can break a query that previously worked in a different code path. **P1**.
- **When the PR says "assumed/likely/follow-up to verify" — verify NOW.** Do not defer. Run the query in the same session.

---

## Exhaustive Consumer Search

- **When a PR sanitizes, filters, or validates a field: `grep -rn` the field name across the ENTIRE repo before approving. P1.**
- List every file that reads/writes/joins on the field. Verify EACH consumer is covered by the fix or explicitly deferred with a follow-up issue.
- **A fix that covers 5 of 8 consumers is WORSE than no fix** — it creates false confidence that the problem is solved.
- The reviewer who approves "with coverage" owns the consumers they missed. → lessons-ledger.md
