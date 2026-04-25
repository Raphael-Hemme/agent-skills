---
name: spezi-elicit
description: Run a structured discovery session to build a new `.feature.md` through conversation. Use when the user wants to create a new spec from scratch, elicit or gather requirements, capture domain behaviour, specify a feature or system, define what a system should do, or is describing functionality and needs help shaping it into a behavioural specification. Workshop, not generator — confirm every non-trivial decision before persisting; never invent behaviour. For extracting a spec from existing code use `/spezi-distill`; for targeted edits use `/spezi-tend`. After Wrap-up, offer to invoke `/spezi-propagate`.
---

# spezi-elicit

Run an interactive elicit session. Seed (free-form one-liner like `add SSO to login`) is the starting point and may be empty — if it is, the first phase asks what the feature is.

## Reading the initial prompt

Before opening Phase 1, classify what the user brought:

| User arrived with | Start at |
|---|---|
| Vague idea, no entities yet | Phase 0 (process discovery) |
| Process described, entities named | Phase 1 (scope) |
| Existing code | Exit, suggest `/spezi-distill` |
| Existing `.feature.md` they want changed | Exit, suggest `/spezi-tend` |

## Output

- Feature file: `specs/gherkin/<slug>.feature.md` — match `spezi/reference/feature-file.template.md`.
- Decision record: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>.md` — base shape from `spezi/reference/decision-record.template.md`.

Create `specs/gherkin/` and `specs/gherkin/decisions/` lazily on first write. Never overwrite an existing feature file without explicit user confirmation. If the prospective slug collides, ask whether to **rename** the new feature, **replace** the old one (requires `yes, replace`), or exit and run `/spezi-distill` instead.

## Slug derivation (fix only after Scope confirms)

1. Lowercase ASCII.
2. Replace runs of non-alphanumeric characters with `-`.
3. Strip leading/trailing `-`.
4. Truncate to 48 chars at a word boundary.

Examples: `"SSO login for enterprise users"` → `sso-login-for-enterprise-users`.

## Escape hatches (recognise at any prompt)

- `/done` — current phase / flag list is sufficient. Advance using whatever answers exist; missing answers become open questions in the record. On a flag list, unanswered = `defer`.
- `/skip <letter>` skips one labelled question; bare `/skip` skips the phase and logs a gap.
- `/back` returns to the prior phase; current in-progress answers are discarded, prior confirmed sections stay. From Scope, `/back` is a no-op synonym for `/abort` — say so.
- `/abort` ends the session. Persist confirmed sections as `status: aborted` with a draft marker; write the record with `Outcome: aborted`. Do not delete partial work.

A user who literally needs `/done` in an answer can quote it (`"/done"`). Any other unquoted leading-`/` token: ask, don't guess.

## Batch Q&A shape

Every phase opens with a labelled batch:

- 3–7 questions, labelled `a)`, `b)`, ... .
- Closing line: `Answer whichever you can in any order. /done if we have enough; /skip <letter> to skip one; /skip alone to skip the phase.`
- After the user answers, one optional follow-up batch if critical gaps remain. Never chain more than two batches per phase — flag the rest as ambiguities and move on.

## Ambiguity-flag pattern

```
Flagged for your attention (Scope | Happy path | Edge cases | Wrap-up):
1. [topic] <one-sentence observation or question>
2. [topic] <observation>

Reply with `1: answer ...`, `1: defer`, `1: out of scope`, or free-form.
Multiple on one line OK. Silence = defer all.
```

Items still open at end of phase get re-surfaced once at Wrap-up; otherwise logged to the record's *Deferred / open questions*. Never silently drop.

## Full-section rewrite

Section updates are **replacements, never patches**. Compose the whole section's prose from accumulated answers, show it for confirmation, on confirm write the entire section into the file. If the user says "change X to Y" after seeing a draft, rewrite the whole section and re-present — never hand-merge a diff. After a `/back` revises Scope or Happy path, ask whether previously-confirmed later sections are still correct (Keep / Revise / Discard) — never auto-edit them.

## Phases

Each phase: **Open** (state goal) → **Batch** → optional follow-up batch → **Flag** (any ambiguities) → **Propose** (full section) → **Confirm** (`yes`/`looks good`/`confirm`) → **Persist** (full-section rewrite). Phase 0 is optional; Phase 4 skips Batch.

| Phase | Goal | Batch topics (pick 3–7) | Section output |
|-------|------|-------------------------|----------------|
| 0 — Process discovery (optional) | Surface the larger process the feature lives in before homing in. Skip if user arrives with entities. | what triggers this work?; who's involved?; what state changes hands?; what marks the work done?; what other processes touch this? | No section yet — informs Phase 1 scoping. |
| 1 — Scope | Name the feature and its boundaries. | one-line summary; primary actor; trigger; success criterion; known out-of-scope items; dependencies on prior specs; non-functional constraints. | `# <Title>` + prose summary + `## Boundaries` with **In scope** / **Out of scope** / **Depends on**. Empty bullets → `- none declared`. |
| 2 — Happy path | Canonical success scenario(s). | Given (start state); When (action); Then (outcome); pre-existing setup; happy/edge boundary; scenario name(s). | One or more `## Scenario: Happy path — <name>` blocks. Multiple OK; keep each narrow. |
| 3 — Edge cases | Failure modes, alternatives, invariants. | failure mode per happy step; invalid inputs; concurrency/timing/ordering; permission boundaries; invariants; explicit non-behaviours. | One `## Scenario: Edge case — <name>` per case. Cross-scenario invariants → `## Invariants` bulleted block. |
| 4 — Wrap-up | Surface remaining ambiguities, confirm whole document, finalise the record. | *(no batch)* | Re-present open ambiguities → show full feature file → "Confirm as-is, revise a section (`/back`), or `/abort`?" → on confirm, set `status: complete`, write the record with `Outcome: completed`. |

## Library-spec candidates

If the user describes a generic integration pattern (OAuth, payment, email delivery, calendar sync, ATS sync, file storage, webhook handlers implementing third-party contracts), surface it as a flag at end of Scope:

```
1. [library candidate] OAuth flow looks reusable across features. Extract to specs/gherkin/lib/oauth-<provider>.feature.md?

Reply `1: extract`, `1: keep inline`, `1: defer`. Silence = keep inline.
```

`extract` → finish this elicit session for the main feature first, then suggest a follow-up `/spezi-elicit` for the library spec. Don't open a nested session.

## Calibration tests (run continuously through every phase)

- **The "Why?" test.** Why does the stakeholder care? If you can't answer, drop the detail.
- **The "Could it be different?" test.** Could it be implemented another way and still be the same system? If yes, it's implementation; redirect.
- **The "Template vs instance" test.** Is the user naming a category or a specific instance? Default to category; promote to instance only when the user-facing flow names it.
- **The "Obviously" trap.** Probe assumptions stated as obvious — they often aren't.
- **The "Missing actor" trap.** Every action needs an actor. If the user says "the system does X", ask which actor.

## Persistence

- On Scope confirmation: fix the slug, create the feature file (`status: draft`), initialise the decision record (`Started` timestamp, base sections empty).
- On each subsequent phase confirmation: rewrite that phase's section in full; refresh `updated`.
- Decision log is appended in-memory; flushed at Wrap-up. *Decisions* entries note the phase in italics: `*phase: scope*`.

## Completion

- **Complete** when all four phases confirmed and Wrap-up accepted: feature file gets `status: complete`, record gets `Outcome: completed`.
- **Aborted** on `/abort`: feature file gets `status: aborted`, record gets `Outcome: aborted`.
- **Draft** if the environment ends the session before Wrap-up: feature file keeps `status: draft`, record gets `Outcome: draft`.

## Linking

A fresh `.feature.md` is created with no `allium:` block. If during Scope the user names an existing Allium file, write the entry with `status: pending` and `linkedAt: now` — no further action. Never read, write, or delete `.allium` files from this skill.

## After Wrap-up — Propagate hook

If the just-written file has `status: complete` and at least one `## Scenario:`, ask:

```
Elicit complete. Generate tests for this spec via /spezi-propagate?

Reply `yes` / `not now` / `never this session`.
```

On `yes`, hand off with `{ seed: <slug> }`. `/spezi-propagate` decides Allium handoff vs. framework-native generation internally. Log outcome in the record's *Session trail*. Skip the hook if the user passed `--no-propagate`.

## What this skill does not do

- No implementation hints, file scaffolding, code suggestions.
- No extraction from existing code (run `/spezi-distill` for that).
- No edits to an existing spec (run `/spezi-tend` for that).
- No alignment-checking against code (run `/spezi-weed`).
- No opinions on testing framework or CI.
- No silent defaults: every decision is confirmed, deferred, or declared out of scope.
