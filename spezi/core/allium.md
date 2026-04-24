# Spezi — Allium Orchestration

Spezi is the only component that talks about Allium. The Gherkin
sub-skill reads and writes the front-matter `allium` block per the
schema defined here, but does not know anything about *when* to
suggest invoking Allium, *how* to invoke it, or *what* to do with
stale references. All of that is orchestration, and orchestration is
Spezi's job.

This file specifies:

1. The linking convention between `.feature.md` and `.allium` files
   (schema of the front-matter `allium` block).
2. The post-Gherkin Allium assessment hook (when, how, with what
   prompt).
3. Router flag interpretation for the hook (`--gherkin`, `--both`,
   `--allium`).
4. Allium-side reconciliation — what Spezi does about stale or
   missing links, and what it deliberately does not.

## Linking convention

### Front-matter `allium` block — schema

A `.feature.md` may carry an ordered `allium` list in its YAML front
matter. Each entry is an object:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `path` | relative POSIX path, string | yes | Location of the linked Allium artifact, relative to the project root. Must point inside `specs/allium/`. |
| `kind` | enum string | yes | Role of the linked file. Canonical values: `tests` (Allium file tests this spec), `plan` (Allium-side test plan derived from this spec), `generated-from` (Allium file was generated from this spec). Unknown values are preserved verbatim for forward compatibility but may be reported at read time. |
| `linkedAt` | ISO 8601 UTC timestamp | yes | When the link was last established or reconciled. Not necessarily the Allium file's mtime. |
| `status` | enum string | yes | `active` — link is current. `stale` — the feature file has been updated after `linkedAt`. `pending` — the link was declared but the Allium file does not yet exist (intent, not fact). |

An empty list (`allium: []`) and a missing `allium` key are
equivalent on read. On write, prefer omitting the key entirely when
there are no references.

The Gherkin sub-skill applies this schema verbatim when it writes
`.feature.md`. See `spezi-gherkin/SKILL.md` §Linking behaviour for
which Gherkin operations touch the block.

### Round-trip back-reference

Every linked Allium file *should* carry a `gherkin` field pointing
back to the `.feature.md` that linked it. Minimum shape:

```
gherkin:
  path: specs/gherkin/<slug>.feature.md
  updatedAt: <ISO 8601 UTC timestamp>
```

The Gherkin side produces enough information for Allium to round-trip
the link (path, timestamp). The Allium side owns its own file format;
Spezi does not prescribe it beyond the back-reference field name.
Spezi verifies round-trip when it reads Allium files but never
modifies them.

### Staleness

A reference entry is *stale* when:

```
<feature file updated timestamp>  >  <entry linkedAt>
```

Staleness is a signal, not a failure. It means "the spec changed
after the Allium artifact last reconciled with it" — the artifact
may still be correct, may be obsolete, or may need regeneration.
Resolution belongs to Allium (typically through a follow-up
invocation), not to Spezi or Gherkin.

## Post-Gherkin assessment hook

The hook is the moment Spezi pauses and asks the user whether Allium
should be invoked now. It runs *after* `spezi-gherkin` returns
control — never inside the sub-skill.

### When it fires

Spezi evaluates the hook once per invocation, immediately after
`spezi-gherkin` exits. Eligibility is inferred from what landed on
disk during this invocation plus the current flag state; no explicit
handshake with the sub-skill is needed.

**Eligibility — all must hold.** If any fails, the hook stays silent
and the invocation ends normally.

1. The routing environment reports `alliumAvailable == true`.
2. The parsed flags do not contain `--gherkin` or `--both` (see
   *Router flag interpretation* below).
3. The user has not already declined Allium earlier in this
   invocation (see *Decline memory*).
4. Spezi detects, on disk, a `.feature.md` that was written or
   updated during the just-completed sub-skill run and whose front
   matter satisfies **both**:
    - `status` is `complete` (not `draft`, not `aborted`, not
      missing).
    - The document contains at least one `## Scenario:` block.

The sub-skill writes `status: complete` on successful elicit Wrap-up
or successful distill Confirm. `status: aborted` or `draft` makes
the file ineligible. This is the implicit contract between Spezi and
`spezi-gherkin`: the sub-skill does not return a signal — Spezi
reads the spec it just produced.

### Prompt shape

When eligible, Spezi renders:

```
Allium assessment: Allium is available and this spec is ready for
handoff. Invoke Allium now to <one-line role summary>?

Reply `yes` / `not now` / `never this session`.
```

The one-line role summary is supplied by the adapter's
`describeAllium()` when implemented; otherwise Spezi falls back to
`"continue with the Allium-side workflow"`. Spezi does not invent
specifics about what Allium will do.

### User responses

- `yes` → Spezi invokes Allium directly, passing
  `{ mode: "post-gherkin", seed: <slug>, featureFile: <path> }`.
  The invocation follows the adapter's Allium entry-point.
- `not now` → no invocation this run; the hook does not re-fire.
  Recorded in the just-written decision record's *Session trail* as
  `allium-hook: declined (this invocation)`.
- `never this invocation` (any of: `never this session`, `never`,
  `never this invocation`) → same as `not now`, plus sets the
  in-memory decline flag.

### Decline memory

Decline memory is in-memory only, scoped to the current Spezi
invocation. A new Spezi invocation starts clean. This preserves the
dialogical principle across time: Spezi asks again tomorrow rather
than silently honouring yesterday's "no." Persistent opt-out lives
in `.spezi/state.json`'s `config` block and is out of scope for this
step.

## Router flag interpretation

Three parser flags alter the hook's behaviour. They do not alter the
sub-skill's phase flow — `spezi-gherkin` runs identically regardless
of which of these were set.

| Flag observed | Effect on the hook |
|---------------|---------------------|
| `--gherkin` explicit | Gherkin-only intent. The hook does **not** fire. Existing `allium` references in the feature file are still honoured (Gherkin's read/write behaviour applies), but Spezi does not suggest invoking Allium. |
| `--both` explicit | The router already resolved `{gherkin, allium}` as targets. The hook does **not** fire; instead, Spezi invokes Allium directly after `spezi-gherkin` returns, with the same payload shape as a `yes` reply. No user turn is added. |
| `--allium` explicit | The router routes to Allium only; `spezi-gherkin` is not invoked. The hook is structurally unreachable. |
| None of the above | Default. The hook fires if eligibility holds. |

A parsed `--both` takes precedence over `--gherkin` if both were
somehow set — the router should have already rejected that as
contradictory, but the hook treats `--both` as the binding choice
defensively.

## Allium-side reconciliation

Spezi's reconciliation role is narrow: it *flags* problems and
*dispatches* to Allium. It never edits `.allium` files directly,
and neither does `spezi-gherkin`.

### What Spezi detects

Before and after a `spezi-gherkin` run, Spezi may walk the
just-touched feature file's `allium` block and note:

- **Stale entries** — `linkedAt` older than feature `updated`.
- **Missing-file entries** — `path` points at a file that does not
  exist on disk.
- **Mismatched-slug entries** — the linked Allium file's `gherkin`
  back-reference points at a different `.feature.md`.
- **Pending entries** — `status: pending` with no Allium file yet.

### What Spezi does

- **At dispatch time** (before invoking `spezi-gherkin`): nothing.
  Gherkin must read the spec with its existing references intact.
- **At hook time** (after `spezi-gherkin` returns, before the hook
  prompt): detected issues are summarised into a one-line
  pre-prompt notice:
  ```
  Notice: 1 stale, 1 missing-file Allium reference on this spec.
  ```
  The notice is informational; it does not gate or alter the hook.
- **On `yes`**: the reconciliation burden passes to Allium. Spezi's
  invocation payload includes the detected issues as a structured
  field the adapter forwards verbatim; Allium decides how to act.
- **On `not now` / `never this invocation`**: detected issues are
  recorded in the decision record's *Session trail* and the run
  ends. No automatic follow-up.

### What Spezi never does

- Edit or delete `.allium` files.
- Change the `status` of an entry without the user's explicit say-so
  (the Gherkin sub-skill may flag a stale entry during distill and
  mark it `stale` with user consent — that is a Gherkin-side
  behaviour documented in `spezi-gherkin/SKILL.md`).
- Install or configure Allium.
- Regenerate an Allium artifact from the spec.
