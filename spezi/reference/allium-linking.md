# Allium linking, availability probe, and handoff

How `.feature.md` files link to Allium artifacts, how to detect Allium's availability, and the handoff payload `spezi-propagate` uses. Spezi-side only — never edit `.allium` files from any Spezi skill.

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

`<feature updated> > <entry linkedAt>` ⇒ `stale`. Signal, not failure. Resolution belongs to Allium (or to a follow-up `/spezi-propagate` run), not to Spezi unilaterally.

## Availability probe

Read-only. First match wins:

1. Plugin manifest at `~/.claude/plugins/allium/SKILL.md`, `~/.claude/skills/allium/SKILL.md`, `.claude/plugins/allium/SKILL.md`, or `.claude/skills/allium/SKILL.md`.
2. Slash command at `.claude/commands/allium*.md` or `~/.claude/commands/allium*.md`.
3. Otherwise unavailable.

The probe must not invoke `/allium`, run Allium code, or trigger Allium's own setup. Detection only.

## Handoff payload (used by `spezi-propagate`)

When `spezi-propagate` chooses the Allium path (logic-heavy spec or explicit user request), invoke Allium with:

```
{
  mode: "post-gherkin",
  seed: "<slug>",
  featureFile: "specs/gherkin/<slug>.feature.md",
  detectedIssues: [           // optional; may be empty
    { kind: "stale", path: "specs/allium/<slug>.allium" },
    { kind: "missing-file", path: "specs/allium/<other>.allium" },
    { kind: "mismatched-slug", path: "..." }
  ]
}
```

Allium reconciles `detectedIssues` on its side. Spezi never reconciles `.allium` files itself.

## What Spezi never does

- Edit or delete `.allium` files.
- Change an `allium:` block entry's `status` without explicit user consent (`spezi-tend` and `spezi-distill` may flag a stale entry and mark it `stale` only with the user's accept).
- Install, configure, or regenerate Allium artifacts.
