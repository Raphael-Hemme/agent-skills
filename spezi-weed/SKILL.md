---
name: spezi-weed
description: Weed the Gherkin garden. Find where a `.feature.md` and the implementation code have diverged, then help resolve the divergences. Three modes — `check` (default; report only), `update-spec` (align spec to code), `update-code` (align code to spec). Use when the user wants to verify their spec still matches reality, find drift, audit alignment, or fix divergences. Does not extract from code (use `/spezi-distill`); does not author new specs (use `/spezi-elicit`).
---

# spezi-weed

Compare a `.feature.md` to the implementation. Surface divergences both ways: spec says X but code does Y, and code does Z but the spec is silent. Classify each divergence; let the user confirm; act per the chosen mode.

## Modes

Pick one. Default to `check` if not specified.

| Mode | What it does | Writes |
|---|---|---|
| `check` | Report divergences. No edits. | Nothing. Report stays in chat. May propose appending unresolved items to the spec's `## Open questions`. |
| `update-spec` | Align the `.feature.md` to current code behaviour. | `.feature.md` only. |
| `update-code` | Align code to spec — surface guidance, never patches. | Nothing. Guidance is emitted to chat. |

`check` is safe to run anywhere. `update-spec` writes only the feature file. `update-code` never edits code itself — it produces guidance blocks the user (or another agent) acts on.

## Output

- (`update-spec` only) updated `specs/gherkin/<slug>.feature.md` with refreshed `updated`.

## Boundaries

- Does not build new specs (use `/spezi-elicit`).
- Does not extract specs from code (use `/spezi-distill`).
- Does not modify the language reference, framework conventions, or architectural choices.
- `update-code` never writes code — it produces guidance for the user (or another agent) to act on.

## Step 1 — Resolve

Seed → one feature file (slug, path, or free-form; ambiguous → list candidates and ask; zero → exit). Confirm the mode if the user didn't pass one explicitly:

```
Found <slug>.feature.md, last updated <date>, <N> scenarios.
Mode: check (default) / update-spec / update-code? Use `check` to start.
```

## Step 2 — Map both sides

Read the `.feature.md` and the implementation. For each `## Scenario:`, locate the code path that implements it (or the absence). Build a per-scenario alignment table:

| Scenario | Code present? | Behaviour matches? | Notes |
|---|---|---|---|
| Happy path — SSO login | yes | yes | — |
| Edge case — expired token | yes | drift: 24h in spec, 7d in code | spec or code? |
| Edge case — rate limited | no | n/a | code returns 429 but spec is silent |
| Invariant — audit log | yes | partial | only logs success, not failure |

"Behaviour matches" means the scenario's Given/When/Then is congruent with the code's expressed behaviour, not a byte comparison.

## Step 3 — Classify each divergence

For each row that isn't a clean match:

| Class | Meaning |
|---|---|
| **Spec bug** | Spec is wrong; code is correct. Resolve via `update-spec`. |
| **Code bug** | Code is wrong; spec is correct. Resolve via `update-code` (guidance only). |
| **Aspirational design** | Spec describes intent that wasn't built yet. Defer or `update-code`. |
| **Intentional gap** | Spec deliberately silent on this code (out-of-scope, infrastructure, library). Confirm; no further action. |

Confirm classification with the user before proceeding:

```
Flagged for your attention (Weed — <slug>):
1. [scenario: Edge case — expired token] spec says 24h, code says 7d
   Proposed: spec bug (the 7d is the live behaviour)
2. [code-only] /api/sso/health endpoint returns 200; spec is silent
   Proposed: intentional gap (operations health check)

Reply `1: accept`, `1: reclassify <new class>`, `1: defer`. Silence = accept all.
```

## Step 4 — Act per mode

### `check`

Report the alignment table and classifications in chat. End with: `Re-run with --update-spec or --update-code to resolve.`

If any items are classified `aspirational` or remain ambiguous, ask:

```
Append the following to <slug>.feature.md → `## Open questions`?
- <classification>: <one-line note>
Reply `append` / `discard` / `pick: 1, 3` for a subset.
```

`append` is the only writing path in this mode; default on silence is `discard`.

### `update-spec`

For each `spec bug` (or `aspirational` reclassified to `spec bug`), invoke the scoped spec-update flow:

1. Confirm exit from read-only on `.feature.md` (one-line summary).
2. Compose a section-level rewrite — the affected scenario(s), `## Boundaries`, or `## Invariants`. Replace the section in full; refresh `updated`.
3. Flag any meaning-changing aspect with `accept` / `reject` / `defer` / `<amended>` buckets; silence = reject all. `defer` items go into the spec's `## Open questions` section.
4. **One redraft cycle maximum.** If the user is not satisfied, suggest `/spezi-tend` for deeper rework.
5. Persist the spec.

Never write to code in this mode.

### `update-code`

For each `code bug` (or `aspirational` reclassified that way), emit a guidance block — never a patch:

```
[scenario: Edge case — expired token]
Spec:     Given a token older than 24 hours,
          When it is presented to /sso/callback,
          Then the response is 401 Unauthorized.
Observed: code accepts tokens up to 7 days old (config: TOKEN_TTL_DAYS=7).
Likely fix: change TOKEN_TTL_DAYS to 1, or rephrase the spec to 7 days.
```

Guidance blocks are emitted to chat only. Code changes are the user's responsibility — copy the blocks into commit messages, issues, or hand them to another agent.

## Step 5 — Cross-entity / process-level checks

After per-scenario divergences, scan for higher-order gaps:

- **Unreachable transitions.** A spec scenario presumes a state the code never enters. Flag.
- **Missing surfaces.** A spec assumes a trigger the code lacks (e.g. webhook handler). Flag.
- **Implicit code state machines.** Code has a state machine the spec doesn't describe. Flag as `intentional gap` or `aspirational` per user direction.

## Escape hatches

- `/done` — current flag list is sufficient. Unanswered = `defer`.
- `/abort` — exit immediately. Nothing is persisted.

## Library-spec divergences

If a divergence sits at a library boundary (third-party integration not owned by the user's domain), surface as:

```
[library boundary] OAuth callback handling drift — owned by library spec specs/gherkin/lib/oauth-google.feature.md?

Reply `1: out of scope here`, `1: belongs in library spec`, `1: this spec owns it`.
```

Silence = `out of scope here`. Do not edit library specs from this skill — that's a separate weed run on the library's own `.feature.md`.
