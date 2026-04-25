---
name: spezi-red
description: TDD red phase. Compare a `.feature.md` against existing tests, surface mismatches as grouped flags (`generate` / `update-spec` / `skip` / `handoff`), and let Allium generate failing tests for accepted scenarios. Spec, prior tests, prior records are read-only — only new test files and a TDD decision record are written. Use when the user wants to add failing tests for a spec, or types `/spezi --red <slug>`.
---

# spezi-red

Write failing tests for a feature. Spec is the source of truth. Allium does the generating; this skill runs the alignment analysis and routes mismatches.

## Read-only contract

- Read-only this session: `.feature.md`, prior decision records, existing test files, implementation source.
- Writable this session: newly created test files (Allium's layout, typically `specs/allium/<slug>/`); a fresh TDD decision record.
- The only path to a `.feature.md` write is `/update-spec` (below).

If you detect a write to the read-only set, halt and surface the attempt verbatim. Phrase proposed changes as observations, never as fait accompli.

## Flow

1. **Resolve.** Seed → one feature file (slug, path, or free-form; ambiguous → list candidates and ask; zero → exit).
2. **Preamble.** State the contract:
   > `--red`: *<slug>.feature.md*, prior tests, and prior decision records are read-only. New test files under `specs/allium/<slug>/` may be created. Use `/update-spec` to change the spec; `/abort` to exit.
3. **Alignment check.** Read the feature file. Ask Allium which scenarios are covered by existing tests (per its own manifest / naming convention / index). Compile a per-scenario table:

   | Scenario | Test present? | Up to date? | Asserting? |
   |----------|---------------|-------------|------------|
   | Happy path — …          | yes | yes | yes |
   | Edge case — expired     | no  | —   | — |
   | Edge case — rate limit  | yes | stale: spec updated 3d after test | yes |
   | Invariant — audit log   | yes | yes | no (skipped) |

   "Up to date" means the scenario's Given/When/Then text is congruent with the test's expressed expectations — not a byte comparison.

4. **Grouped mismatch flags.** Present mismatches as:

   ```
   Flagged for your attention (Red — <slug>):
   1. [scenario: <name>] <one-line gap>
   2. [scenario: <name>] stale: spec updated <date>, test from <date>

   Reply with `1: generate`, `1: update-spec`, `1: skip`, or `1: handoff`.
   Multiple on one line OK. Silence = skip all.
   ```

5. **Resolve.**
   - `generate` → ask Allium to generate a test using the scenario's text as authoritative source. New file under `specs/allium/<slug>/` (or Allium's layout). Verify the new test references the scenario.
   - `update-spec` → invoke `/update-spec` (below), then re-evaluate the mismatch against the new spec before regenerating.
   - `skip` → log under *Deferred / open questions* in the TDD record.
   - `handoff` → hand the gap to the user; log as deferred with reason `user-handoff`.

6. **Persist.** Write the TDD decision record at `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>-red.md`: base shape + `## Alignment outcomes` per `spezi/reference/decision-record.template.md`. New test files land per Allium's rules with the back-reference per `spezi/reference/allium-linking.md` §Round-trip back-reference.

7. **Confirm.** Final prompt: `Generated <N> tests, deferred <M>, handed off <K>. Confirm decision record? (yes / revise)`. On confirmation, the record is finalised and the session ends.

## Escape hatches

- `/done` — current flag list is sufficient. Unanswered flags = `skip`.
- `/skip <n>` defers one flag.
- `/abort` ends the session. Persist the record with `Outcome: aborted`, including any tests already generated.

## `/update-spec` — escape from read-only on `.feature.md`

Triggered by the user, or offered when a mismatch can only be resolved by a spec change.

1. **Confirm exit.** Ask:
   > Exiting read-only for *<feature file>*. Scope of proposed change: <one-line summary>. Proceed? (yes / cancel)

   `cancel` returns to red-mode without writing.

2. **Scoped draft.** Compose the rewrite at the **section level** — not the whole file. Replace affected section(s) in full.

3. **Flag confirmation.** Buckets `accept` / `reject` / `defer` / `<amended>`; silence = reject all.

4. **Resolve + redraft.** **One redraft cycle maximum.** If the user is not satisfied after one revision, suggest exiting and running `/spezi-distill`.

5. **Persist.**
   - Write the updated `.feature.md` (refresh `updated`).
   - Append to the current TDD record's `## Spec updates` section, with a `Cross-ref:` line to any related earlier distill / elicit record (one-way; never edit the prior record).

6. **Return to red-mode.** Re-run alignment for any scenario whose text changed.

## Scope guardrails

- This skill does **not** run tests. That is `/spezi-green`.
- This skill does **not** edit existing tests. A mismatch on an existing test is `update-spec` (if the spec is wrong) or deferred (if the test is wrong — refactor-phase concern).
- This skill does **not** install or configure Allium. If Allium is unavailable, surface a clear error and exit; do not fall back to "Gherkin only" — there are no tests to generate without Allium.
