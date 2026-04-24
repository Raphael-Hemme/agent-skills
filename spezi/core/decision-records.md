# Decision Records

A decision record is the per-session artifact that captures what was
decided, deferred, or ruled out. Every Spezi session that changes
artifacts writes one. This file is the canonical home for the
schema; sub-skills describe *when* to write a record but refer here
for *what* a record contains.

## Path convention

```
specs/gherkin/decisions/<YYYY-MM-DD>-<slug>[-<session-type>].md
```

- `<YYYY-MM-DD>` is the session's start date in UTC.
- `<slug>` matches the feature file slug.
- `<session-type>` is required for TDD sessions (`read`, `red`,
  `green`) and for distill when it would otherwise collide with a
  same-day elicit record. Optional for elicit (omit when unambiguous).

If a path with the composed name already exists, append `-<n>` where
`n` starts at `1` and increases until the name is unique. Never
overwrite an existing decision record.

Examples:

- `2026-04-24-sso-login.md` — elicit session (the default).
- `2026-04-24-sso-login-distill-1.md` — distill session on the same
  day as an earlier record.
- `2026-04-24-sso-login-red.md` — `--red` session.
- `2026-04-24-sso-login-green.md` — `--green` session.
- `2026-04-24-sso-login-read.md` — `--read` session that invoked
  `/update-spec` (pure reads write no record).

## Base schema

Every record, regardless of session type, begins with the shape below.
Headings are stable — tools may grep by heading name.

```
# <Feature title> — <session-type> session, <YYYY-MM-DD>

- **Slug**: <slug>
- **Session type**: elicit | distill | tdd-read | tdd-red | tdd-green
- **Feature file**: specs/gherkin/<slug>.feature.md
- **Outcome**: completed | aborted | draft
- **Started**: <ISO 8601 UTC timestamp>
- **Ended**: <ISO 8601 UTC timestamp>

## Decisions

- **<topic>**: <decision> *(phase: <scope|happy|edge|wrap>, if elicit)*
- ...

## Deferred / open questions

- <question> *(flagged in <phase>; picked up by distill or next session)*
- ...

## Out of scope (explicit)

- <exclusion>
- ...

## Session trail

- `<event>` at <timestamp>: <one-line why>
- ...
```

Field semantics:

- **Outcome**: `completed` iff the session's terminal confirmation
  step ran; `aborted` on explicit `/abort`; `draft` if the environment
  ended the session before the terminal step (user walked away,
  harness crash).
- **Decisions**: each resolved item, one line. For elicit, suffix
  the phase in italics so the reader can tell when it was decided.
- **Deferred / open questions**: cross-session. Distill and TDD
  sessions *append* to this section across records for the same slug
  — they never replace or edit prior entries. A reader assembling
  the full deferred list for a slug concatenates across records in
  chronological order.
- **Out of scope (explicit)**: items the user actively declared out
  of scope, verbatim-ish. Distinct from "deferred" — these are
  never coming back inside the current feature.
- **Session trail**: short chronological log of notable control
  events (`/back`, `/skip` at significant points, `/update-spec`
  invocations, hook responses). Not a transcript.

The base schema is sufficient for a pure elicit session. Every other
session type adds one or more sections below.

## Distill extension (spezi-gherkin)

A distill record carries the base schema plus:

```
- **Cycles**: <n>       # number of Draft ↔ Flag ↔ Redraft cycles

## Accepted suggestions

- [scenario: <name>] <summary of change>
- ...

## Rejected suggestions

- [scenario: <name>] <summary of proposal>: rejected by user
- ...
```

The `Accepted` and `Rejected` sections reflect the user's bucket
assignments during flag resolution. The *Deferred / open questions*
section from the base schema remains the single place for deferrals;
distill appends, never replaces.

## TDD extension (spezi-tdd)

A TDD record (`--read` with `/update-spec`, `--red`, `--green`)
carries the base schema plus whichever of the following sections
apply to the mode:

```
## Alignment outcomes       # --red only
- [scenario: <name>] generated → specs/allium/<slug>/<test>
- [scenario: <name>] deferred (reason: <text>)
- [scenario: <name>] handed off

## Pass/fail roster         # --green only
- [scenario: <name>] pass
- [scenario: <name>] fail → guidance delivered; user reports in progress
- [scenario: <name>] fail → `/update-spec` invoked

## Spec updates             # any TDD mode, if /update-spec invoked
- [scenario: <name>] <summary of accepted change>
  Cross-ref: specs/gherkin/decisions/<YYYY-MM-DD>-<slug>[-<type>].md
```

`Cross-ref` lines point at related decision records (typically the
most recent distill or elicit record for the same slug). The TDD
session does **not** edit the cross-referenced record; the reference
is one-way.

A `--read` session writes a record **only** if `/update-spec` was
invoked. A pure read is not a decision.

## Cross-session continuity

Deferred questions survive across sessions by appending to each
record's *Deferred / open questions* section rather than threading a
single shared list. Rationale: each record stays standalone
(readable without context) and the chronological read order across
records gives the full history.

When distill or TDD picks up a deferral from a prior record:

- The picking-up session's *Decisions* section records the
  resolution.
- The picking-up session does **not** edit the prior record to
  strike the deferral. Prior records are immutable.
- Readers who want "currently open questions for slug X" filter
  across all records for that slug, subtracting any deferrals later
  resolved in *Decisions* entries.
