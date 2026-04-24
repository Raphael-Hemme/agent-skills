---
name: gherkin
description: Behavior-spec authoring and refinement for Behavioral Spec Driven Development. Produces Gherkin-style `.feature.md` specs alongside a decision record. Invoked by Spezi's router; not typically called directly.
---

# Gherkin Sub-skill

## Purpose

Author and maintain behavior specs in a markdown-embedded Gherkin style.
The sub-skill is a workshop, not a generator: it refuses to invent
behavior on the user's behalf and confirms every non-trivial decision
before persisting it.

## Modes

| Mode | Status | Summary |
|------|--------|---------|
| `elicit` | **Specified below** | Interactive session to produce a new `.feature.md` from a seed idea. |
| `distill` | Deferred | Tighten or restructure an existing `.feature.md`. |
| `read` | Deferred | Read-only review of existing specs. |
| `green` / `red` / `refactor` | Deferred | TDD-cycle operations over specs and implementation. |

A conforming implementation of this file must not silently run deferred
modes — if the router passes one, respond with "not yet implemented."

## Shared concerns

### Output locations

Relative to the project root (the directory where Spezi was invoked):

- Feature file: `specs/gherkin/<slug>.feature.md`
- Decision record: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>.md`

Create the `specs/gherkin/` and `specs/gherkin/decisions/` directories
lazily on first write. Never overwrite an existing feature file without
explicit confirmation from the user — if the derived `<slug>` collides
with an existing file, treat it as an ambiguity and ask.

### Slug derivation

The slug is derived from the confirmed *Scope* one-liner:

1. Lowercase ASCII.
2. Replace runs of non-alphanumeric characters with a single `-`.
3. Strip leading/trailing `-`.
4. Truncate to 48 characters at a word boundary if longer.

Examples: `"SSO login for enterprise users"` → `sso-login-for-enterprise-users`;
`"Empty-cart checkout edge case"` → `empty-cart-checkout-edge-case`.

The slug is only fixed after Scope is confirmed. Before that, the
session has no persistent identity.

### Escape hatches

The user may type one of the following at any prompt. The sub-skill
must recognize these as control tokens, not content.

| Token | Meaning |
|-------|---------|
| `/done` | The user considers the current phase sufficient. Advance to the next phase using whatever answers exist; missing answers become open questions in the decision record. |
| `/skip` | Skip *this specific prompt*. If given at a question batch, the sub-skill asks which question(s) to skip (by letter/number); a bare `/skip` at the start of a phase skips the whole phase and logs a gap. |
| `/back` | Return to the previous phase. The current phase's in-progress answers are discarded; already-confirmed sections remain. The sub-skill re-opens the prior section for revision. `/back` at Scope has no prior phase — the sub-skill offers `/abort` as the alternative. |
| `/abort` | End the session now. Persist any confirmed sections as a draft `.feature.md` with a prominent `Status: draft (aborted)` marker, and write a decision record with `Outcome: aborted`. Do not delete partial work. |

These tokens are reserved. A user who literally needs the string
`/done` in their answer can quote it (`"/done"`). Treat any unquoted
leading `/` token not in the table above as a typo — ask rather than
guess.

### Batch Q&A pattern

Every phase begins with a **single batch** of questions. Rules:

- 3–7 questions per batch. Fewer if the phase is nearly trivial; more
  only if the user has explicitly asked to be thorough.
- Questions are labeled (`a)`, `b)`, ...). The label becomes the user's
  reference when using `/skip a` or when answering out of order.
- The batch includes a closing instruction line:
  > Answer whichever you can in any order. `/done` if we have enough;
  > `/skip <letter>` to skip one; `/skip` alone to skip the phase.
- After the user answers, the sub-skill may ask **one** follow-up
  batch if critical gaps remain. Do not chain more than two batches in
  a single phase — at that point, flag the remaining gaps as
  ambiguities and move on.

### Ambiguity flagging

During a phase the sub-skill accumulates an internal **ambiguity list**
for items that are unclear, contradictory, or under-specified. At the
end of the phase — before proposing the section rewrite — the list is
presented as a single grouped block:

```
Flagged for your attention ([phase]):
1. [topic] <question as a short sentence>
2. [topic] <question>
3. [topic] <question>

Reply with `1: <answer>`, `1: defer`, or `1: out of scope` for each.
Multiple on one line OK. Silence = defer all.
```

The user's responses map to three buckets:

- **Answered** → folded into the section rewrite.
- **Deferred** → carried forward to the decision record under
  *Open questions*.
- **Out of scope** → added to *Boundaries → Out of scope* in the
  feature file, verbatim-ish, so the exclusion is explicit.

Ambiguities are re-surfaced once at Wrap-up if they are still open.
They are never silently dropped.

### Full-section rewrite rule

Section updates are **replacements, never patches**. When a phase is
confirmed, the sub-skill composes the entire section's prose from the
accumulated answers, shows it to the user for confirmation, and on
confirmation writes the whole section into the feature file —
replacing whatever prior content that section held.

Consequences:

- A `/back` into a prior phase followed by edits may invalidate later
  sections. When the user revises Scope or Happy path, the sub-skill
  must re-confirm that the subsequent confirmed sections are still
  correct — do **not** auto-edit them, and do **not** leave them
  silently stale. Offer: "Edge cases was confirmed earlier under a
  different Scope. Keep as-is, revise, or discard?"
- Never hand-merge user feedback into a diff. If the user says "change
  X to Y" after seeing a proposed section, rewrite the whole section
  and re-present.

### Session state

The session is in-memory. The only durable artifacts are the feature
file and the decision record, both written only on phase confirmation
or on `/abort`. The sub-skill does not maintain a sidecar session log
beyond the decision record. Resumption across sessions is out of scope
for this step.

## Elicit mode

### When to invoke

Via Spezi, when the router emits:
`{ kind: "invoke", targets: [{ skill: "gherkin", mode: "elicit" }], seed }`.

The `seed` is a free-form one-liner (e.g. "add SSO to login"). It may
be empty — in which case the first question of the Scope phase asks
what the feature is.

### Preconditions

- Writable project root.
- `specs/gherkin/` and `specs/gherkin/decisions/` creatable.
- No prior `.feature.md` with the same prospective slug — if detected
  after Scope is confirmed, ask whether to **revise** the existing
  file (which switches behavior to distill mode, currently deferred —
  so decline and ask the user to pick a different scope), **replace**
  it (requires an explicit "yes, replace"), or **rename** the new
  feature's slug.

### Four-phase structure

Each phase follows the same shape:

1. **Open**: state the phase's goal in one sentence.
2. **Batch**: ask 3–7 questions.
3. (Optional) **Follow-up batch**: one more batch if critical gaps.
4. **Flag**: present the ambiguity list if non-empty.
5. **Propose**: compose the full section and show it.
6. **Confirm**: await explicit confirmation (`yes`, `looks good`,
   `confirm`) or a revision. Escape hatches always usable here.
7. **Persist**: on confirmation, rewrite the section in the feature
   file and append resolved decisions to the in-memory decision log.

Phase-specific content:

#### Phase 1 — Scope

Goal: name the feature and its boundaries.

Batch topics (select 3–7):
- Feature name / one-line summary.
- Primary actor (end user, admin, system, external caller).
- Trigger — what initiates the behavior.
- Success criterion — how do we know it worked.
- Out-of-scope items the user knows up front.
- Dependencies or prior specs this relies on.
- Non-functional constraints the scope hinges on (latency budget,
  compliance, platform).

Section output (in `.feature.md`):

```
# <Feature title>

<One- or two-sentence prose summary. Actor, trigger, success.>

## Boundaries

- **In scope**: <bulleted list>
- **Out of scope**: <bulleted list; empty list is written as "- none declared" to make the absence explicit>
- **Depends on**: <bulleted list of prior specs or systems, or "- none">
```

Persistence: on Scope confirmation, the slug is fixed, the feature
file is created, and the decision record is initialized.

#### Phase 2 — Happy path

Goal: the canonical success scenario(s) in Given/When/Then form.

Batch topics:
- Starting state (Given).
- User/system action (When).
- Observable outcome (Then).
- Pre-existing data or setup assumptions.
- Any branching that is still part of the "happy" flow vs. an edge
  case (the boundary is the user's to decide).
- Naming for the scenario(s).

Section output:

```
## Scenario: <Happy path — short name>

Given <precondition>
And <precondition>
When <action>
Then <observable outcome>
And <observable outcome>
```

Multiple happy scenarios allowed; keep them narrow and named.

#### Phase 3 — Edge cases

Goal: failure modes, alternatives, and invariants.

Batch topics:
- Failure modes for each step of the happy path.
- Invalid inputs and their handling.
- Concurrency, timing, or ordering concerns.
- Permission / authorization boundaries.
- Invariants that must hold across scenarios.
- Explicit non-behaviors (things that must *not* happen).

Section output: one `## Scenario: <Edge case — name>` block per case,
in the same Given/When/Then form. Invariants that span scenarios go
under a dedicated block:

```
## Invariants

- <invariant sentence>
- <invariant sentence>
```

#### Phase 4 — Wrap-up

Goal: surface remaining ambiguities, confirm the whole document, and
finalize the decision record.

Flow:
1. Re-present any still-open items from the running ambiguity list.
2. Show the full current feature file.
3. Ask: "Confirm as-is, revise a specific section (`/back`), or
   `/abort`?"
4. On confirmation: append any final decisions to the decision record,
   set `Outcome: completed`, and write the record.

No new batch of questions is asked in Wrap-up; the user's attention
is on the whole document.

### Completion criteria

The session is **complete** when all four phases have been confirmed
and the Wrap-up has been accepted. It is **aborted** on `/abort`. It
is **draft** if the agent's environment ends the session before
Wrap-up (e.g. the user walks away) — in that case, flush whatever
has been confirmed to disk with a draft marker in the feature file
front matter.

### Non-goals for elicit

- No implementation hints, file scaffolding, or code suggestions.
- No cross-spec refactoring.
- No opinions on testing framework, CI, or runtime.
- No silent defaults: every decision is either confirmed, deferred,
  or declared out of scope.

## File formats

### `.feature.md`

Front matter (YAML) is optional but recommended. When present, it
carries only bookkeeping — not behavior.

```
---
slug: <slug>
status: draft | complete | aborted
created: <YYYY-MM-DD>
session: <path to decision record>
---

# <Feature title>

<Prose summary.>

## Boundaries
- **In scope**: ...
- **Out of scope**: ...
- **Depends on**: ...

## Background
Given <shared preconditions, optional>

## Scenario: Happy path — <name>
Given ...
When ...
Then ...

## Scenario: Edge case — <name>
Given ...
When ...
Then ...

## Invariants
- ...
```

Rules:
- Gherkin keywords (`Given`, `When`, `Then`, `And`, `But`) appear at
  the start of a line, capitalized.
- Scenarios are `##` headings prefixed with `Scenario:` so standard
  markdown renderers group them and grep can find them.
- One sentence per step. Nested behavior goes into its own scenario.
- No implementation detail in steps (no "click button X at selector
  Y") — the sub-skill must push back when a user supplies one, and
  ask for the behavior it represents.

### Decision record

```
# <Feature title> — elicit session, <YYYY-MM-DD>

- **Slug**: <slug>
- **Feature file**: specs/gherkin/<slug>.feature.md
- **Outcome**: completed | aborted | draft
- **Started**: <ISO 8601 timestamp>
- **Ended**: <ISO 8601 timestamp>

## Decisions

- **<topic>**: <decision> *(phase: <scope|happy|edge|wrap>)*
- ...

## Deferred / open questions

- <question> *(flagged in <phase>; will be picked up in distill)*
- ...

## Out of scope (explicit)

- <exclusion>
- ...

## Session trail

- `/back` from edge → scope at <timestamp>: <one-line why>
- `/skip` on scope question (d) at <timestamp>
- ...
```

The *Session trail* is a short chronological log of notable control
events — not a transcript. It exists so a future reader can tell
which phase was revisited and why.

## Worked example (abridged)

Seed: `add SSO to login`.

**Scope batch**
> a) What's the one-line feature summary? b) Which identity
> providers? c) Actor — end user only, or also admins? d) Is
> password login retained alongside SSO? e) Out-of-scope items you
> already know? f) Any compliance constraint (e.g. SAML assertions
> must be signed)?
>
> /done if enough; /skip <letter> to skip one.

User: `a) SSO login for enterprise users. b) Okta + Azure AD. c) end users. d) yes, retained. e) SCIM provisioning — separate feature. f) SAML must be signed.`

Ambiguities flagged: `1. [scope] Session lifetime — same as password
sessions or different? 2. [scope] MFA — inherited from IdP or
enforced locally?`

User: `1: same. 2: defer.`

Proposed Scope section shown; user confirms. Slug fixed as
`sso-login-for-enterprise-users`. Feature file and decision record
created. Advance to Happy path.

(Happy path, Edge cases, Wrap-up proceed analogously.)
