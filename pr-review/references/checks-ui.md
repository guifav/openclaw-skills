# UI Checks

Open this when the PR touches UI or client-side code — client error swallowing, perceived latency, and error state visibility are the primary failure modes.

---

## UI Technical Correctness

- Do components render with missing data (null props, empty arrays, undefined fields)?
- Does text avoid breaking layout under realistic data lengths?
- Do links, images, and fallbacks work? Do external images have `onError` handlers?
- Are loading, error, and empty states covered — all three, not just the happy path?
- Do labels reflect actual data, not hardcoded or aspirational values?
- Is a tooltip NOT the only way to understand something critical? Tooltips are invisible on mobile/touch.
- No lost actions on mobile/touch/keyboard navigation?
- Are all Tailwind classes valid (no typos, no unknown utilities)?
- No race conditions in local state (e.g., stale closures, state set after unmount)?

---

## UI Error Visibility — **P0/P1**

Every `catch` in client code must update a user-visible state (error banner, toast, inline message). This is not a nit — it is a blocker.

- **`catch { console.error }` without UI feedback = P1 blocker.** The user clicks, nothing happens, they have no idea why. → lessons-ledger.md
- **`Promise.allSettled` that maps rejected → empty array** must show a degraded indicator, not a "healthy empty state." During a database outage, the pipeline looks like it has zero data — completely false confidence. → lessons-ledger.md
- Distinguish clearly in the UI: "no data" (legitimate, expected) vs "failed to load" (error, actionable).

**Checklist for every `catch` block in the diff:**

1. Is there a `try/catch` at all? (An uncaught promise rejection is worse.)
2. Does the catch update a state variable that renders a user-visible message?
3. If using `Promise.allSettled`, does the result-mapping distinguish settled-with-rejection from settled-with-empty?
4. If using degraded data (falling back to `[]` or `null`), is there a visible degraded indicator?
