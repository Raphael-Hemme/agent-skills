---
name: spezi-gherkin
description: Dialogical spec authoring for Behavioral Spec Driven Development. Runs elicit and distill sessions to produce `.feature.md` specs alongside a decision record. Invoked by Spezi — not directly. Allium orchestration (when to invoke Allium, linking schema, decline memory) lives in the Spezi meta-skill, not here.
---

# spezi-gherkin

## Purpose

Author and maintain behaviour specs in a markdown-embedded Gherkin
style. The sub-skill is a workshop, not a generator: it refuses to
invent behaviour on the user's behalf and confirms every non-trivial
decision before persisting it.

The sub-skill is harness-agnostic and contains no Allium logic. It
receives `{ mode, seed }` from Spezi, runs its session, writes its
artifacts to disk, and exits. Spezi inspects the result and decides
what happens next (including any Allium hook) — see
`spezi/core/allium.md`.

## Modes

| Mode | Status | Summary |
|------|--------|---------|
| `elicit` | **Specified below** | Interactive session to produce a new `.feature.md` from a seed idea. |
| `distill` | **Specified below** | Tighten or restructure an existing `.feature.md` — draft-first, then grouped flags. |

TDD phase modes (`--read`, `--red`, `--green`, `--refactor`) are
specified in the sibling `spezi-tdd` skill, not here. Spezi routes
those flags to `spezi-tdd` directly; this sub-skill is not invoked
for them.

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

### Shared dialogical patterns

Elicit and distill use the same escape-hatch vocabulary (`/done`,
`/skip`, `/back`, `/abort`), the same batch Q&A shape (3–7 labeled
questions, two-batch cap), the same ambiguity-flag presentation
(grouped numbered list, per-context buckets), the same full-section
rewrite rule (replace, never patch), and the same in-memory session
contract. These are specified once in `spezi/core/patterns.md`.

Sub-skill narrowings:

- Elicit's ambiguity list uses the `answer` / `defer` / `out of scope`
  buckets (default on silence: defer all).
- Distill's flag list uses the `accept` / `reject` / `defer` /
  `<amended>` buckets (default on silence: reject all).
- Distill's `/back` from Read is a no-op synonym for `/abort` (Read
  has no prior step).
- Ambiguities in elicit are re-surfaced once at Wrap-up if still
  open; they are never silently dropped.

### Linking behaviour

The schema of the front-matter `allium` block, the round-trip
back-reference contract, and everything about *when* Allium gets
invoked belong to Spezi — see `spezi/core/allium.md`. This
sub-skill only defines how Gherkin operations *read and write* that
block.

- **Elicit.** A fresh `.feature.md` is created with no `allium`
  block. If the user's declared scope includes an existing Allium
  file (supplied as a scope answer), the field is created with
  `status: pending` and `linkedAt: now` — no further action.
- **Elicit replace.** When the user explicitly replaces an existing
  `.feature.md` during Scope confirmation, any existing `allium`
  entries are **preserved verbatim** and each entry's `status` is
  coerced to `stale` (the prior links may be out of date relative
  to the new content). Never drop entries silently.
- **Distill.** Each `allium` entry whose `linkedAt` is older than
  the feature file's `updated` timestamp is flagged to the user as a
  meaning-changing observation (same grouped-flag pattern as any
  other distill observation). Accepted flags may change the entry's
  `status` (e.g. to `stale`) or remove the entry; rejected flags
  leave it as-is. Gherkin never unilaterally edits or deletes an
  entry.
- **Never.** Gherkin never reads, writes, or deletes `.allium`
  files. If an Allium-side edit is appropriate, it is Spezi's or
  Allium's concern — see `spezi/core/allium.md` §Allium-side
  reconciliation.

The Allium assessment hook (when to suggest invoking Allium after a
successful session) is fully external to this sub-skill. Gherkin
writes `status: complete` on successful elicit Wrap-up or distill
Confirm, and that is the only handshake Spezi needs.

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

Each of Phases 1–3 follows the shape: **Open** (state goal) → **Batch**
(3–7 questions per `patterns.md` §Batch Q&A) → optional **Follow-up
batch** → **Flag** (present ambiguity list if non-empty) → **Propose**
(compose the full section) → **Confirm** (explicit `yes`/`looks good`/
`confirm`, or revision; escape hatches always usable) → **Persist**
(full-section rewrite per `patterns.md`). Phase 4 skips Batch; it
re-surfaces open ambiguities, shows the whole document, and writes
the decision record.

| Phase | Goal | Batch topics (pick 3–7) | Section output |
|-------|------|-------------------------|----------------|
| 1 — Scope | Name the feature and its boundaries. | Feature name / one-line summary; primary actor; trigger; success criterion; known out-of-scope items; dependencies on prior specs; non-functional constraints. | `# <Title>` + one-paragraph prose summary + `## Boundaries` with **In scope** / **Out of scope** / **Depends on** bullets. Empty boundary list is written as `- none declared` to make the absence explicit. |
| 2 — Happy path | Canonical success scenario(s) in Given/When/Then form. | Given (starting state); When (action); Then (outcome); pre-existing data / setup; happy-vs-edge boundary; scenario name(s). | One or more `## Scenario: <Happy path — name>` blocks with Given/When/Then/And steps. Multiple happy scenarios allowed; keep each narrow and named. |
| 3 — Edge cases | Failure modes, alternatives, invariants. | Failure mode per happy-path step; invalid inputs; concurrency/timing/ordering; permission boundaries; invariants; explicit non-behaviors. | One `## Scenario: <Edge case — name>` block per case. Cross-scenario invariants go under a dedicated `## Invariants` block (bulleted). |
| 4 — Wrap-up | Surface remaining ambiguities, confirm the whole document, finalize the decision record. | *(no batch)* | Re-present still-open ambiguities → show the full current feature file → ask "Confirm as-is, revise a specific section (`/back`), or `/abort`?" → on confirmation, append final decisions to the record, set `Outcome: completed`, and write. |

Persistence: on Scope confirmation the slug is fixed, the feature file
is created, and the decision record is initialized. Subsequent phase
confirmations rewrite their respective sections; the decision log is
appended in-memory and flushed at Wrap-up.

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
   record for its slug. Open with a one-line synopsis ("Distilling
   *<slug>*, last updated <date>, <N> scenarios. Ready? (`yes` /
   `/abort`)").
2. **Draft.** Compose a revised full `.feature.md` applying only
   low-risk tightenings (see *Draft guardrails*). Present the whole
   proposed document, preceded by a short "What I touched" summary
   of categories — **not** a line diff (e.g. "normalized Gherkin
   capitalization; split three run-on steps; refreshed `updated`
   timestamp; added `- none declared` to an empty *Out of scope*").
3. **Flag.** Present meaning-changing observations the sub-skill did
   **not** fold into the draft, using the grouped-flag pattern from
   `spezi/core/patterns.md`. Buckets: `accept` / `reject` / `defer` /
   `<amended>`; silence = reject all.
4. **Resolve.** `accept` / `<amended>` → fold the change into the
   next draft; log under *Accepted suggestions*. `reject` → leave the
   draft as-is; log under *Rejected suggestions*. `defer` → log under
   *Deferred / open questions* (distill appends, never replaces).
5. **Redraft.** If any flags resolved to `accept` or `<amended>`,
   compose a second full proposal and repeat from Flag for any newly
   introduced observations. Otherwise skip to Confirm.
6. **Confirm.** User types `confirm` / `looks good` / `yes`, or
   critiques freeform (→ redraft). On confirmation: write the
   feature file, write the decision record, let Spezi evaluate the
   Allium hook.

A distill session may iterate Draft ↔ Flag ↔ Redraft up to **three
cycles**. A fourth unresolved cycle halts to ask "commit as-is or
`/abort`?" — the distill equivalent of elicit's two-batch cap.

### Draft guardrails

The initial draft applies only changes the user can reasonably expect
without being asked. Anything a reader might reasonably interpret two
ways is meaning-changing — flag it.

| Apply silently (low-risk) | Never apply without a flag (meaning-changing) |
|---------------------------|-----------------------------------------------|
| Gherkin keyword casing (`given` → `Given`) | Adding, removing, merging, or splitting scenarios |
| Splitting run-on steps where the split is mechanical (conjunction-driven, no semantic inference) | Renaming actors, triggers, or scenarios |
| Consistent `Scenario:` heading prefix | Changing step wording beyond keyword casing |
| Empty boundary lists → `- none declared` | Altering or adding invariants |
| Front-matter key ordering; `updated` timestamp refresh | Changing *In scope* / *Out of scope* / *Depends on* membership |
| Trailing whitespace; blank-line normalization | Modifying `allium` references (add, remove, status change) |
| Typo fixes in Gherkin connective words only (`Whn` → `When`) | Rewriting the prose summary in a way that could change scope |

Typos in user prose (summary, boundaries, step content) are **not**
silently corrected — propose them as flags.

### Distill decision record

Schema and path convention (including the same-day
`-distill-<n>` suffix) live in `spezi/core/decision-records.md`
§Distill extension. Distill writes the base schema plus *Accepted
suggestions*, *Rejected suggestions*, and a `Cycles` count.

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

The `allium` block — fields, value enumerations, and the round-trip
back-reference contract — is defined in `spezi/core/allium.md`
§Linking convention. This sub-skill writes that schema verbatim and
applies the read/write rules in §Linking behaviour above; it does
not re-specify the schema here.

### Decision record

Schema, path convention, and cross-session continuity live in
`spezi/core/decision-records.md`. Elicit writes the base schema; a
successful session ends with `Outcome: completed`, an aborted one
with `Outcome: aborted`, and an environment-terminated one with
`Outcome: draft`.

## Worked example (abridged)

Seed: `add SSO to login`. Scope batch asks six labeled questions
(summary, providers, actor, password-retention, known out-of-scope,
compliance). The user answers most inline; remaining gaps surface
as an ambiguity list (session lifetime, MFA ownership). The user
resolves `1: same` and `2: defer`, the proposed Scope section is
confirmed, the slug `sso-login-for-enterprise-users` is fixed, and
the feature file plus decision record are created. Happy path, Edge
cases, and Wrap-up proceed analogously.
