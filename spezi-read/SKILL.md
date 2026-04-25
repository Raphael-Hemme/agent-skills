---
name: spezi-read
description: Read-only spec reference. Present a `.feature.md` and answer focused questions by quoting verbatim — never paraphrase normative text. Writes nothing unless the user invokes `/update-spec` to propose a scoped section change. Use when the user wants to orient on an existing spec, review one before TDD, or types `/spezi --read <slug>`. A pure read writes no decision record.
---

# spezi-read

Read-only spec reference. Seed identifies one feature file (slug, path, or free-form). Resolution:

- One match → proceed.
- Empty / ambiguous → list candidates (up to 10, newest first) and ask. Never read a file the user did not name.
- Zero matches → report and exit.

## Read-only contract

During this session, `.feature.md`, decision records, `.allium` files, test files, and implementation source are **read-only**. The only path to a write is `/update-spec` (below). State this in the preamble, before the first turn.

If you detect you are about to write any read-only artifact, halt and surface the attempt verbatim. Phrase any proposed edits as observations, never as "I'll change X" prose.

## Flow

1. **Resolve.** Seed → one feature file.
2. **Preamble.** State the contract:
   > `--read`: *<slug>.feature.md* and its decision records are read-only. No files will be written. Use `/update-spec` to propose a change; `/abort` to exit.
3. **Synopsis.** One block: title, last-updated date, scenario count (happy vs edge), boundaries summary, `allium:` references with their `status`.
4. **Follow-up queries.** Answer focused questions by quoting the file verbatim or summarising — never by paraphrasing normative text.
5. **Exit.** On user signal or `/abort`. `/update-spec` may be invoked at any prompt; after the update, the session returns to read-mode with the refreshed file.

A pure read writes no decision record. The session is not a decision.

## `/update-spec` — the only escape from read-only

Triggered by the user typing `/update-spec`, or offered by this skill when a question can only be resolved by a spec change.

1. **Confirm exit.** Ask:
   > Exiting read-only for *<feature file>*. Scope of proposed change: <one-line summary>. Proceed? (yes / cancel)

   `cancel` returns to read-mode without writing.

2. **Scoped draft.** Compose the proposed rewrite **at the section level** — not the whole file. Replace the affected section(s) in full. Show the proposed section(s) for confirmation.

3. **Flag confirmation.** For any meaning-changing aspect, present:

   ```
   Flagged for your attention (/update-spec — <slug>):
   1. [<section>] <observation>

   Reply with `1: accept`, `1: reject`, `1: defer`, or `1: <amended>`.
   Silence = reject all.
   ```

4. **Resolve + redraft.** Same buckets as distill. **One redraft cycle maximum** under `/update-spec`. If the user is not satisfied after one revision, suggest exiting and running `/spezi-distill`.

5. **Persist.**
   - Write the updated `.feature.md` (replace affected section(s) in full; refresh `updated`).
   - Write a decision record at `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>-read.md` with base shape + `## Spec updates` section per `spezi/reference/decision-record.template.md`.
   - If a recent distill or elicit record exists, add a `Cross-ref:` line in *Spec updates*. The cross-reference is one-way — do not edit the prior record.

6. **Return to read-mode** with the refreshed spec; re-present the synopsis.

## Guardrails

- Writes only `.feature.md` and the `--read` session's own decision record. Never `.allium` files, test files, or implementation source.
- No cascading updates. A spec change that invalidates tests is recorded; downstream consequences are addressed in `/spezi-red` (or `/spezi-green`).
- `/update-spec` is not a general-purpose edit command — it writes only what the scoped draft proposes. For broader rework, exit and run `/spezi-distill`.
