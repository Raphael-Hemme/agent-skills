# Allium linking and assessment hook

How `.feature.md` and `.allium` files link, when to ask whether to invoke Allium, and how to detect Allium's availability. Spezi-side only — never edit `.allium` files.

## `allium:` block schema (in `.feature.md` front matter)

Ordered list. Each entry:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `path` | relative POSIX | yes | Location of the linked Allium artifact, under `specs/allium/`. |
| `kind` | enum | yes | `tests` (Allium tests this spec), `plan` (test plan), `generated-from` (Allium generated from this spec). Unknown values preserved verbatim. |
| `linkedAt` | ISO 8601 UTC | yes | When the link was last established / reconciled. Not the Allium file's mtime. |
| `status` | enum | yes | `active`, `stale` (feature `updated` > entry `linkedAt`), or `pending` (Allium file does not yet exist). |

Empty list and missing key are equivalent on read; on write, omit the key when there are no entries.

## Round-trip back-reference (in the Allium file)

```
gherkin:
  path: specs/gherkin/<slug>.feature.md
  updatedAt: <ISO 8601 UTC timestamp>
```

Allium owns its file format; Spezi only prescribes this field name. Verify on read; never modify.

## Staleness

`<feature updated> > <entry linkedAt>` ⇒ `stale`. Signal, not failure. Resolution belongs to Allium, not to Spezi.

## Hook — when and how to ask

Fire after a successful elicit Wrap-up or distill Confirm (the just-written `.feature.md` has `status: complete` and at least one `## Scenario:`). Eligibility — all must hold:

1. Allium is available (probe below).
2. Allium hasn't already been declined this session.
3. The user invoked Spezi via `/spezi` (the catch-all). Direct `/spezi-elicit` and `/spezi-distill` invocations also fire the hook unless the user passed `--no-allium` or equivalent.

Prompt:

```
Allium assessment: Allium is available and this spec is ready for handoff.
Invoke Allium now to <one-line role summary>?

Reply `yes` / `not now` / `never this session`.
```

Role summary: first non-empty line of the located Allium SKILL.md's `description` field, truncated to 80 characters. Falls back to `"continue with the Allium-side workflow"`.

Responses:

- `yes` → invoke Allium with `{ mode: "post-gherkin", seed: <slug>, featureFile: <path> }`. Pre-prompt notice if any `stale`, missing-file, or mismatched-slug entries exist on the spec (`Notice: 1 stale, 1 missing-file Allium reference on this spec.`).
- `not now` → skip; record `allium-hook: declined (this invocation)` in the *Session trail*.
- `never this session` → same as `not now`, plus the decline persists for the rest of this Spezi invocation.

Decline memory is invocation-scoped, never persistent. A new invocation asks again.

## Availability probe

Read-only. First match wins:

1. Plugin manifest at `~/.claude/plugins/allium/SKILL.md`, `~/.claude/skills/allium/SKILL.md`, `.claude/plugins/allium/SKILL.md`, or `.claude/skills/allium/SKILL.md`.
2. Slash command at `.claude/commands/allium*.md` or `~/.claude/commands/allium*.md`.
3. Otherwise unavailable.

The probe must not invoke `/allium`, run Allium code, or trigger Allium's own setup. Detection only.

## Reconciliation — Spezi flags, Allium fixes

Spezi may detect on its own walk: stale entries, missing-file entries, mismatched-slug entries (the linked Allium file's `gherkin` back-reference points elsewhere), pending entries.

What Spezi does:

- **Hook accepted (`yes`)**: pass detected issues to Allium verbatim in the invocation payload. Allium reconciles.
- **Hook declined**: log detected issues in the decision record's *Session trail*. No automatic follow-up.

What Spezi never does: edit or delete `.allium` files; change an entry's `status` without explicit user consent (distill may flag a stale entry and mark it `stale` only with the user's accept).
