# PR Review — Developer/Owner Dimensions

_Reference checklist: items that require business context, tribal knowledge, or production access to validate. The technical reviewer flags these; the developer/owner resolves them._

---

## 1. Real production data
- Real NULL distribution vs the schema doc.
- Value diversity — e.g. how many records fall outside the default currency/locale.
- How many edge records exist (multi-year contracts, multi-entity accounts).
- How many rows actually have the key field populated.
- Patterns in a name/identifier field (one universal format, or variants?).
- Dirty free-form fields ('Brasil' vs 'Brazil' vs 'BR').
- Outliers: a record with 50+ child rows, an account with unicode, a merged/duplicated record.
- Soft-deleted records on a replica that still return in `SELECT`.
- Typical replica lag and its impact on the feature.
- Long history: rows from years ago under an older schema.
- Drift between the schema/mapping doc and the real schema.

## 2. UX, perception, and visual communication
- Do labels make sense to the end user / operator?
- Tooltip / superscript — does anyone actually click it?
- Truncated text that cuts mid-sentence?
- Palette choices — elegant, or jarring against the product's design system?
- WCAG contrast on colored text/background combinations.
- Screen reader (is there an `aria-label` on the interactive element?).
- Does OS dark mode change anything?
- Perceived latency (is 2s acceptable for this interaction?).
- Skeleton loaders in the right place?
- Progressive render vs blocking.

## 3. Real performance, cost, and scale
- p95 of the request with N backend calls vs the platform's request-timeout limit.
- Connection pool size under concurrent load.
- Cold start on this path (serverless).
- Cache hit rate of any cached dependency (and its TTL).
- External rate limits (third-party APIs, warehouse slots, DB connections).
- Cost per operation (DB, egress, LLM tokens, warehouse scans).
- Operations/min at peak.

## 4. Tribal knowledge and institutional history
- Why a particular field or approach was originally chosen.
- Non-obvious workarounds that look like bad code but are necessary.
- Unwritten conventions.
- Meeting decisions that were never recorded.
- The original author of a critical module — what they were protecting against.
- Historical bugs that the pattern silently prevents.

## 5. Cross-team coordination and downstream impact
- Branches in flight touching the same files.
- Who consumes this endpoint besides the obvious caller (mobile? a bot? a webhook?).
- Low-code automations (Zapier/Make-style) that read a field this PR changes.
- Does the data team depend on this view/endpoint?
- Does another team use the same external field?
- A reverse sync that writes to fields we now read.

## 6. Product decisions and prioritization
- Should the behavior react to additional signals?
- When to switch to a different source field or metric?
- Ship-now vs ship-right — what's the current appetite?
- Should the PR be sliced?

## 7. Trust signals and author context
- Tone of the description (organized vs rushed).
- Churn history on the file.
- Deadline pressure.
- LLM co-author — how much was reviewed vs accepted raw?

## 8. Organizational and political risk
- Does a design-system exception set a precedent?
- Does a rename break another team's report?
- Who has the context + time to review this by hand?
- Authority over external fields — does someone own them?

## 9. Security and the real threat model
- Does the CSP allow external images/content from the third-party domains in use?
- Free-form fields may contain HTML/script from web forms (stored XSS into anything that renders them).
- The supply chain of any AI-generated field (what produced it, can it be poisoned).
- External sharing/visibility rules: does an ID-based lookup respect them, or read across tenants?
- IAM on the read replica — does the user have `SELECT` only where intended?
- External media URLs — does the token expire? Referer leak?

## 10. Compliance, legal, ethics
- Privacy regulation (GDPR/LGPD/CCPA): personal data (photo, bio, phone) in a feature — what's the legal basis?
- A personal photo in an "AI summary" without the subject knowing — invasive?
- Retention: does the derived/mirrored data have a TTL? Right to be forgotten?
- Third-party AI ToS — commercial use without a disclaimer?
- Cross-border transfer (where do the source, the replica, and the compute live?).

## 11. Manual verification in production
- Smoke test on 3 representative records.
- Real frequency of 401/403 on external URLs.
- Real return order of a list function.
- p95 under real traffic.
- A dependency actually going down (not a mock).
- **"I validated" in an external comm requires the query to have been run in the same session.** Before asserting "validated", "verified", or "confirmed" in a PR comment, email, or message to another team: run the corresponding query/command. Fabricated confidence disguised as fact costs credibility when someone cross-references it later. (Observed pattern: writing "all 9 materialized views already have a unique index" without having run the index query — 8 of 9 did not.) If not validated, write "the documentation suggests that X" or "assuming Y".
- **Validate the schema before asserting a column name.** In any dimensional/SQL/contract output you write (spec, docs, message to a dev, PR body), confirm the column name exists on the referenced table. Cost: 30s of query, avoids hours of rework on a wrong doc.
- **Line numbers in docs rot on the next commit.** Referencing `file.ts:534` in documentation = a reference that will be wrong next week. Pattern: always `grep -n 'stable_function_or_const' path/file.ts`, which locates by name. Names change less than line numbers.

## 12. Environment state, deploy, and rollback
- Does the runtime version in prod match the manifest (`package.json` / lockfile)?
- Container rebuild with the new imports/deps?
- Is the build/version identifier bumped everywhere it appears?
- Do prod env vars match the defaults the code assumes?
- Feature flag to hide the new behavior?
- Rollback plan (previous version? tagged image? canary?).

## 13. Incidents, observability, alerts
- Does the PR resolve an open RCA?
- Will the error tracker get noisy with new errors?
- Do the dashboards still work with the new response shape?
- Did the log shape change — do log-sink consumers break?
- Should a new drop/degradation become a metric?

## 14. Concurrency and consistency
- Typical replica lag.
- Stale writes: two operators editing the same record.
- Eventual consistency (a server timestamp vs the immediate read).
- A cache invalidating mid-request.

## 15. Maintainability, capacity, and team culture
- Bus factor: who maintains this critical module?
- Can a junior debug a 3-level fallback chain in 3 months?
- Does the onboarding doc need an update?
- PR size — does the team accept it, or prefer to slice it?

## 16. Workflow and governance
- **An issue passes a critique gate BEFORE becoming a PR.** After the issue + spec exist, request a critical review of the issue before delegating to an implementer. The reviewer looks for 4–6 specific gaps: repo convention, well-sized scope, documented options, measurable acceptance criteria. Only after the issue is robust does the code begin.
- **A written plan is the explicit contract before code.** An issue = "what to do"; the plan doc = "how I'll do it", written before the PR appears. The reviewer evaluates the plan when asked, focusing on: repo conventions, adherence to scope, operational risks, and open points that need a decision from the owner (not from the implementer). The owner's explicit approval of the plan = the unlock for writing code.
- **Architecture as a blocking gate even with a finished PR.** A technically-approved PR can still be rejected if the architecture is in debt — real rework cost is preferable to debt committed to main. Signal: a reviewer ending with "approve without reservations" on a non-trivial architectural change should re-check the architectural criterion before approving definitively. Full detail in `dimensions-technical-deep.md` §1.
- **Reframe vs isolated fix.** When a bug can reappear in other instances of the same architectural pattern, propose a reframe (Option B — treats the whole class) instead of an isolated fix (Option A — treats the symptom). Offer three options in the issue body: A (isolated), B (whole class, recommended), C (accept tech debt while risk is low).
- **Signal before filter.** A sound default data policy: bring the maximum available signal first; filters are iteration 2 based on real usage, not a guess. Except when "noise" has a concrete cost (compute, money, latency, PII). On a PR that proposes filtering "obvious" data, validate with the owner before approving.

## 17. Roadmap and sequencing
- Does this PR pull forward or block any item planned in the roadmap for the next 2 sprints?
- Does the delivery order make sense: is there a dependency on another delivery not yet done (a table that doesn't exist yet, an unmapped field, an undeployed endpoint)?
- Does the PR assume a future system state that will only exist after another PR is merged? If so, it needs a feature flag or a coordinated atomic deploy.
- If the roadmap has an item that will CONTRADICT a choice made here (a hard-coded org, a fixed row limit), flag it now — refactoring later costs more.
- Ask the owner: "Which roadmap item does this PR go ahead of? Is any delivery blocked by it?"

## 18. Multi-tenant and multi-org
- Does the behavior change between orgs/tenants in any undocumented way?
- Does the code implicitly assume a single org (a hard-coded `WHERE org_id = '...'`, a storage path with no per-tenant segmentation)?
- Is data isolation between orgs preserved: a user of org A cannot see org B's data via a direct query or via a shared cache?
- Per-org feature flags and config: does the PR use a config accessor (`getOrgConfig()` or equivalent), or assume defaults valid for all orgs?
- When the second tenant onboards, what in this PR will need to change? List it explicitly — that is the PR's multi-tenant debt.

## 19. Brand voice and tone
- Do user-visible texts (labels, error messages, tooltips, automated emails) match the product's brand tone and vocabulary?
- Terminology consistent with the product: use the canonical product term, not a synonym ("member" vs "contact" vs "lead").
- Error messages to the operator should be actionable ("Data unavailable — try again in a few minutes"), not technical ("query timeout 503").
- Did the emails and notifications generated by the PR go through a copy review? Does the formal vs informal tone fit the channel?
- Hard-coded strings in one language for a product with multilingual users are a warning sign: confirm whether i18n is needed or the audience is always one locale.

## 20. Tolerance for tech debt
- Is the debt introduced explicit and acceptable for the current moment (a commented TODO with a linked issue, not a loose TODO with no context)?
- Is there real team appetite to pay this debt later, or is it the kind that piles up for months?
- Does the follow-up become a real issue with acceptance criteria, or is it just a verbal promise?
- Debt patterns often acceptable: a hard-coded limit/enum while the product is single-org; lack of pagination while volume is low. Patterns that are NOT: a "temporary" auth bypass, missing logging on a critical path.
- Flag when the PR takes on debt that affects observability or security — those are not "acceptable for now".

## 21. Side effects and external integrations
- Does the PR change what triggers webhooks, low-code automations, or a downstream writeback?
- Any side effect on an external system not documented in the PR body (a field overwritten as a side effect of a user action)?
- If the PR changes the shape of an event or webhook payload, were the downstream consumers (automations, bots, mobile) notified?
- External integrations: does the PR read from a field that another system also writes? Is there a race-condition risk between writeback and read?
- Automated emails or notifications triggered by this flow: could the PR accidentally increase the volume or change the trigger condition?

## 22. Business metrics and impact
- Does the change affect a KPI or counter someone tracks on a dashboard?
- Is the impact declared in the PR body ("reduces churn", "increases conversion") verifiable against real data within a defined timeframe?
- If the PR changes the calculation logic of an existing metric, historical values will diverge — is that expected and communicated, or a silent regression?
- Cost metrics: does the PR increase query volume, third-party calls, or LLM tokens? Estimate the order of magnitude before approving a change on a high-volume path.
- Ask the owner: "What's the measurable success criterion for this PR in 2 weeks?"

## 23. Operator mental model
- Does the change match how the operator thinks about the flow? Does it introduce a new concept that will require training or internal comms?
- If the UI changes a behavior the operator has already internalized (field order, a status icon), does that need a notice or changelog?
- Do errors or degraded states surfaced in the UI make sense to a non-technical operator, or are they messages for the dev?
- New flows: does the operator need more than 2 clicks to complete the main action? If so, review the UX before approving.
- Ask the owner: "Will an operator who wasn't trained on this change get confused?"

## 24. Cognitive load (operator)
- Does the operator need to remember more state or steps than before to complete a task?
- Is critical information hidden behind a tooltip, hover, or accordion? If the operator doesn't discover it, they may make the wrong decision.
- Does the PR increase the number of places the operator must act to complete a flow (update an external field AND confirm in the product)?
- New forms or inputs: sensible defaults reduce load; required fields with no default increase friction — justify each new field.
- If cognitive load increases, check whether there's a UX plan (skeleton, progress, confirmation) to compensate.

## 25. Knowledge gaps and reviewer intuition
- Explicitly record what ONLY the domain owner knows and the reviewer cannot validate without special access: the semantics of an undocumented field, a tactical business rule, the behavior of a legacy integration.
- If something "looks wrong" but there's no proof, say it as an explicit question: "This `COALESCE` seems to assume the key is always an Account — is that guaranteed by the business rule?" — don't suppress the signal.
- Common gaps the reviewer should declare when unverified: (a) real NULL distribution in a field, (b) replica-lag behavior at peak hours, (c) external sharing rules for the touched object, (d) custom picklist/enum semantics, (e) warehouse slot quotas on the prod project.
- "I couldn't validate this locally" is useful information — writing that is more honest than implying the flow was tested end-to-end.
