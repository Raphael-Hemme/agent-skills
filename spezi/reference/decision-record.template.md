# Decision record template

Path: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>[-<type>].md`.

- `<type>` is required for distill, tend, weed, propagate; optional for elicit (omit if it does not collide on the same day).
- If the composed name already exists, append `-<n>` starting at `1` until unique. Never overwrite.
- Records are immutable across sessions: a later session **appends** to its own new record. Never edit prior records.

## Base shape — every session writes this

```
# <Feature title> — <session-type> session, <YYYY-MM-DD>

- **Slug**: <slug>
- **Session type**: elicit | distill | tend | weed | propagate
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

## Scope decisions            # what was deliberately excluded from extraction
- excluded: <category> (legacy | infra | deprecated | experimental | other-domain)

## Accepted suggestions
- [scenario: <name>] <summary of change>

## Rejected suggestions
- [scenario: <name>] <summary of proposal>: rejected by user

## Library spec candidates
- [<pattern>] extracted | kept inline | deferred
```

## Tend — base + these sections

```
- **Cycles**: <n>            # Draft ↔ Flag ↔ Redraft cycles, max 3

## Accepted suggestions
- [scenario: <name>] <summary of change>

## Rejected suggestions
- [scenario: <name>] <summary of proposal>: rejected by user

## Library spec candidates    # if any surfaced this session
- [<pattern>] extracted | kept inline | deferred
```

## Weed — base + whichever apply

```
- **Mode**: check | update-spec | update-code

## Alignment table            # always
- [scenario: <name>] match | drift | code-only | spec-only — <one-line note>

## Divergence classification  # always
- [scenario: <name>] spec bug | code bug | aspirational | intentional gap

## Spec updates               # update-spec mode only
- [scenario: <name>] <summary of accepted change>
  Cross-ref: specs/gherkin/decisions/<YYYY-MM-DD>-<slug>[-<type>].md

## Guidance issued            # update-code mode only
- [scenario: <name>] <one-line summary of guidance block>
```

## Propagate — base + these sections

```
- **Path taken**: allium-handoff | framework-native
- **Framework**: <pytest-bdd | cucumber-js | godog | ...>     # framework-native only

## Allium handoff             # allium-handoff path only
- artifact: <path>
- payload: { mode: post-gherkin, seed: <slug>, featureFile: <path> }

## Tests generated            # framework-native only
- [scenario: <name>] tests/features/<file>::<step ids>
- [invariant: <name>] tests/features/<file>::<step ids> | TODO

## Reused
- [scenario: <name>] tests/<existing-test>: covered, skipped generation
```

`Cross-ref` lines point at related earlier records (typically the most recent elicit / distill / tend record). One-way reference — the current session does **not** edit the cross-referenced record.
