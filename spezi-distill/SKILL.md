---
name: spezi-distill
description: Extract a Gherkin specification from an existing codebase. Use when the user has existing code and wants to distil behaviour into a `.feature.md`, reverse engineer a specification from implementation, generate a spec from code, turn implementation into a behavioural specification, or document what a codebase does in Gherkin terms. The output is the same shape as `/spezi-elicit`'s but the source is code, not conversation.
---

# spezi-distill

Read the codebase, surface domain behaviour, and emit a `.feature.md` plus a decision record. The hard part is the same as elicit: choosing the right level of abstraction. Code is **over-specified** — it tells you *how*; the spec must capture *what* and *why*.

## Output

- Feature file: `specs/gherkin/<slug>.feature.md` — match `spezi/reference/feature-file.template.md`.
- Decision record: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>-distill[-<n>].md` — base + Distill section per `spezi/reference/decision-record.template.md`.

If the prospective slug collides with an existing `.feature.md`, ask: **rename** the new feature, **replace** the old one (requires `yes, replace`), or exit and run `/spezi-tend` instead.

## Boundaries

- This skill writes a new `.feature.md` from code. For targeted edits to an existing one, use `/spezi-tend`. For divergence-checking against code, use `/spezi-weed`.
- Never invents requirements that aren't grounded in code or stakeholder confirmation.
- Never modifies the codebase.

## Step 1 — Scope the distillation

Before reading any code, ask 3-5 labelled questions. The answers fix what gets specified.

```
a) What subset of the codebase should this spec cover? (a service, a domain, a feature)
b) Anything to deliberately exclude? (legacy, infra, deprecated, experimental)
c) Who owns this spec? (single team, single domain)
d) Is there a working `.feature.md` for this slug already? (yes → /spezi-tend; no → continue)
e) Is Allium available for downstream test generation? (informs propagate handoff later)
```

Apply the **"Would we rebuild this?"** test for any code path encountered: yes → include; no/legacy/infra/workaround → exclude.

## Step 2 — Map the territory

Read-only walk:

- **Entry points.** API routes, CLI commands, message handlers, scheduled jobs.
- **Domain models.** `models/`, `entities/`, `domain/`, schema files.
- **Business logic.** Services, use cases, handlers.
- **External integrations.** Webhook receivers, third-party clients.

Open with a one-line synopsis and confirm before extracting:

```
Found: <N> entry points, <M> domain models, <K> services, <J> integrations.
Looks like: <brief shape>. Proceed to extraction? (yes / refine / abort)
```

## Step 3 — Extract entities, states, transitions

For each domain model, read enum/status fields and the code paths that change them. Translate to Gherkin scenarios:

| Code pattern | Gherkin pattern |
|---|---|
| `if x.status != 'pending': raise` | `Given x.status is pending` |
| `if x.expires_at < now: raise` | `Given x.expires_at is in the future` |
| `x.status = 'accepted'` | `Then x.status becomes accepted` |
| `Model.create(...)` | `Then a <Model> is created with ...` |
| `send_email(...)` | `Then an email is sent to ...` |
| Scheduled job with time guard | `## Scenario: <Time-based — name>` with `Given <time elapsed>` |

Each extracted transition becomes one `## Scenario: <name>` block. Group scenarios by entity. For temporal triggers, label the scenario `## Scenario: <Time-based — name>`. For external boundaries (data the system reads but never writes), declare an `external entity` line in the prose summary.

## Step 4 — Filter to domain level

Apply three tests to every detail:

- **Why?** "Why does the stakeholder care?" If you can't answer, it's implementation.
- **Could it be different?** "Could this be implemented differently while still being the same system?" Yes → drop. No → keep.
- **Template vs instance.** Generic category (auth provider) vs specific instance (Google OAuth)? Default to template; promote to instance only when the user-facing flow names it.

Drop these silently: DB column types, ORM/query syntax, HTTP status codes, framework concepts, programming-language types, infrastructure (Redis, Kafka, S3), tokens/secrets, raw timedeltas. Replace foreign keys with relationship names. Convert `timedelta(days=7)` to `7 days`.

## Step 5 — Library-spec candidates

Watch for generic integration patterns. If you find OAuth handlers, payment processors, email delivery, calendar sync, ATS sync, file storage, or webhook patterns implementing third-party contracts, surface them:

```
Library spec candidates detected:
1. [oauth] Google OAuth flow → specs/gherkin/lib/oauth-google.feature.md?
2. [email] SendGrid integration → specs/gherkin/lib/email-sendgrid.feature.md?

Reply `1: extract`, `1: keep inline`, `1: defer`. Silence = keep inline.
```

`extract` → write a separate `.feature.md` under `specs/gherkin/lib/`, link it from the main spec via `Depends on`. Don't write library specs silently.

## Step 6 — Confirm with the user

Show the proposed full `.feature.md` plus a *What I dropped* summary (categories: db types, framework, secrets, etc.) and a *What I flagged* summary (ambiguous code paths, missing error handling that may be a bug, dead code, scattered logic that consolidated into one rule).

Flag list pattern (same shape as elicit):

```
Flagged for your attention (Distill — <slug>):
1. [scenario: <name>] <observation>
2. [implicit state machine] <observation>

Reply `1: accept`, `1: reject`, `1: defer`, or `1: <amended>`.
Silence = accept all  ← note: distill default is accept, since the source is code, not the user.
```

User can also redirect: `That scenario is wrong — actually X happens` → redraft.

## Step 7 — Persist

On confirmation:
- Write `.feature.md` with `status: complete`.
- Write the decision record. Sections: base + `Cycles`, `## Accepted suggestions`, `## Rejected suggestions`, `## Deferred / open questions`. Append a `## Scope decisions` section recording what was excluded (legacy, infra, deprecated) per Step 1.

## Common code-reading challenges

- **Implicit state machines.** Multiple nullable timestamp / FK fields encoding a hidden state machine. Make states explicit, name them, propose to the user.
- **Scattered logic.** Same conceptual rule split across handler + model + service. Consolidate into one scenario with all preconditions and effects.
- **Dead code.** Reachability is unclear. Flag rather than include.
- **Silent failure / missing error handling.** Capture *intended* behaviour, not the bug. Flag the bug separately (don't fix it; not this skill's job).
- **Over-engineered abstractions.** Strategy patterns, DI, factories. Cut through to the actual behaviour. The spec doesn't need the layers.

## After Confirm — Propagate hook

If `status: complete` and at least one `## Scenario:`, ask:

```
Distill complete. Generate tests for this spec via /spezi-propagate?

Reply `yes` / `not now` / `never this session`.
```

On `yes`, hand off with `{ seed: <slug> }`. Log outcome in *Session trail*.

## What this skill does not do

- No new behaviour elicitation. If the code is silent on something the user wanted, flag it once and suggest `/spezi-elicit` to add it.
- No code modifications. If the code has a bug, flag it; do not fix it.
- No `.allium` file edits. If an Allium link should be added (e.g. user names an existing Allium artifact during scoping), write the entry with `status: pending`.
