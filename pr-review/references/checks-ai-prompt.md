# AI Prompt Checks

Open this when the PR touches LLM prompts, AI calls, or any code that feeds data into a language model — invalid data feeding a prompt is a blocker, not an owner decision.

---

## Prompt Data Integrity — **P0/P1**

Every field that feeds into an LLM prompt must be either (a) correct, or (b) explicitly marked as unavailable in the prompt text itself.

- **Hardcoded defaults that enter the prompt will be narrativized as facts.** `target: 0` becomes "the member has zero coverage" in the narrative. This is a **P1 blocker**, not an "owner decision."
- If a field is "not yet implemented", the prompt must say `"target not available"` — not `"target: £0"`.
- Applies to all prompt inputs: snapshot data, user-supplied text, external CRM field values, any dynamic injection.
- **RBAC on secondary resources referenced in prompts**: when a route accepts an ID for a secondary resource (e.g. `comparedToSnapshotId`), verify the fetched doc's ownership is checked against the viewer before it enters the prompt. RBAC on the primary resource does NOT protect the secondary one — a viewer can read another owner's pipeline data through the LLM. → lessons-ledger.md

**The rule**: invalid data feeding an AI is a blocker. Not a flag. Not an owner call. A blocker.

---

## SDK Defaults That Affect Latency and Cost — **P1**

When the PR configures a timeout for an external SDK (the LLM SDK, an email-delivery SDK, etc.):

- Check the SDK's default `maxRetries`. `timeout × (maxRetries + 1)` must fit within the infra timeout (platform request timeout, edge/proxy timeout).
- Example: an LLM SDK with default `maxRetries = 2`. With `timeout = 45s`: **45 × 3 = 135s > a 100s proxy timeout** → gateway timeout, not a clean error.
- If total wall-clock does not fit, set `maxRetries: 0` explicitly.
- Also check: default backoff strategy, connection pooling, keepalive settings.
- → lessons-ledger.md

---

## Write Failure After Expensive AI Call — **P1**

When the flow is: `external AI call (tokens spent) → persist (Firestore/PG) → response`:

- If the persist step fails, tokens are already spent and the user receives a 500 with nothing to show.
- Verify at least one of:
  1. The persist has a retry.
  2. On persist failure, the draft/result is returned in the response anyway (degraded mode — user has the content even if it wasn't saved).
  3. The persist is fire-and-forget and the response is returned before persistence completes (acceptable only if losing the write is tolerable).
- Identity/cache write failures (e.g. updating a viewer-ID cache) must not propagate to crash the main request — cache writes are best-effort and must have their own `try/catch`. → lessons-ledger.md

---

## Template Placeholder Validation on Publish — **P1**

When the PR allows users (including superadmins) to edit templates with required placeholders (`{{member_name}}`, `{{company}}`, `${title}`, etc.):

- The **publish endpoint** must validate that all required placeholders are present before accepting the change.
- Without this: a superadmin publishes a prompt missing `${title}`, every downstream refresh breaks silently — the variable either becomes an empty string or renders as the literal `${title}`.
- The catalog/schema should declare required placeholders per template. Publish rejects content missing any required one.
- **Test coverage**: verify the test suite includes a case for publishing a template with a missing placeholder.
- → lessons-ledger.md
