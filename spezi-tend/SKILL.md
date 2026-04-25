---
name: spezi-tend
description: Tend the Gherkin garden. Use when the user wants to write, edit, update, add to, improve, clarify, refine, restructure, fix, or migrate an existing `.feature.md` spec. Targeted edits only — for new specs use `/spezi-elicit`; for extracting from code use `/spezi-distill`. Apply only low-risk tightenings silently; flag anything meaning-changing. Up to three Draft ↔ Flag ↔ Redraft cycles. After Confirm, offer to invoke `/spezi-propagate`.
---

# spezi-tend

Targeted edits to an existing `.feature.md`. Seed identifies the file:

- A slug (`sso-login`).
- A path (`specs/gherkin/sso-login.feature.md`).
- Free-form text (`the SSO login spec`).

Resolution:

- One match → proceed.
- Empty / ambiguous → list candidates (up to 10, newest first) and ask. Never tend a file the user did not clearly name.
- Zero matches → report and exit.

## Boundaries

- Edits `.feature.md` only.
- Does **not** extract behaviour from code (use `/spezi-distill`).
- Does **not** run discovery sessions for unclear requirements (use `/spezi-elicit`).
- Does **not** check spec-vs-code alignment (use `/spezi-weed`).

If the request goes beyond targeted edits, name the right skill and exit.

## Preconditions

- Target `.feature.md` exists and parses (front matter optional; scenarios, boundaries, steps must be present).
- `specs/gherkin/decisions/` is writable.
- Existing `allium:` entries are read into memory as context. Never edit `.allium` files.

## Output

- Updated `specs/gherkin/<slug>.feature.md` — match `spezi/reference/feature-file.template.md`.
- Decision record: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>-tend[-<n>].md` — base + Tend section per `spezi/reference/decision-record.template.md`.

The `-tend` suffix is required. If a same-day record collides, append `-1`, `-2`, ... until unique. Never overwrite.

## Escape hatches

- `/done` — current flag list is sufficient. Unanswered flags = `reject`.
- `/skip <n>` defers one flag; bare `/skip` is a synonym for `/done`.
- `/back` from Read is a no-op synonym for `/abort` — say so. From later steps it returns to Read.
- `/abort` ends the session. Persist nothing to the feature file; write the record with `Outcome: aborted`.

## Full-section rewrite

The whole `.feature.md` is composed and re-presented per cycle. Section-level edits are replacements, never patches. If the user critiques a draft freeform, redraft the whole document — never hand-merge a diff.

## Loop — Read → Draft → Flag → Resolve → Redraft → Confirm

1. **Read.** Load the feature file and the most recent decision record for its slug. Open with one line: `Tending <slug>, last updated <date>, <N> scenarios. Ready? (yes / /abort)`.
2. **Draft.** Compose a revised full `.feature.md` applying only low-risk tightenings (table below). Present the whole proposed document, preceded by a short *What I touched* summary of categories — not a line diff.
3. **Flag.** Present meaning-changing observations the skill did **not** fold into the draft, using the pattern:

   ```
   Flagged for your attention (Tend — <slug>):
   1. [scenario: <name>] <observation>
   2. [boundaries] <observation>

   Reply with `1: accept`, `1: reject`, `1: defer`, or `1: <amended text>`.
   Multiple on one line OK. Silence = reject all.
   ```

4. **Resolve.** `accept` / `<amended>` → fold into the next draft, log under *Accepted suggestions*. `reject` → leave draft as-is, log under *Rejected suggestions*. `defer` → log under *Deferred / open questions* (append, never replace prior entries).
5. **Redraft.** If any flag resolved to `accept` or `<amended>`, compose a second full proposal and repeat from Flag for any newly-introduced observations. Otherwise skip to Confirm.
6. **Confirm.** User types `confirm` / `looks good` / `yes`, or critiques freeform (→ redraft). On confirmation: write the feature file (`status: complete`, `updated` refreshed), write the record (`Outcome: completed`, `Cycles: <n>`).

Cap: **three** cycles of Draft ↔ Flag ↔ Redraft. A fourth unresolved cycle halts: `Commit as-is or /abort?`.

## Draft guardrails

| Apply silently (low-risk) | Never apply without a flag (meaning-changing) |
|---------------------------|-----------------------------------------------|
| Gherkin keyword casing (`given` → `Given`) | Adding, removing, merging, splitting scenarios |
| Mechanical splits of run-on steps (conjunction-driven, no semantic inference) | Renaming actors, triggers, scenarios |
| Consistent `Scenario:` heading prefix | Changing step wording beyond keyword casing |
| Empty boundary lists → `- none declared` | Altering or adding invariants |
| Front-matter key ordering; `updated` timestamp refresh | Changing *In scope* / *Out of scope* / *Depends on* membership |
| Trailing whitespace; blank-line normalisation | Modifying `allium:` references (add, remove, change status) |
| Typo fixes in Gherkin connective words only (`Whn` → `When`) | Rewriting prose summary in a way that could change scope |

Typos in user prose (summary, boundaries, step content) are **not** silently corrected — propose them as flags.

## Stale Allium handling

For each `allium:` entry whose `linkedAt` is older than the feature `updated` timestamp, surface it as a meaning-changing flag in the same flag list. Accept may change `status` (e.g. to `stale`) or remove the entry; reject leaves it as-is. Never unilaterally edit. See `spezi/reference/allium-linking.md` for schema.

## After Confirm — Propagate hook

If `status: complete` and at least one `## Scenario:`, ask:

```
Tend complete. Generate or refresh tests for this spec via /spezi-propagate?

Reply `yes` / `not now` / `never this session`.
```

On `yes`, hand off with `{ seed: <slug> }`. Log outcome in *Session trail*. Skip the hook if the user passed `--no-propagate` or invoked under `/spezi --no-propagate`.

## Library-spec candidates

If a flagged change would belong in a reusable integration spec rather than this feature (OAuth flows, payment, email delivery, calendar sync, ATS sync, file storage), surface it as a separate flag: `[library candidate] extract <pattern> to specs/gherkin/lib/<name>.feature.md?` with buckets `accept` / `reject` / `defer`. Do not extract silently.
