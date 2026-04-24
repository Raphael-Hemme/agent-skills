# Spezi — Allium Orchestration

Spezi is the only component that talks about Allium. The Gherkin
sub-skill reads and writes the front-matter `allium` block per the
schema defined here but knows nothing about *when* or *how* to
invoke Allium — all orchestration lives here.

This file specifies the `.feature.md` ↔ `.allium` linking
convention, the post-Gherkin assessment hook, router flag
interpretation, and Allium-side reconciliation rules.

## Linking convention

### Front-matter `allium` block — schema

A `.feature.md` may carry an ordered `allium` list in its YAML front
matter. Each entry:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `path` | relative POSIX path | yes | Location of the linked Allium artifact, relative to the project root. Must point inside `specs/allium/`. |
| `kind` | enum string | yes | `tests` (Allium file tests this spec), `plan` (Allium-side test plan), or `generated-from` (Allium file was generated from this spec). Unknown values are preserved verbatim for forward compatibility. |
| `linkedAt` | ISO 8601 UTC | yes | When the link was last established or reconciled (not the Allium file's mtime). |
| `status` | enum string | yes | `active` (current), `stale` (feature updated after `linkedAt`), or `pending` (declared intent; Allium file does not yet exist). |

An empty list (`allium: []`) and a missing key are equivalent on
read. On write, prefer omitting the key entirely when there are no
references. See `spezi-gherkin/SKILL.md` §Linking behaviour for
which Gherkin operations touch the block.

### Round-trip back-reference

Every linked Allium file *should* carry a `gherkin` field pointing
back to the `.feature.md` that linked it:

```
gherkin:
  path: specs/gherkin/<slug>.feature.md
  updatedAt: <ISO 8601 UTC timestamp>
```

Allium owns its own file format; Spezi prescribes only the
back-reference field name. Spezi verifies round-trip when it reads
Allium files but never modifies them.

### Staleness

An entry is *stale* when `<feature updated> > <entry linkedAt>`.
Staleness is a signal, not a failure: the artifact may still be
correct, obsolete, or in need of regeneration. Resolution belongs
to Allium (typically via a follow-up invocation), not to Spezi or
Gherkin.

## Post-Gherkin assessment hook

The moment Spezi pauses and asks whether Allium should be invoked.
Evaluated once per invocation, immediately after `spezi-gherkin`
exits — never inside the sub-skill. Eligibility is inferred from
what landed on disk plus the current flag state; no explicit
handshake is needed.

### Eligibility

All must hold, or the hook stays silent and the invocation ends:

1. `alliumAvailable == true`.
2. Parsed flags contain neither `--gherkin` nor `--both` (see *Router
   flag interpretation* below).
3. The user has not declined Allium earlier in this invocation (see
   `spezi/core/patterns.md` §Decline memory).
4. A `.feature.md` was written or updated during the just-completed
   sub-skill run whose front matter has `status: complete` **and**
   at least one `## Scenario:` block.

The sub-skill writes `status: complete` on successful elicit Wrap-up
or distill Confirm. `status: aborted` or `draft` makes the file
ineligible. The contract is implicit: Spezi reads the spec the
sub-skill just produced.

### Prompt shape and responses

```
Allium assessment: Allium is available and this spec is ready for
handoff. Invoke Allium now to <one-line role summary>?

Reply `yes` / `not now` / `never this session`.
```

The role summary comes from the adapter's `describeAllium()` if
implemented; otherwise Spezi falls back to `"continue with the
Allium-side workflow"`.

- `yes` → Spezi invokes Allium via the adapter's entry-point,
  passing `{ mode: "post-gherkin", seed: <slug>, featureFile: <path> }`.
- `not now` → no invocation this run; the hook does not re-fire.
  Logged in the decision record's *Session trail* as
  `allium-hook: declined (this invocation)`.
- `never this invocation` (synonyms: `never this session`, `never`) →
  same as `not now`, plus sets the invocation-scoped decline flag
  per `spezi/core/patterns.md` §Decline memory.

## Router flag interpretation

These parser flags alter the hook but not `spezi-gherkin`'s own
flow:

| Flag | Effect on the hook |
|------|---------------------|
| `--gherkin` explicit | Hook does **not** fire. Existing `allium` references are still honoured by Gherkin's read/write behaviour. |
| `--both` explicit | Hook does **not** fire; Spezi invokes Allium directly after Gherkin returns, same payload as a `yes` reply. No user turn added. |
| `--allium` explicit | Gherkin is not invoked; hook is structurally unreachable. |
| None of the above | Default — hook fires if eligibility holds. |

`--both` defensively overrides `--gherkin` if both were somehow set
(the router should already have rejected that as contradictory).

## Allium-side reconciliation

Spezi *flags* problems and *dispatches* to Allium. It never edits
`.allium` files directly, and neither does `spezi-gherkin`.

### What Spezi detects

Before and after a `spezi-gherkin` run, Spezi may walk the
feature file's `allium` block and note:

- **Stale entries** — `linkedAt` older than feature `updated`.
- **Missing-file entries** — `path` points at a file that does not
  exist.
- **Mismatched-slug entries** — the linked Allium file's `gherkin`
  back-reference points at a different `.feature.md`.
- **Pending entries** — `status: pending` with no Allium file yet.

### What Spezi does

- **At dispatch time** (before Gherkin runs): nothing.
- **At hook time**: summarise detected issues into a one-line
  pre-prompt notice (`Notice: 1 stale, 1 missing-file Allium
  reference on this spec.`). Informational only — it does not gate
  the hook.
- **On `yes`**: hand the reconciliation burden to Allium. The
  invocation payload includes detected issues as a structured field
  the adapter forwards verbatim.
- **On `not now` / `never`**: log detected issues in the decision
  record's *Session trail*. No automatic follow-up.

### What Spezi never does

- Edit or delete `.allium` files.
- Change an entry's `status` without the user's explicit say-so.
  (Distill may flag a stale entry and mark it `stale` with user
  consent — a Gherkin-side behaviour.)
- Install, configure, or regenerate Allium artifacts.
