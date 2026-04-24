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
| `distill` | **Specified below** | Tighten or restructure an existing `.feature.md` — draft-first, then grouped flags. |
| `read` | Specified in `spezi/tdd/SKILL.md` | Read-only reference for a spec. Gherkin-only target. |
| `red` / `green` | Specified in `spezi/tdd/SKILL.md` | TDD phases. Gherkin supplies the spec and alignment analysis; Allium handles tests. |
| `refactor` | Deferred | TDD-cycle refactor operations over specs and implementation. |

A conforming implementation of this file must not silently run deferred
modes — if the router passes one, respond with "not yet implemented."
The TDD modes defer their cross-sub-skill protocol to
`spezi/tdd/SKILL.md`; Gherkin's Allium-linking rules and full-section
rewrite rule are honored unchanged by that protocol.

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

### Router flag interpretation

The sub-skill receives the `ParsedInvocation` alongside the routed
mode. From the sub-skill's point of view, three parser flags matter
for its own behavior (everything else is a routing concern):

| Flag observed | Effect inside Gherkin |
|---------------|-----------------------|
| `--gherkin` explicit | Gherkin-only intent. Suppress the Allium assessment hook entirely. Existing `allium` references in the feature file are still surfaced and respected (e.g. flagged as stale in distill), but no suggestion to invoke Allium is made. |
| `--both` explicit | Gherkin runs normally. The Allium assessment hook is suppressed because the router has already dispatched to Allium; Gherkin instead emits a non-prompting *handoff signal* on completion (see *Allium assessment hook*). |
| `--allium` explicit | Not reachable here — the router would not route to Gherkin. If somehow received, treat as a bug and refuse to proceed. |
| None of the above | Default. The Allium assessment hook may fire per its own rules. |

These overrides only affect the Allium-facing surface. They do not
change the four-phase elicit flow, the distill draft-first flow,
the ambiguity pattern, or any output format.

### Allium assessment hook

The hook is the moment the Gherkin sub-skill pauses and asks whether
Allium should be brought in. It is a *suggestion*, not a handoff;
the user opts in.

**When it fires.** The hook is evaluated at exactly one point per
mode and fires at most once per session:

- Elicit — immediately after the Wrap-up final confirmation, before
  finalizing the decision record's `Outcome` field.
- Distill — immediately after the final `confirm` of the distilled
  document, before writing the decision record.

It never fires at mid-session points (phase transitions, draft
reveals, ambiguity resolutions), on `/abort`, on draft-outcome
endings, or when the feature file ends without a `Scenario:` block.

**Gating conditions (all must hold).** If any fails, the hook is
silent and the session completes normally.

1. The routing environment reports `alliumAvailable == true`.
2. The parsed flags do not contain `--gherkin` or `--both`.
3. The user has not declined Allium earlier in this session (see
   *Decline memory*).
4. The feature file has at least one `Scenario:` block.

**Hook prompt shape.**

```
Allium assessment: Allium is available and this spec is ready for
handoff. Invoke Allium now to <one-line role summary>?

Reply `yes` / `not now` / `never this session`.
```

The one-line role summary comes from the adapter's
`describeAllium()` when present, otherwise falls back to
`"continue with the Allium-side workflow"`. The sub-skill does not
invent specifics.

**User responses.**

- `yes` → return control to Spezi with a structured handoff request:
  `{ invoke: "allium", seed: <slug>, reason: "post-gherkin-<mode>" }`.
  The Gherkin sub-skill does not invoke Allium directly; the router
  does.
- `not now` → no handoff this session; the hook does not re-fire.
- `never this session` → same as `not now`, plus sets an in-memory
  suppression flag. (Persistent opt-out is a config concern for a
  later step.)

Both non-`yes` responses are recorded in the decision record's
*Session trail* as `allium-hook: declined (this session)`.

**Handoff signal under `--both`.** The prompt is skipped, but the
sub-skill still emits a non-interactive *handoff signal* on
completion: `{ signal: "gherkin-complete", slug, featureFile }`.
The adapter/router consumes this to sequence Allium if it is not
already running. No user turn is added.

### Decline memory

Decline memory is in-memory only, scoped to the current Spezi
invocation. A new Spezi invocation starts clean. This preserves the
dialogical principle across time: the sub-skill asks again tomorrow,
rather than silently remembering yesterday's "no."

### Linking to Allium artifacts

A `.feature.md` may declare references to Allium artifacts in its
front matter (schema: see *File formats → `.feature.md`*). The
Gherkin sub-skill's contract with Allium files is:

- **Structure is canonical in the feature file.** Allium files mirror
  the reference; they do not override it.
- **Round-trip.** Each linked Allium file *should* carry a `gherkin`
  back-reference (path + timestamp). Gherkin produces its own link
  with enough information for Allium to round-trip it, but does not
  enforce Allium's shape.
- **Staleness.** A reference is *stale* when the feature file's
  `updated` timestamp is newer than the reference's `linkedAt`.
  Distill detects and flags staleness; elicit preserves stale entries
  across a confirmed replace rather than silently dropping them.
- **No silent writes.** Gherkin never edits Allium files. It may flag
  missing-file, mismatched-slug, or stale references; reconciliation
  is Allium's job (typically through a follow-up `--both` run).

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

## Distill mode

Distill tightens or restructures an existing `.feature.md`. It
replaces elicit's four-phase structure with a **draft-first** loop:
propose a revised document, then surface a grouped list of
meaning-changing observations the sub-skill deliberately did *not*
apply on its own.

### When to invoke

Via Spezi, when the router emits:
`{ kind: "invoke", targets: [{ skill: "gherkin", mode: "distill" }], seed }`.

The `seed` identifies which feature to distill. It may be:

- A slug (`sso-login`).
- A path (`specs/gherkin/sso-login.feature.md`).
- Free-form text naming the feature (`the SSO login spec`).

Resolution:

- If the seed resolves to exactly one existing feature file → proceed.
- If empty or ambiguous → the sub-skill lists candidates (up to 10,
  newest first) and asks the user to pick. It never distills a file
  the user did not clearly name.
- If it resolves to zero candidates → the sub-skill reports nothing
  found and exits (no file written, no decision record).

### Preconditions

- The target `.feature.md` exists and parses — front matter optional,
  but scenarios, boundaries, and steps must be present.
- `specs/gherkin/decisions/` is writable.
- Any existing `allium` references are read into memory as context.
  Distill never edits Allium files; see *Linking to Allium artifacts*.

### Draft-first flow

A single loop: Read → Draft → Flag → Resolve → Redraft → Confirm.

1. **Read.** Load the feature file and the most recent decision
   record for its slug, if any. Open with a one-line synopsis:
   > Distilling *<slug>*, last updated <date>, <N> scenarios.
   > Ready? (`yes` / `/abort`)

   `/back` at this point is a no-op synonym for `/abort` since there
   is no prior phase.

2. **Draft.** Compose a revised full `.feature.md` applying only
   *low-risk tightenings* (see *Draft guardrails*). Present the whole
   proposed document to the user, preceded by a short "What I
   touched" summary of categories — **not** a line diff. Example:
   > What I touched: normalized Gherkin capitalization; split three
   > run-on steps; refreshed `updated` timestamp; added `- none
   > declared` to an empty *Out of scope*.

3. **Flag.** Immediately after the draft, present a grouped list of
   meaning-changing observations the sub-skill did **not** fold into
   the draft. Same visual shape as the elicit ambiguity list:

   ```
   Flagged (not applied):
   1. [scenario: <name>] Steps appear to cover two behaviors —
      split into two scenarios?
   2. [scope] "Depends on" lists `auth-v2` which has no feature file.
      Update reference or drop?
   3. [invariants] "Session tokens never exceed 1 hour" restates
      scenario content — remove from invariants?
   4. [allium] specs/allium/sso-login.allium is stale (feature file
      newer by 12d). Mark as stale, leave as-is, or unlink?

   Reply with `1: accept`, `1: reject`, `1: defer`, or
   `1: <amended>` for each. Silence = reject all (leave as-is).
   ```

4. **Resolve.** Map each reply to a bucket:
    - `accept` → fold the change into the next draft; log under
      *Accepted suggestions* in the decision record.
    - `reject` → leave the draft as-is; log under *Rejected
      suggestions*.
    - `defer` → log under *Deferred / open questions* (shared with
      elicit; distill appends, never replaces).
    - `<amended>` → the user's variant replaces the proposal; fold in
      and log under *Accepted suggestions* with the amended wording.

5. **Redraft.** If any flags resolved to `accept` or `<amended>`,
   compose a second full proposal and show it. Repeat from Flag with
   any newly introduced observations. If no flags changed state, skip
   to Confirm.

6. **Confirm.** The user types `confirm` / `looks good` / `yes`, or
   requests another revision (freeform critique → redraft). On
   confirmation: write the feature file, write the decision record,
   evaluate the Allium assessment hook.

A distill session may iterate Draft ↔ Flag ↔ Redraft up to **three
cycles**. A fourth unresolved cycle forces the sub-skill to stop and
ask the user whether to commit the current state as-is or `/abort` —
this is the distill-side equivalent of the elicit "two-batch" cap.

### Draft guardrails

The initial draft applies only changes the user can reasonably expect
without being asked. The boundary:

**Apply silently (low-risk).**
- Gherkin keyword casing (`given` → `Given`).
- One-sentence-per-step formatting; splitting run-on steps where the
  split is mechanical (conjunction-driven, no semantic inference).
- Consistent scenario heading prefix `Scenario:`.
- Empty boundary lists → `- none declared`.
- Front-matter key ordering, `updated` timestamp refresh.
- Stripping trailing whitespace, normalizing blank lines.
- Typo fixes in Gherkin connective words only (`Whn` → `When`).
  Typos in user prose (summary, boundaries, step content) are
  **not** silently corrected — propose them as flags.

**Never apply without a flag (meaning-changing).**
- Adding, removing, merging, or splitting scenarios.
- Renaming actors, triggers, or scenarios.
- Changing step wording beyond keyword casing.
- Altering or adding invariants.
- Changing *In scope* / *Out of scope* / *Depends on* membership.
- Modifying `allium` references (add, remove, status change).
- Rewriting the prose summary paragraph in a way that could change
  scope.

A proposal a reader might reasonably interpret two ways counts as
meaning-changing — flag it.

### Escape hatches in distill

- `/done` — at the Flag step, treat all un-answered flags as
  `reject` and proceed to Redraft (with no changes) → Confirm.
- `/skip` — at the Flag step, skip an individual numbered item by
  `/skip <n>`; it is recorded as deferred. Bare `/skip` skips all
  open flags (synonym for `/done`).
- `/back` — from Flag or Redraft, return to the immediately prior
  draft. The sub-skill re-presents the prior proposal and the prior
  flag list; any `accept`/`<amended>` decisions for the discarded
  draft are cleared. `/back` from Read exits the session (see step 1).
- `/abort` — persist nothing. The original `.feature.md` is left
  untouched. A decision record is still written, with
  `Outcome: aborted` and any flags that were presented logged under
  *Rejected suggestions* with a `(session aborted)` note.

### Distill decision record

Same path convention as elicit:
`specs/gherkin/decisions/<YYYY-MM-DD>-<slug>.md`. If a same-day
record already exists, append a suffix `-distill-<n>` rather than
overwriting (e.g. `2026-04-24-sso-login-distill-1.md`).

Body uses the shared decision-record shape plus two distill-specific
sections and a `session-type` marker:

```
# <Feature title> — distill session, <YYYY-MM-DD>

- **Slug**: <slug>
- **Session type**: distill
- **Feature file**: specs/gherkin/<slug>.feature.md
- **Outcome**: completed | aborted
- **Started**: <ISO 8601 timestamp>
- **Ended**: <ISO 8601 timestamp>
- **Cycles**: <n>

## Accepted suggestions

- [scenario: <name>] <summary of change>
- ...

## Rejected suggestions

- [scenario: <name>] <summary of proposal>: rejected by user
- ...

## Deferred / open questions

- <question>  *(this section is shared across elicit and distill
  records for the same slug; distill appends)*
- ...

## Session trail

- ...
```

### Non-goals for distill

- No new behavior elicitation. If distill notices that a whole area
  is missing, it flags that *once* as an observation; it does not
  open an elicit-style question batch.
- No step reordering within a scenario when the order carries
  semantics. Given/When/Then ordering is a floor, not a ceiling.
- No Allium file edits. Stale Allium links are flagged here; the
  actual reconciliation happens in Allium (or a follow-up `--both`
  run, which the Allium assessment hook facilitates).

## File formats

### `.feature.md`

Front matter (YAML) is optional but recommended. When present, it
carries only bookkeeping and linkage — not behavior.

```
---
slug: <slug>
status: draft | complete | aborted
created: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
session: <path to decision record>
allium:
  - path: specs/allium/<slug>.allium
    kind: tests | plan | generated-from
    linkedAt: <ISO 8601 timestamp>
    status: active | stale | pending
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
- `updated` is refreshed on every successful write (elicit
  confirmations and distill confirms). `created` is written once.

**`allium` block — reference field format.**

The `allium` key holds an ordered list of references. Each entry is
an object with the following fields:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `path` | relative POSIX path, string | yes | Location of the linked Allium artifact, relative to the project root. Must point inside `specs/allium/`. |
| `kind` | enum string | yes | Role of the linked file. Canonical values: `tests` (Allium file tests this spec), `plan` (Allium-side test plan derived from this spec), `generated-from` (Allium file was generated from this spec). Unknown values are preserved verbatim for forward compatibility but may be reported at read time. |
| `linkedAt` | ISO 8601 UTC timestamp | yes | When the link was last established or reconciled. Not necessarily the Allium file's mtime. |
| `status` | enum string | yes | `active` — link is current. `stale` — the feature file has been updated after `linkedAt`. `pending` — the link was declared but the Allium file does not yet exist (intent, not fact). |

An empty list (`allium: []`) and a missing `allium` key are
equivalent on read. On write, prefer omitting the key when there are
no references.

The Allium-side back-reference is expected to carry a `gherkin` field
with `path` and `updatedAt`. Gherkin reads this (if present) to
verify round-trip integrity but does not modify it.

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
