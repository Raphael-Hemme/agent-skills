---
name: tdd
description: TDD-phase protocol for Behavioral Spec Driven Development. Specifies `--read`, `--red`, and `--green` modes with a shared read-only enforcement contract and a single spec-update escape hatch. Invoked indirectly — the router targets Gherkin and/or Allium with these modes; this document is the cross-sub-skill agreement they both honor.
---

# TDD Phase Protocol

The TDD modes are **not** a separate sub-skill target. The router
continues to dispatch to Gherkin (spec side) and Allium (execution
side) per `spezi/core/router.md`. This document is the contract those
sub-skills follow when invoked with a TDD-phase mode, plus the
user-facing protocol that spans them.

## Modes

| Mode | Status | Target resolution (default) | Writes any `.feature.md`? |
|------|--------|-----------------------------|---------------------------|
| `read` | **Specified below** | `{gherkin}` | No — read-only. `/update-spec` is the sole exception. |
| `red` | **Specified below** | `{gherkin, allium}` | No — read-only. `/update-spec` is the sole exception. |
| `green` | **Specified below** | `{gherkin, allium}` | No — read-only. `/update-spec` is the sole exception. |
| `refactor` | Deferred | `{gherkin, allium}` | — |

A conforming implementation must refuse a deferred mode with "not yet
implemented" rather than fall back to a nearest neighbor.

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

The enforcement contract is behavioral, not sandboxed. An agent
running this protocol must:

1. **Announce the contract at session open.** Before the first turn,
   surface a one-paragraph preamble naming the read-only paths and
   the `/update-spec` escape. The user should never be surprised by
   what this mode will and will not touch.
2. **Refuse writes to the read-only set.** If the session flow would
   otherwise write, edit, or delete a read-only file, the sub-skill
   must stop, describe the intended change, and ask the user to
   either confirm via `/update-spec` or pick a different approach.
   A silent edit is a contract violation.
3. **Never propose a read-only edit as a fait accompli.** Proposals
   that would mutate read-only files must be phrased as
   *observations* routed through the grouped-flag pattern — never as
   "I'll change X" prose.
4. **Honor the contract across sub-skill boundaries.** When the
   router invokes both Gherkin and Allium for a TDD mode, each
   sub-skill enforces on its own side. Gherkin must not instruct
   Allium to write a read-only file, and vice versa. The user's
   consent via `/update-spec` is what either side waits for.

### Session-scoped exceptions

The only exceptions are:

- `/update-spec` — exits read-only for a single, scoped write to the
  feature file (and its decision record). Returns to read-only
  immediately afterwards. See *Spec update step*.
- The TDD decision record itself — created fresh per session; not a
  read-only artifact because it did not exist before this session.
  Prior decision records (for other sessions) remain read-only.
- Newly created test files under `--red` — the whole point of the
  mode; not "read-only" because they did not exist before this
  session.

No other writes are legitimate. A conforming sub-skill that detects
it is about to violate the contract must halt and surface the
attempt to the user verbatim.

## `--read` mode

Purpose: present spec material as a read-only reference. Useful for
orientation at the start of a TDD cycle, for review, or for copying
canonical text into a conversation.

### Target

Router default: `{gherkin}` only. Allium is not involved unless the
user explicitly passes `--allium` (in which case Allium runs its own
read mode; that is Allium's concern).

### Flow

1. **Resolve.** Seed → one feature file, using the same resolution
   rules as distill (slug / path / free-form; multiple matches →
   short list + ask; zero matches → report and exit).
2. **Preamble.** Announce the read-only contract:
   > `--read`: *<slug>.feature.md* and its decision records are
   > read-only. No files will be written. Use `/update-spec` to
   > propose a change; `/abort` to exit.
3. **Synopsis.** In one block, report: title, last-updated date,
   scenario count (happy vs edge), boundaries summary, any `allium`
   references with their `status`.
4. **Follow-up queries.** The user may ask focused questions — "show
   the auth edge cases," "what's the invariant list," "what's linked
   from Allium." The sub-skill answers by quoting the file verbatim
   or summarizing, never by paraphrasing normative text.
5. **Exit.** The session ends when the user says so, or on `/abort`.
   `/update-spec` may be invoked at any time and flips to the spec
   update step; after the update, the session returns to read mode
   with the refreshed file.

### Writes

None, unless `/update-spec` is invoked. No decision record is
written for a pure `--read` session — there is nothing to decide.
The spec update step, when invoked, writes its own record.

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

4. **Grouped mismatch flags.** Mismatches are presented as a grouped
   flag list, same visual shape as elsewhere:

   ```
   Alignment gaps:
   1. [scenario: Edge case — expired] No test found.
   2. [scenario: Edge case — rate limited] Test exists but spec Then
      is stronger than test assertion.
   3. [scenario: Invariant — audit log] Test exists but is skipped.
   4. [manifest] specs/allium/sso-login.allium references a deleted
      test file.

   Reply with `1: generate`, `1: update-spec`, `1: skip`, or
   `1: handoff` for each. Silence = skip all.
   ```

5. **Resolve.** Map each reply:
   - `generate` → Allium generates a test for that scenario. Gherkin
     supplies the scenario's text as the authoritative source. The
     test is written to a new file under `specs/allium/<slug>/` (or
     wherever Allium's layout dictates). Gherkin verifies that the
     new test references the scenario.
   - `update-spec` → invoke the spec update step, then re-evaluate
     this mismatch against the new spec before regenerating or
     skipping.
   - `skip` → log under *Deferred* in the TDD decision record.
   - `handoff` → stop generating; hand the gap to the user (e.g. a
     human will write the test). Log as deferred with reason
     `user-handoff`.

6. **Persist.** Write the TDD decision record. Any new test files
   land per Allium's rules (with the Allium-side back-reference to
   the feature file — see `spezi/gherkin/SKILL.md` *Linking to
   Allium artifacts*). No `.feature.md` write occurs outside
   `/update-spec`.

7. **Confirm.** Final prompt: "Generated <N> tests, deferred <M>,
   handed off <K>. Confirm decision record?" On confirmation, the
   record is finalized and the session ends.

### Scope guardrails

- `--red` does **not** run tests. Running them is `--green`'s job.
  A brand-new failing test is presumed failing; verification belongs
  to the next phase.
- `--red` does **not** edit existing tests. A mismatch on an
  existing test is `update-spec` (if the spec is wrong) or deferred
  (if the test is wrong — that belongs to refactor, when specified).

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

1. **Resolve.** Seed → one feature file (or a focused subset:
   `seed = "<slug> happy path"` narrows to a scenario name).
2. **Preamble.** Announce the contract:
   > `--green`: *<slug>.feature.md*, prior decision records, and
   > all test files are read-only. Implementation source is not
   > directly written by this session; I will guide, not edit. Use
   > `/update-spec` to propose a spec change; `/abort` to exit.
3. **Run.** Allium executes the targeted tests and reports a
   pass/fail roster. Gherkin annotates each failing test with the
   governing scenario's Given/When/Then, quoted from the feature
   file.
4. **Guidance.** For each failing test, the sub-skill produces a
   *guidance block*, never a patch:

   ```
   [scenario: Happy path — SSO login]
   Test:     specs/allium/sso-login/happy-path.<ext>
   Expects:  Given a user with an Okta IdP assertion,
             When the assertion is POSTed to /sso/callback,
             Then a session cookie is issued scoped to the user.
   Observed: assertion accepted; session cookie absent from response.
   Likely gap: cookie issuance path in the callback handler.
   Files to inspect: <if Allium can introspect; otherwise omit>.
   ```

   The guidance block quotes the spec verbatim and summarizes the
   failure. It does not propose code. If the agent ecosystem
   supports handoff to a coding agent, the sub-skill offers that
   handoff — it does not silently invoke it.

5. **Loop.** The user reports "done" (or re-runs explicitly) → step
   3 repeats with the current pass/fail roster. Continue until all
   targeted tests pass or the user exits.

6. **Spec divergence.** If guidance reveals that the spec is wrong
   (e.g. Then clause is actually impossible), the sub-skill
   surfaces this as a flag rather than quietly compensating:
   > [scenario: …] The Then clause appears to contradict
   > <observed behavior>. Invoke `/update-spec` to revise, or
   > `skip` to proceed under the test as written.

7. **Persist.** A TDD decision record is appended at session end,
   summarizing which tests passed, which were deferred, and whether
   `/update-spec` was used.

### What `--green` does not do

- It does not write implementation code. Coding agents may be
  invoked by the adapter or the user; Spezi's protocol stops at
  guidance.
- It does not edit tests. Tests are read-only in `--green`. A test
  that is wrong is a refactor-phase concern (deferred).
- It does not silently modify the feature file. Spec issues surface
  as flags pointing at `/update-spec`.

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

2. **Scoped draft.** The sub-skill composes a proposed diff *at the
   section level* — not the whole file, unlike distill. Only the
   section(s) affected by the update are drafted. The output uses
   the same full-section-rewrite rule from
   `spezi/gherkin/SKILL.md` *Full-section rewrite rule*: the
   section is rewritten in full, not patched in place.

3. **Grouped flag confirmation.** Any meaning-changing aspect of
   the proposed rewrite is flagged, exactly as in distill:

   ```
   Flagged:
   1. [scenario: …] Proposed Then clause adds a new assertion not
      previously in the spec — accept?
   2. [invariants] Removing the "audit log immutable" invariant —
      is this intentional or a drafting slip?

   Reply with `1: accept`, `1: reject`, `1: defer`, or
   `1: <amended>` for each. Silence = reject all (leave as-is).
   ```

4. **Resolve + redraft.** Same accept / reject / defer / amended
   semantics as distill. One redraft cycle maximum under a spec
   update step — if the user is not satisfied after one revision,
   the sub-skill suggests ending the TDD session and running
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

- The spec update step writes **only** to `.feature.md` and the
  current session's decision record. It does not touch Allium
  files, test files, or implementation source. Those are downstream
  consequences of a spec change and are addressed in their own
  phases (`--red` for missing/stale tests, `--green` for failing
  implementation).
- No cascading updates. A spec change that invalidates existing
  tests is recorded as a new alignment mismatch the next time
  `--red` runs; it is not fixed inside the spec update step.
- `/update-spec` is *not* a general-purpose edit command. It writes
  what the scoped draft proposes, and nothing more. A user who
  wants to rework the spec broadly should exit and run `--distill`.

## TDD decision record

Path: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>-<mode>.md`
(`<mode>` ∈ `read` | `red` | `green`). If the same file already
exists, append `-<n>`.

Shape (extends the elicit/distill record shape):

```
# <Feature title> — <mode> session, <YYYY-MM-DD>

- **Slug**: <slug>
- **Session type**: tdd-<mode>
- **Feature file**: specs/gherkin/<slug>.feature.md
- **Outcome**: completed | aborted
- **Started**: <ISO 8601 timestamp>
- **Ended**: <ISO 8601 timestamp>

## Alignment outcomes   # --red only
- [scenario: …] generated → specs/allium/<slug>/<test>
- [scenario: …] deferred (reason: …)
- [scenario: …] handed off

## Pass/fail roster     # --green only
- [scenario: …] pass
- [scenario: …] fail → guidance delivered; user reports in progress

## Spec updates         # any mode, if /update-spec invoked
- [scenario: …] <summary of accepted change>
  Cross-ref: <path to related decision record, if any>

## Deferred / open questions
- ...

## Session trail
- `/update-spec` invoked at <timestamp>: <scope>
- ...
```

A `--read` session writes a decision record **only** if
`/update-spec` was invoked; otherwise the read is not a decision.
