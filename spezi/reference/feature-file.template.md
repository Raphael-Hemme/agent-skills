# Feature file template

Path: `specs/gherkin/<slug>.feature.md`. Match this shape exactly.

```
---
slug: <slug>
status: draft | complete | aborted
created: <YYYY-MM-DD>
updated: <YYYY-MM-DD>
allium:
  - path: specs/allium/<slug>.allium
    kind: tests | plan | generated-from
    linkedAt: <ISO 8601 UTC timestamp>
    status: active | stale | pending
---

# <Feature title>

<One-paragraph prose summary.>

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

## Open questions
- ...
```

## Rules

- Gherkin keywords (`Given`, `When`, `Then`, `And`, `But`) start a line and are capitalised.
- Scenarios use `## Scenario: <name>` so renderers and grep both find them.
- One sentence per step. Nested behaviour goes into its own scenario.
- No implementation detail in steps (no "click button X at selector Y") — push back when the user supplies one and ask for the behaviour it represents.
- Empty `Boundaries` bullets are written as `- none declared`, not omitted.
- `created` is written once. `updated` is refreshed on every successful write.
- Omit the `allium:` key entirely when there are no references; `allium: []` and a missing key are equivalent on read.
- Status: `draft` while in progress, `complete` on successful Wrap-up / Confirm, `aborted` on `/abort`.
- `## Open questions` is omitted entirely when there are none. Resolved questions are deleted (folded into scenarios, boundaries, or invariants), not struck-through.
