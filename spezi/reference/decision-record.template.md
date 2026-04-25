# Decision record template

Path: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>[-<type>].md`.

- `<type>` is required for distill, tdd-read, tdd-red, tdd-green; optional for elicit (omit if it does not collide with another record on the same day).
- If the composed name already exists, append `-<n>` starting at `1` until unique. Never overwrite.
- Records are immutable across sessions: a later session **appends** to its own new record. Never edit prior records.

## Base shape — every session writes this

```
# <Feature title> — <session-type> session, <YYYY-MM-DD>

- **Slug**: <slug>
- **Session type**: elicit | distill | tdd-read | tdd-red | tdd-green
- **Feature file**: specs/gherkin/<slug>.feature.md
- **Outcome**: completed | aborted | draft
- **Started**: <ISO 8601 UTC timestamp>
- **Ended**: <ISO 8601 UTC timestamp>

## Decisions
- **<topic>**: <decision> *(phase: <scope|happy|edge|wrap>, elicit only)*

## Deferred / open questions
- <question> *(flagged in <phase>)*

## Out of scope (explicit)
- <exclusion>

## Session trail
- `<event>` at <timestamp>: <one-line why>
```

`Outcome`: `completed` if the terminal confirmation step ran, `aborted` on `/abort`, `draft` if the environment ended the session early.

## Distill — base + these sections

```
- **Cycles**: <n>            # Draft ↔ Flag ↔ Redraft cycles, max 3

## Accepted suggestions
- [scenario: <name>] <summary of change>

## Rejected suggestions
- [scenario: <name>] <summary of proposal>: rejected by user
```

## TDD — base + whichever apply

```
## Alignment outcomes        # spezi-red only
- [scenario: <name>] generated → specs/allium/<slug>/<test>
- [scenario: <name>] deferred (reason: <text>)
- [scenario: <name>] handed off

## Pass/fail roster          # spezi-green only
- [scenario: <name>] pass
- [scenario: <name>] fail → guidance delivered
- [scenario: <name>] fail → /update-spec invoked

## Spec updates              # any TDD mode that invoked /update-spec
- [scenario: <name>] <summary of accepted change>
  Cross-ref: specs/gherkin/decisions/<YYYY-MM-DD>-<slug>[-<type>].md
```

`Cross-ref` lines point at related earlier records (typically the most recent distill or elicit record for the same slug). One-way reference — the TDD session does **not** edit the cross-referenced record.

A `--read` session writes a record **only** when `/update-spec` was invoked. A pure read is not a decision.
