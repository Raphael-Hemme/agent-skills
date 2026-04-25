---
name: spezi-green
description: TDD green phase. Run failing tests, then for each failure emit a guidance block — quote the governing scenario verbatim, summarise the observed failure, name the likely gap. Never propose a code patch. Spec, decision records, and tests are read-only; implementation source is the user's to write. Use when the user wants to drive failing tests to green, or types `/spezi --green <slug>`.
---

# spezi-green

Guide the user (or another agent) toward making failing tests pass. Spec is the source of truth for "what the test expects to mean." Allium runs the tests and reports failures; this skill annotates each with the governing scenario.

## Read-only contract

- Read-only this session: `.feature.md`, prior decision records, all test files.
- Implementation source is **not** directly written by this session. Spezi guides; does not edit. A coding agent or the user writes code.
- A fresh TDD decision record is the only thing this skill writes.
- The only path to a `.feature.md` write is `/update-spec` (below).

If you detect a write to the read-only set, halt and surface the attempt. Phrase observations, never patches.

## Flow

1. **Resolve.** Seed → one feature file. A focused subset narrows to a scenario name: `seed = "<slug> happy path"` runs only the happy-path tests.
2. **Preamble.** State the contract:
   > `--green`: *<slug>.feature.md*, prior decision records, and all test files are read-only. Implementation source is not directly written by this session; I will guide, not edit. Use `/update-spec` to propose a spec change; `/abort` to exit.
3. **Run.** Allium executes the targeted tests and reports a pass/fail roster. For each failing test, annotate with the governing scenario's Given/When/Then quoted verbatim from the feature file.
4. **Guidance.** For each failing test, emit a *guidance block* — never a patch. The block quotes the spec verbatim, summarises the observed failure, names a likely gap. It does not propose code. If a coding-agent handoff is available in the harness, offer it; do not invoke silently.

   ```
   [scenario: Happy path — SSO login]
   Test:     specs/allium/sso-login/happy-path.<ext>
   Expects:  Given a user with an Okta IdP assertion,
             When the assertion is POSTed to /sso/callback,
             Then a session cookie is issued scoped to the user.
   Observed: assertion accepted; session cookie absent from response.
   Likely gap: cookie issuance path in the callback handler.
   ```

5. **Loop.** User reports `done` (or re-runs explicitly) → step 3 repeats with the current roster. Continue until all targeted tests pass or the user exits.
6. **Spec divergence.** If guidance reveals the spec is wrong (e.g. a Then clause is physically impossible), surface the observation as a flag pointing at `/update-spec` — never compensate silently:

   ```
   Flagged for your attention (Green — <slug>):
   1. [scenario: <name>] <observation suggesting spec is wrong>

   Reply with `1: update-spec` or `1: skip`. Silence = skip all.
   ```

7. **Persist.** Write the TDD record at `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>-green.md`: base shape + `## Pass/fail roster` per `spezi/reference/decision-record.template.md`.

## Escape hatches

- `/done` — end the loop now. Persist the record with the current roster.
- `/abort` ends the session. Persist the record with `Outcome: aborted`.

## `/update-spec`

Triggered by the user, or via the `update-spec` flag bucket above.

1. **Confirm exit.** Ask:
   > Exiting read-only for *<feature file>*. Scope of proposed change: <one-line summary>. Proceed? (yes / cancel)

2. **Scoped draft.** Compose the rewrite at the **section level**. Replace affected section(s) in full.

3. **Flag confirmation.** Buckets `accept` / `reject` / `defer` / `<amended>`; silence = reject all.

4. **Resolve + redraft.** **One redraft cycle maximum.** If the user is not satisfied, suggest exiting and running `/spezi-distill`.

5. **Persist.**
   - Write the updated `.feature.md` (refresh `updated`).
   - Append to the current TDD record's `## Spec updates` section, with a `Cross-ref:` line to any related earlier record. One-way reference.

6. **Return to green-mode.** Re-derive failing-test guidance for any scenario whose text changed.

## What this skill does not do

- Write implementation code. A coding-agent handoff is offered if available; not silently invoked.
- Edit tests. Tests stay read-only. A wrong test is a refactor-phase concern, deferred.
- Silently modify the feature file. Spec issues surface as flags pointing at `/update-spec`.
