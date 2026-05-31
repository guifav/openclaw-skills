# Security & RBAC Catalog

Open this when the PR touches authentication, authorization, access control, config toggles, external IDs, or any write route that creates/updates/deletes data.

---

## RBAC on All Write Routes

### Rule (P1)
Every POST / PATCH / PUT / DELETE route must call `requireAccess` (or the project's equivalent) with the correct action before touching data. Missing RBAC on a write route means any authenticated user can write, bypassing policy entirely.

**A route that validates input and rate-limits but skips RBAC is worse than no route — it looks secured but isn't.**

### How to check
```bash
# List every write-method route handler in the PR diff
grep -rn 'method.*POST\|method.*PATCH\|method.*PUT\|method.*DELETE' app/api/

# Confirm requireAccess appears in each
grep -n 'requireAccess' app/api/<route>/route.ts
```

Checklist: for each POST/PATCH/PUT/DELETE route introduced or modified by the PR, confirm `requireAccess` (or equivalent) is called and the action matches the new resource name.

→ lessons-ledger.md

---

## RBAC on Referenced (Secondary) Resources

### Rule (P1 / P0 if data enters an AI prompt)
When a route accepts an ID for a **secondary** resource (`comparedToSnapshotId`, `parentJourneyId`, `linkedOutputId`), the **fetched document's ownership** must be checked against the viewer — not just the primary resource.

RBAC on the primary resource (e.g., the forecast draft the user owns) does **not** protect a secondary resource (e.g., a comparison snapshot owned by another user). A viewer can supply any valid document ID and read another owner's data indirectly.

This is **P0** when the referenced data is fed into an AI prompt, because the LLM narrativizes the stolen data.

### How to check
```bash
# Find every place a body/param ID is used to fetch a Firestore doc
grep -n 'doc(body\.\|doc(params\.' app/api/<route>/route.ts

# Confirm an ownership check follows immediately after .get()
grep -n 'ownerId\|ownerUid\|owner.*uid\|uid.*owner' app/api/<route>/route.ts
```

Pattern to look for:
```ts
// WRONG — no ownership check on secondary resource
const snap = await db.collection('snapshots').doc(body.comparedToSnapshotId).get();
// snap.data() used directly without checking snap.data().ownerUid === viewerUid
```

→ lessons-ledger.md

---

## RBAC Consistency: List vs Per-Resource Endpoints

### Rule (P1)
When a list endpoint (selector, feed) uses one access helper (e.g., `ownerAllowed`) and per-resource endpoints (detail, stats, candidates) use another (`requireAccess`), both must agree on every edge case.

**If the list shows an item, the per-resource endpoint must be able to open it.**

Critical edge cases:
- Viewer whose external user ID cannot be resolved → list shows items but per-resource returns 404.
- Resource with no owner (orphan document) → `ownerAllowed([], orphanDoc)` returning `true` with an empty allow-list is almost certainly a bug. An empty list should be deny-all, not orphan pass-through.

```bash
grep -n 'ownerAllowed\|requireAccess\|cochairOwnerAllowed' app/api/<route>/route.ts
grep -n 'allowList.*\[\]\|allow.*\[\]' lib/rbac*.ts lib/access*.ts
```

→ lessons-ledger.md

---

## Security — Auth, Inputs, and Injection

### Authentication & authorization (P1)
- Confirm existing auth middleware is still applied after the diff — no accidental removal or bypass.
- Existing authorization must not have been weakened (scope narrowed, guard removed).
- New data paths must go through the same auth checks as existing ones.

### Input and external-ID validation (P1)
- All user-supplied IDs must be validated before use. Example: for an external CRM's Contact IDs that always start with a fixed prefix and have a fixed length (e.g., a `003` prefix, exactly 15 or 18 chars):
  - Correct regex: `^003[a-zA-Z0-9]{12}([a-zA-Z0-9]{3})?$`
  - Generic `^[a-zA-Z0-9]{15,18}$` is too loose — accepts 16/17-char IDs that don't exist in the source system, and any prefix.
- Test the validation against **realistic negative examples** from the actual bug (e.g., an HTML URL fragment), not just obvious junk.

### Domain comparison (P1)
**Never use `endsWith` for domain checks.** `evil.example.com` passes `endsWith('example.com')`. Use exact string comparison.

```bash
grep -n 'endsWith' lib/ app/api/
```

### Prompt injection (P1)
When external content (user text, external CRM field values, event notes) enters an LLM prompt, verify it is escaped or clearly delimited. Inconsistency within the same file (one field escaped, another raw) is a P1.

```bash
grep -n 'prompt\|systemPrompt\|userMessage' app/api/<route>/route.ts
```

### XSS and external links (P2)
- No `dangerouslySetInnerHTML` with unescaped external data.
- All external links must have `rel="noopener noreferrer"`.

```bash
grep -n 'dangerouslySetInnerHTML\|rel=' components/
```

### Entropy and secret exposure (P1)
- Secrets and env vars must not appear in logs, responses, or tooltips.
- Count guards: how many independent checks must an attacker bypass? Fewer than two is a red flag.
- Secrets must have entropy floors (no placeholder strings accepted as valid).

---

## Config & Security Toggles — Fail Loud

### Rule (P0 / P1)
Env vars that control security behavior (RBAC mode, auth bypass, feature flags) **must fail loudly on invalid values**. A silent fallback to `disabled` on a typo is a **vulnerability**.

The parse function must emit a structured warning that includes:
1. The variable name
2. The received value
3. The fallback that was applied

```ts
// CORRECT
if (!VALID_MODES.includes(raw)) {
  console.warn(JSON.stringify({
    severity: 'WARNING',
    component: 'env',
    event: 'rbac_mode_invalid',
    varName: 'RBAC_MODE',
    received: raw,
    fallback: 'disabled',
  }));
}
```

```bash
# Find every env var that controls security
grep -rn 'RBAC_MODE\|AUTH_MODE\|FEATURE_FLAG' lib/env.ts app/
# Confirm each has explicit invalid-value handling
```

Per-feature override variable names must match the documented naming convention exactly (singular vs plural, case, prefix). `RBAC_MODE_CONTACT` vs `RBAC_MODE_CONTACTS` silently disables the granular rollout.

The parse function must live where the design doc specifies (e.g., `lib/env.ts`), not inline in a domain module.

→ lessons-ledger.md
