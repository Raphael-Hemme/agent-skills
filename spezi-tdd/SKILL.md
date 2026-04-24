---
name: spezi-tdd
description: TDD-phase runner for Behavioral Spec Driven Development. Specifies `--read`, `--red`, and `--green` modes with a shared read-only enforcement contract and a single spec-update escape hatch. Invoked by Spezi — not directly. Alongside `spezi-gherkin`; Spezi routes to whichever skill matches the mode.
---

# spezi-tdd

The sibling sub-skill to `spezi-gherkin`. Spezi routes the TDD-phase
flags (`--read`, `--red`, `--green`, and later `--refactor`) here,
passing `{ mode, seed }`. For `--red` and `--green`, Spezi additionally
invokes Allium so tests can be generated and executed; this sub-skill
supplies the spec-side alignment analysis and guidance, Allium does
the execution. Neither side edits the other's artifacts.

This file owns the read-only enforcement contract, the alignment
check, and the `/update-spec` escape hatch. Shared dialogical
patterns (escape hatches, batch Q&A, grouped ambiguity flags,
full-section rewrite, session state) live in
`spezi/core/patterns.md`. Decision-record schema — base plus the
TDD extension — lives in `spezi/core/decision-records.md`.

## Modes

| Mode | Status | Gherkin-side skill | Allium involved? | Writes any `.feature.md`? |
|------|--------|--------------------|------------------|---------------------------|
| `read` | **Specified below** | `spezi-tdd` | No | No — read-only. `/update-spec` is the sole exception. |
| `red` | **Specified below** | `spezi-tdd` | Yes | No — read-only. `/update-spec` is the sole exception. |
| `green` | **Specified below** | `spezi-tdd` | Yes | No — read-only. `/update-spec` is the sole exception. |
| `refactor` | Deferred | `spezi-tdd` | Yes | — |

A conforming implementation must refuse a deferred mode with "not
yet implemented" rather than fall back to a nearest neighbour.

## Read-only enforcement contract

The TDD modes hinge on one invariant: during a TDD session, the
authored-artifact surface is **read-only by default**. The only path
from read-only to write is the *spec update step* (see below). The
contract applies symmetrically to Gherkin and Allium.

### What is read-only, per mode

| Mode | Read-only set | Writable set (by this session) |
|------|--------------|-------------------------------|
| `read` | `.feature.md`, decision records, `.allium` files, test files, implementation source | — (nothing; pure read) |
| `red` | `.feature.md`, prior decision records, existing test files, implementation source | Newly created test files (Allium side); a new TDD decision record (Gherkin side). |
| `green` | `.feature.md`, decision records, `.allium` files, test files | Implementation source is writable *outside* this protocol; Spezi does not direct writes to it. A new TDD decision record is appended. |

### Agent-side enforcement

The contract is behavioral, not sandboxed. A conforming agent must:

| Requirement | Behaviour |
|-------------|-----------|
| Announce at session open | One-paragraph preamble names the read-only paths and the `/update-spec` escape, before the first turn. |
| Refuse silent writes to the read-only set | Stop, describe the intended change, ask the user to either invoke `/update-spec` or pick a different approach. |
| Never propose a read-only edit as a fait accompli | Phrase proposals as observations routed through the grouped-flag pattern (see `spezi/core/patterns.md`), not as "I'll change X" prose. |
| Honor the contract across sub-skill boundaries | Gherkin must not instruct Allium to write a read-only file, and vice versa. Each side enforces on its own; `/update-spec` is what both wait for. |

### Session-scoped exceptions

Only three writes escape read-only:

- `/update-spec` — a single scoped write to the feature file and its
  decision record, returning to read-only afterwards. See *Spec
  update step*.
- The TDD decision record itself — created fresh per session, so it
  did not exist before the session and is not read-only. Prior
  decision records remain read-only.
- Newly created test files under `--red` — the whole point of the
  mode.

A conforming sub-skill that detects it is about to violate the
contract must halt and surface the attempt to the user verbatim.

## `--read` mode

Purpose: present spec material as a read-only reference. Useful for
orientation at the start of a TDD cycle, for review, or for copying
canonical text into a conversation.

### Target

Router default: `{gherkin}` only. Allium is not involved unless the
user explicitly passes `--allium` (in which case Allium runs its own
read mode; that is Allium's concern).

### Flow

1. **Resolve.** Seed → one feature file (same resolution rules as
   distill: slug / path / free-form; multiple matches → short list +
   ask; zero matches → report and exit).
2. **Preamble.** Announce the read-only contract:
   > `--read`: *<slug>.feature.md* and its decision records are
   > read-only. No files will be written. Use `/update-spec` to
   > propose a change; `/abort` to exit.
3. **Synopsis.** Report in one block: title, last-updated date,
   scenario count (happy vs edge), boundaries summary, `allium`
   references with their `status`.
4. **Follow-up queries.** Answer focused questions by quoting the
   file verbatim or summarizing — never by paraphrasing normative
   text.
5. **Exit.** Session ends on user signal or `/abort`. `/update-spec`
   may be invoked at any time; after the update, the session
   returns to read mode with the refreshed file.

No writes unless `/update-spec` is invoked. A pure `--read` session
writes no decision record — there is nothing to decide.

## `--red` mode

Purpose: write a failing test that captures a behavior from the
spec. The spec and existing tests are read-only; the only legitimate
writes are new test files (Allium side).

### Target

Router default: `{gherkin, allium}`. Gherkin supplies the spec and
runs the alignment analysis; Allium generates the tests. Neither
side edits `.feature.md`.

### Flow

1. **Resolve.** Seed → one feature file (as in distill).
2. **Preamble.** Announce the contract:
   > `--red`: *<slug>.feature.md*, prior tests, and prior decision
   > records are read-only. New test files under
   > `specs/allium/<slug>/` may be created. Use `/update-spec` to
   > change the spec; `/abort` to exit.
3. **Alignment check.** Gherkin reads the feature file; Allium
   reports which scenarios are covered by existing tests (by
   whatever mechanism Allium uses — a manifest, naming convention,
   or its own index). Gherkin compiles a per-scenario alignment
   table:

   | Scenario | Test present? | Up to date? | Asserting? |
   |----------|---------------|-------------|------------|
   | Happy path — …           | yes | yes | yes |
   | Edge case — expired      | no  | —   | — |
   | Edge case — rate limited | yes | stale: spec updated 3d after test | yes |
   | Invariant — audit log    | yes | yes | no (skipped) |

   "Up to date" means the scenario's Given/When/Then text is
   congruent with the test's expressed expectations, not a byte
   comparison.

4. **Grouped mismatch flags.** Mismatches present via the
   grouped-flag pattern (`spezi/core/patterns.md`). Buckets:
   `generate` / `update-spec` / `skip` / `handoff`; silence = skip
   all.

5. **Resolve.** `generate` → Allium generates a test using the
   scenario's text as the authoritative source; the new test is
   written under `specs/allium/<slug>/` (or Allium's layout).
   Gherkin verifies the new test references the scenario.
   `update-spec` → invoke the spec update step, then re-evaluate
   this mismatch against the new spec before regenerating.
   `skip` → log under *Deferred* in the TDD decision record.
   `handoff` → hand the gap to the user; log as deferred with reason
   `user-handoff`.

6. **Persist.** Write the TDD decision record. New test files land
   per Allium's rules (with the Allium-side back-reference per
   `spezi/core/allium.md`). No `.feature.md` write occurs outside
   `/update-spec`.

7. **Confirm.** Final prompt: "Generated <N> tests, deferred <M>,
   handed off <K>. Confirm decision record?" On confirmation, the
   record is finalized and the session ends.

### Scope guardrails

- `--red` does **not** run tests. That's `--green`'s job.
- `--red` does **not** edit existing tests. A mismatch on an
  existing test is `update-spec` (if the spec is wrong) or deferred
  (if the test is wrong — that's a refactor-phase concern).

## `--green` mode

Purpose: guide the user (or another agent) toward making the
failing tests pass. Spec and tests are read-only; implementation
source is writable outside this protocol — Spezi does not direct
those writes.

### Target

Router default: `{gherkin, allium}`. Gherkin supplies the spec as
the source of truth for "what the test expects to mean." Allium
runs the tests and reports failures.

### Flow

1. **Resolve.** Seed → one feature file. A focused subset narrows to
   a scenario name: `seed = "<slug> happy path"`.
2. **Preamble.** Announce the contract:
   > `--green`: *<slug>.feature.md*, prior decision records, and
   > all test files are read-only. Implementation source is not
   > directly written by this session; I will guide, not edit. Use
   > `/update-spec` to propose a spec change; `/abort` to exit.
3. **Run.** Allium executes the targeted tests and reports a
   pass/fail roster. Gherkin annotates each failing test with the
   governing scenario's Given/When/Then, quoted from the feature
   file.
4. **Guidance.** For each failing test, emit a *guidance block* —
   never a patch. The block quotes the spec verbatim, summarizes
   the observed failure, and names a likely gap. It does not
   propose code. If a coding-agent handoff is available, offer it;
   do not invoke silently.

   ```
   [scenario: Happy path — SSO login]
   Test:     specs/allium/sso-login/happy-path.<ext>
   Expects:  Given a user with an Okta IdP assertion,
             When the assertion is POSTed to /sso/callback,
             Then a session cookie is issued scoped to the user.
   Observed: assertion accepted; session cookie absent from response.
   Likely gap: cookie issuance path in the callback handler.
   ```

5. **Loop.** User reports "done" (or re-runs explicitly) → step 3
   repeats with the current roster. Continue until all targeted
   tests pass or the user exits.

6. **Spec divergence.** If guidance reveals the spec is wrong (e.g.
   a Then clause is physically impossible), surface the observation
   via the grouped-flag pattern pointing at `/update-spec` — never
   compensate silently.

7. **Persist.** Write a TDD decision record at session end:
   pass/fail roster, deferred items, any `/update-spec` invocation.

### What `--green` does not do

- Write implementation code (coding agents may be invoked by the
  adapter or the user — Spezi stops at guidance).
- Edit tests (tests stay read-only; wrong tests are a refactor-phase
  concern, deferred).
- Silently modify the feature file (spec issues surface as flags
  pointing at `/update-spec`).

## Spec update step

The sole escape hatch from read-only to a `.feature.md` write.
Available from `--read`, `--red`, and `--green`. Invoked by the user
(`/update-spec`) or offered by the sub-skill when a mismatch can
only be resolved by a spec change.

### Trigger

- User types `/update-spec` at any prompt.
- Sub-skill presents a flag resolution bucket labeled `update-spec`
  and the user selects it.

In either case, the next turn is the confirmation step below.

### Flow

1. **Confirm exit.** The sub-skill asks:
   > Exiting read-only for *<feature file>*. Scope of proposed
   > change: <one-line summary> (or "user-directed"). Proceed?
   > (`yes` / `cancel`)

   `cancel` returns to the current TDD mode without writing.

2. **Scoped draft.** Compose the proposed rewrite *at the section
   level* — not the whole file, unlike distill. Only the section(s)
   affected by the update are drafted, using the full-section
   rewrite rule from `spezi/core/patterns.md` (replace, never
   patch).

3. **Grouped flag confirmation.** Any meaning-changing aspect of
   the proposal is flagged per `spezi/core/patterns.md`
   §Ambiguity-flag presentation; buckets are `accept` / `reject` /
   `defer` / `<amended>`, silence = reject all.

4. **Resolve + redraft.** Same accept / reject / defer / amended
   semantics as distill. **One redraft cycle maximum** under a
   spec update step — if the user is not satisfied after one
   revision, suggest exiting the TDD session and running
   `--distill` for deeper rework.

5. **Persist.**
   - Write the updated `.feature.md` (replacing the affected
     section(s) in full; `updated` timestamp refreshed).
   - Append an entry to the current TDD session's decision record
     under *Spec updates*, referencing the specific scenario(s)
     affected and the accepted flags.
   - If a distill decision record for this slug exists from an
     earlier session, *cross-reference* — do not edit it. The
     cross-reference is a path + date in the TDD record:
     > Cross-ref: specs/gherkin/decisions/2026-03-10-sso-login.md

6. **Return to read-only.** The TDD mode resumes with the refreshed
   spec. In `--red`, the alignment analysis is re-run for any
   scenarios whose text changed. In `--green`, failing-test guidance
   is re-derived. In `--read`, the synopsis is re-presented.

### Guardrails

- Writes **only** to `.feature.md` and the current session's
  decision record. Allium files, test files, and implementation
  source are untouched; downstream consequences are addressed in
  their own phases (`--red` for missing/stale tests, `--green` for
  failing implementation).
- No cascading updates. A spec change that invalidates existing
  tests is recorded as a new alignment mismatch the next time
  `--red` runs.
- `/update-spec` is not a general-purpose edit command — it writes
  only what the scoped draft proposes. For broader rework, exit and
  run `--distill`.

## TDD decision record

Schema, path convention, and the `--red` / `--green` / `Spec updates`
sections live in `spezi/core/decision-records.md` §TDD extension.

A `--read` session writes a record **only** if `/update-spec` was
invoked; otherwise the read is not a decision and no record is
written.
