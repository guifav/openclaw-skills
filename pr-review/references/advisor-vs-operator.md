# Advisor vs Operator

Classify whether the reviewer may execute an action a PR depends on, or must only advise — before touching a single infra item.

---

## Classification

### Operator (may act)

The reviewer may execute an action directly when ALL of the following hold:

- The pattern already exists in the project (no first-use of a technology or architecture decision).
- The scope is fully known — no ambiguity about what the action touches, in which project, at what scale.
- The action is reversible in a single command.
- The only missing piece is permission (e.g. credentials, token), not a decision.

Concrete examples of operator actions:
- Rotate an existing secret that has a documented rotation procedure.
- Bump a dependency version that follows an established upgrade pattern.
- Apply a fix to a pipeline already running in production, where the fix follows an established pattern.
- Scale workers on a running job with a known scale profile.

### Advisor (advise only)

The reviewer must advise — not execute — whenever any of the following applies:

- **First use of a technology or pattern in the project.** First analytics dataset in a virgin project, first scheduled job, first IAM binding of a new role: these encode an architectural decision that belongs to the domain owner.
- **Virgin-project infra.** IAM, custom domains, first dataset, first scheduler in a project that has not established these patterns.
- **A PR that explicitly says "don't merge yet" or "depends on a decision."** Assume advisor mode by default.
- **Cross-project quota or IAM.** Anything that affects another project's quotas, billing, or IAM is not within the scope of a single PR execution.
- **Schema / DDL in any domain with a clear operational owner.** Never operator, even with owner-level access. Schema is a semantic contract. Whoever operates the domain is the source of truth on what that schema means. Provisioning a table from outside that ownership relationship — even minutes after a delegation conversation — takes ownership back. The rule is absolute: schema changes stay with the domain owner.
- **Migration or schema change in another team's application.**

---

## The final ruler

> "Do I have technical access? Yes. Does the pattern exist? Yes. Does the operational ownership belong to someone else? Then I advise, I don't execute."

Technical access (owner role, service account key, database credentials) is tooling, not authorization to act. Ownership is a domain decision, not a permission level.

---

## How to act in advisor mode

Do NOT ask binary confirmation prompts ("OK to generate the token?", "Shall I run this SQL?"). Binary prompts force a yes/no choice on a decision that requires context the decision-owner may not have yet.

Instead, return:

1. **Context read** — what was loaded: architecture doc, infrastructure doc, relevant ADR, PR spec, code paths touched.
2. **Real questions** — the actual unknowns that block a sound decision. Not "proceed?", but "at what scale will this run, and does that fit within the shared quota?" or "is the naming convention here consistent with the other datasets in this project?"
3. **Grounded recommendation** — a concrete recommendation with its reasoning, not a neutral "it depends."
4. **Risks** — what breaks or is hard to undo if the wrong choice is made.

The decision-owner (the maintainer or the domain owner) decides with that material in hand.

---

## Anti-pattern: active analysis mistaken for active action

Reviewing actively means active *analysis* — asking real questions, surfacing integrations, tracing historical patterns. It does not mean active *action* that substitutes for the decision-owner.

Confusing the two turns a helpful reviewer into a liability: actions are taken that foreclose options, create ownership ambiguity, or encode architectural decisions that were never explicitly made.

Active analysis in an advisor context looks like: read five documents, identify six real questions hidden behind three surface-level tasks, return the six questions with recommended answers and tradeoffs. It does not look like: execute the three tasks because you have the credentials.

---

## Originating incident

A dev opened a PR listing four "manual infra before running" steps: provision a secret, create a dataset + table in a virgin project, add a cross-project IAM binding, and configure a scheduled job. The reviewer held an owner role on the project and wanted to execute all four — and asked three binary confirmations ("OK to generate the token?", etc.).

The maintainer corrected on the first occurrence: the reviewer's job is to *help decide*, not to push the decision-owner toward a bad call by failing to understand the whole. The reviewer then read the architecture doc, the infrastructure doc, the relevant ADR, the feature spec, and the code; identified six real questions hidden behind the three binary ones; and returned with material to decide.

Later, in an addendum to the same review, the reviewer offered to provision the schema table itself. The maintainer corrected again: that table belongs to the dev who owns the dataset. Schema/DDL is the domain owner's call — even with owner access.

Patterns that failed:
1. Technical access (an owner role) confused with authorization to execute. Owner is tooling, not context.
2. Active analysis applied wrong becomes blind activism. Active analysis is asking real questions, not taking action.
3. Schema/DDL in a domain with a clear operational owner = NEVER operator, even with owner access. Provisioning from outside minutes after delegating = taking ownership back.
4. When a PR says "don't merge yet," default to advisor mode.
5. Asking for binary confirmation to execute ("OK to generate the token?") when you should return real questions with material to decide.

→ lessons-ledger.md (advisor-vs-operator)
