---
name: spezi
description: Catch-all entry point for Behavioral Spec Driven Development. Routes `/spezi [flags] [seed]` invocations to per-mode skills (`spezi-elicit`, `spezi-distill`, `spezi-read`, `spezi-red`, `spezi-green`) and asks which mode the user wants when the invocation is empty or ambiguous. Each per-mode skill is independently invocable; this catch-all exists for users who don't want to memorise the modes. After elicit / distill, fires the post-Gherkin Allium hook when Allium is available.
---

# spezi

Catch-all router for Behavioral Spec Driven Development. The actual session work lives in the per-mode skills. Invoke them directly (`/spezi-elicit`, `/spezi-distill`, `/spezi-read`, `/spezi-red`, `/spezi-green`) when you know the mode; otherwise this skill picks the right one.

## Routing

Parse `/spezi [flags] [seed]` and dispatch:

| Flag | Dispatch to |
|------|-------------|
| `--elicit` | `spezi-elicit` |
| `--distill` | `spezi-distill` |
| `--read` | `spezi-read` |
| `--red` | `spezi-red` |
| `--green` | `spezi-green` |

Pass the seed (everything after the flag) verbatim. Two or more mode flags → ask which one. No flag and no seed → ask which mode the user wants. No flag with a seed → ask, listing the five modes as choices.

`--no-allium` (anywhere in the invocation) suppresses the post-session Allium hook; pass the suppression to the dispatched skill.

## Allium hook (after elicit / distill)

After `spezi-elicit` or `spezi-distill` returns and the just-written `.feature.md` has `status: complete` and at least one `## Scenario:`, fire the hook from `spezi/reference/allium-linking.md` §Hook. Skip the hook if `--no-allium` was set or if the user already declined Allium this invocation. Spezi-elicit and spezi-distill also fire the hook when invoked directly — this catch-all does not double-fire when it dispatched to one of them.

## Output paths (consuming project)

```
specs/
├── gherkin/
│   ├── <slug>.feature.md
│   └── decisions/<YYYY-MM-DD>-<slug>[-<type>].md
└── allium/                   Allium-owned
```

Per-mode skills create directories lazily on first write.

## Installation

Skills auto-discover by description; the slash-command shim is optional. To register `/spezi`:

`.claude/commands/spezi.md` (project) or `~/.claude/commands/spezi.md` (user):

```markdown
---
description: Spezi — Behavioral Spec Driven Development orchestrator.
argument-hint: "[--elicit|--distill|--read|--red|--green] [seed...]"
---

Treat `$ARGUMENTS` as the raw Spezi invocation. Load `spezi/SKILL.md`.
```

Place the Spezi tree at `spezi/` (project) or `~/.claude/skills/spezi/` (user). Verify Allium availability at any time by asking Spezi to probe per `spezi/reference/allium-linking.md` §Availability probe — read-only, no install.

## Reference files

- `spezi/reference/feature-file.template.md` — `.feature.md` shape and rules.
- `spezi/reference/decision-record.template.md` — record path convention and shape per session type.
- `spezi/reference/allium-linking.md` — `allium:` block schema, hook prompt, availability probe.
