# Shared Dialogical Patterns

This file is the single home for the conversation-shape patterns every
Spezi sub-skill uses. Sub-skills reference — they do not redefine.
Contradicting a pattern specified here is a conformance bug; narrowing
one (e.g. redefining `/back` when there is no prior phase) is allowed
and must be called out in the sub-skill's own section.

The patterns are independent of mode, phase, and harness. They apply
to elicit, distill, `--read`, `--red`, `--green`, and any future
sub-skill that shares Spezi's dialogical contract.

## Escape hatches

The user may type one of these tokens at any prompt. A sub-skill
must recognize them as control, not content.

| Token | Meaning |
|-------|---------|
| `/done` | The current phase or flag list is sufficient. Advance using whatever answers exist; missing answers become open questions in the decision record. On a flag list, unanswered flags are treated as `reject`. |
| `/skip` | Skip *this specific prompt*. On a question batch: `/skip <letter>` skips one question; bare `/skip` skips the whole phase and logs a gap. On a flag list: `/skip <n>` defers one item; bare `/skip` is a synonym for `/done`. |
| `/back` | Return to the previous phase. The current phase's in-progress answers are discarded; already-confirmed sections remain. The prior section is re-opened for revision. At a phase that has no prior phase, `/back` is a no-op synonym for `/abort` and the sub-skill says so. |
| `/abort` | End the session now. Persist any confirmed sections as a draft with a prominent `Status: draft (aborted)` marker; write a decision record with `Outcome: aborted`. Do not delete partial work. |

These tokens are reserved. A user who literally needs the string
`/done` in an answer can quote it (`"/done"`). Treat any unquoted
leading `/` token not in the table above as a typo — ask rather than
guess.

Sub-skills may narrow, but never broaden or reinterpret:

- `spezi-gherkin` distill narrows `/back` from Read to a no-op
  synonym for `/abort` (Read has no prior step).
- `spezi-tdd` treats the TDD decision record as session-scoped
  writable; the rest of the read-only set remains so until
  `/update-spec` is explicitly invoked.

## Batch Q&A pattern

Every elicit phase and any prompt that asks for multiple inputs uses
the same batch shape:

- 3–7 questions per batch. Fewer if the phase is nearly trivial; more
  only if the user explicitly asks to be thorough.
- Questions are labeled (`a)`, `b)`, ...). The label becomes the
  user's reference for `/skip <letter>` or out-of-order answers.
- The batch ends with a closing instruction line:
  > Answer whichever you can in any order. `/done` if we have enough;
  > `/skip <letter>` to skip one; `/skip` alone to skip the phase.
- After the user answers, the sub-skill may ask **one** follow-up
  batch if critical gaps remain. Never chain more than two batches in
  a single phase — at that point, flag the remaining gaps as
  ambiguities and move on.

## Ambiguity-flag presentation

Any time the sub-skill has a grouped list of items it deliberately
did *not* resolve on the user's behalf — ambiguities in elicit,
meaning-changing observations in distill, alignment gaps in `--red`,
spec-divergence flags in `--green`, proposed spec rewrites in
`/update-spec` — it uses the same visual shape:

```
Flagged for your attention ([context]):
1. [topic] <one-sentence observation or question>
2. [topic] <observation>
3. [topic] <observation>

Reply with `1: <bucket>`, `1: <amended>`, or free-form for each.
Multiple on one line OK. Silence = <default bucket>.
```

Buckets by context:

| Context | Buckets | Default on silence |
|---------|---------|--------------------|
| Elicit ambiguities | `answer` / `defer` / `out of scope` | defer all |
| Distill observations | `accept` / `reject` / `defer` / `<amended>` | reject all (leave as-is) |
| `--red` alignment gaps | `generate` / `update-spec` / `skip` / `handoff` | skip all |
| `--green` spec divergence | `update-spec` / `skip` | skip all |
| `/update-spec` proposed rewrite | `accept` / `reject` / `defer` / `<amended>` | reject all (leave as-is) |

The user's replies map to the context-specific buckets defined in
each sub-skill's section. The *shape* — numbered list, one reply per
number, silence = default — is shared. Items are never silently
dropped: anything still open at the end of the relevant phase is
either re-surfaced once (Wrap-up for elicit) or logged to the
decision record as deferred.

## Full-section rewrite rule

Section updates are **replacements, never patches**. When a phase or
flag cycle is confirmed, the sub-skill composes the entire section's
prose from the accumulated answers, shows it to the user for
confirmation, and on confirmation writes the whole section into the
artifact — replacing whatever prior content that section held.

Consequences:

- A `/back` into a prior phase followed by edits may invalidate later
  sections. When Scope or Happy path is revised, the sub-skill must
  re-confirm that subsequent confirmed sections are still correct —
  do **not** auto-edit them, and do **not** leave them silently
  stale. Offer: "Edge cases was confirmed earlier under a different
  Scope. Keep as-is, revise, or discard?"
- Never hand-merge user feedback into a diff. If the user says
  "change X to Y" after seeing a proposed section, rewrite the whole
  section and re-present.

The `/update-spec` step in `spezi-tdd` applies this rule at the
section level only — it rewrites the affected section(s) in full,
not the whole file (unlike distill).

## Session state

A Spezi session is **in-memory**. The only durable artifacts are
on-disk files written at well-defined moments:

- Elicit: section writes on phase confirmation; decision record on
  Wrap-up or `/abort`.
- Distill: feature file and decision record on Confirm or `/abort`.
- `--red`: new test files as each `generate` resolves; TDD decision
  record on session end.
- `--green`: TDD decision record on session end.
- `/update-spec`: feature file section rewrite + decision-record
  append in a single step.

No sidecar session log is maintained beyond the decision record.
Resumption across sessions is out of scope. A crashed or abandoned
session that wrote partial confirmed sections leaves them on disk
with a `Status: draft` front-matter marker; the next session decides
whether to replace or continue.

## Decline memory

User declines to cross-skill hooks (notably the Allium assessment
hook) are scoped to the current Spezi invocation — never persisted.
A new invocation asks again. This preserves the dialogical principle
across time: Spezi never silently honours yesterday's "no." Persistent
opt-outs live in `.spezi/state.json`'s `config` block and are the
domain of the invoking component, not of the patterns here.
