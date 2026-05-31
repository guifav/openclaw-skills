# Concurrency & State Catalog

Open this when the PR touches Firestore writes, state-machine transitions, approval/claim/assign flows, or any cache that serves shared or per-user data.

---

## Atomic Multi-Document Writes

### Rule (P1)
When a route writes to more than one Firestore document — overlay + audit entry, request + audit log, any pair — ALL writes MUST be inside a single `WriteBatch` or `runTransaction`.

Sequential `doc.set()` / `doc.update()` without a batch means the first document commits and the second may fail. The result: a partial write that leaves the audit trail incomplete. Callers that retry see stale state or "no changes."

**Partial writes that leave audit trails incomplete are blockers, not recommendations.**

### How to check
```bash
# Find every Firestore write in the route file
grep -n 'set\|update\|create\|delete' app/api/<route>/route.ts

# Confirm a batch or transaction wraps them
grep -n 'WriteBatch\|writeBatch\|runTransaction\|batch\.commit' app/api/<route>/route.ts
```

- If `writeBatch()` / `batch.set()` / `batch.commit()` are absent, that is a **P1 blocker**.
- Check precedent: if another route or the pipeline uses `WriteBatch` for the same pattern, the new code must too. Divergence from precedent without documented justification is a blocker.

### Anchor
```bash
grep -n 'writeBatch\|WriteBatch\|runTransaction' lib/firestore*.ts app/api/**/*.ts
```

→ lessons-ledger.md

---

## State-Machine Transitions — Read-Guard-Write Race

### Rule (P1)
When a route reads a status field, checks a guard (`status !== 'pending'`), then writes a new status, the entire read-guard-write sequence MUST be wrapped in `db.runTransaction()`.

Without a transaction, two concurrent requests (two managers approving simultaneously) both pass the guard and clobber each other's writes.

**Common patterns that require transactions:** approval flows, status transitions, claim/assign patterns.

### The anti-pattern
```ts
// WRONG — race condition
const snap = await ref.get();
if (snap.data()?.status !== 'pending') throw new Error('...');
await ref.update({ status: 'approved', decidedByUid: uid });
```

### The correct pattern
```ts
await db.runTransaction(async (tx) => {
  const snap = await tx.get(ref);
  if (snap.data()?.status !== 'pending') throw new Error('...');
  tx.update(ref, { status: 'approved', decidedByUid: uid });
});
```

### How to check
```bash
# Find guard checks on status fields
grep -n "status !==\|status ==\|status ==" app/api/<route>/route.ts

# Verify runTransaction wraps them
grep -n 'runTransaction' app/api/<route>/route.ts
```

If `ref.get()` → guard → `ref.update()` exists without `db.runTransaction()` wrapping it, that is a **P1 blocker**.

→ lessons-ledger.md

---

## Cache & Shared State

### Per-user data must never live in shared cache (P1)
A shared cache (Firestore collection, Redis) must NOT contain viewer-specific content: private notes, user-scoped filters, access-controlled fields. The cache writes team-level data; the route filters per viewer at read time.

```bash
grep -n 'private\|notes\|viewer\|userId' lib/cache*.ts lib/firestore-cache*.ts
```

### Write-path invalidation vs TTL — different semantics (P2)
- **Write-path invalidation** (`invalidate()` called after a POST/PATCH/DELETE) must force a **synchronous rebuild** on the next read — stale-while-revalidate is incorrect here.
- **TTL expiry** can use stale-while-revalidate because the trigger is time, not a known data change.

Verify that `invalidate()` covers ALL write paths listed in the design doc — not just POST but also PATCH, DELETE, audio upload, and any other mutation.

```bash
grep -rn 'invalidate\(\)' app/api/ lib/
# Then check every mutation route is represented
```

### Cache metadata semantic changes are breaking changes (P1)
If the PR changes what `cached: true` or `generatedAt` means (e.g., the field previously meant "cache hit in one service" and now means "cache hit in a different service"), downstream consumers that read those fields will misinterpret them — even if the TypeScript type is unchanged. Flag as a **breaking change** requiring a version bump or field rename.

```bash
grep -rn '"cached"\|generatedAt' app/ lib/ components/
```

→ lessons-ledger.md
