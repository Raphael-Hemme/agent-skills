---
name: spezi
description: Catch-all router for Behavioural Spec Driven Development with Gherkin-flavoured `.feature.md` specs. Routes `/spezi [flags] [seed]` invocations to per-mode skills. Mirrors Allium's API (`elicit` / `distill` / `tend` / `weed` / `propagate`) one-to-one with `spezi-` prefixed siblings, so users who know Allium know Spezi. Each per-mode skill is independently invocable; this catch-all exists for users who don't want to memorise the modes. After elicit / distill / tend, fires the propagate hook.
---

# spezi

Catch-all router for Gherkin-flavoured Behavioural Spec Driven Development. Mirrors Allium's API: same five verbs, same vocabulary, same workflow. The actual session work lives in the per-mode skills. Invoke them directly when you know the mode; otherwise this skill picks the right one.

## Routing table

| Verb | Skill | Use when the user wants to ... |
|------|-------|--------------------------------|
| `--elicit` | `spezi-elicit` | Build a new spec through conversation. |
| `--distill` | `spezi-distill` | Extract a spec from existing code (reverse-engineer). |
| `--tend` | `spezi-tend` | Make targeted edits to an existing spec. |
| `--weed` | `spezi-weed` | Find / fix divergences between spec and code. |
| `--propagate` | `spezi-propagate` | Generate tests from a spec (hand off to Allium when available). |

Same vocabulary as Allium. The split: Allium owns formal `.allium` specs and property-based test generation; Spezi owns markdown-embedded Gherkin `.feature.md` specs and BDD-framework test generation. Specs link via the `allium:` front-matter block (see `spezi/reference/allium-linking.md`).

## Routing rules

Parse `/spezi [flags] [seed]` and dispatch:

- One mode flag → dispatch to that skill, pass the seed verbatim.
- Two or more mode flags → ask which one.
- No flag and no seed → ask which mode the user wants.
- No flag with a seed → ask, listing the five modes as choices, with the seed shown for context.

Suppression flags (anywhere in the invocation):

- `--no-propagate` → skip the post-spec propagate hook in elicit/distill/tend.

## Post-spec propagate hook

After `spezi-elicit`, `spezi-distill`, or `spezi-tend` returns and the just-written `.feature.md` has `status: complete` and at least one `## Scenario:`, the per-mode skill itself fires:

```
<mode> complete. Generate tests for this spec via /spezi-propagate?

Reply `yes` / `not now` / `never this session`.
```

This catch-all does not double-fire when it dispatched to one of those skills. `/spezi-weed` does **not** fire the propagate hook — its purpose is divergence-checking, not authoring.

## Output paths (consuming project)

```
specs/
├── gherkin/
│   ├── <slug>.feature.md
│   ├── lib/                       library specs (reusable integration patterns)
│   │   └── <lib-slug>.feature.md
│   └── decisions/<YYYY-MM-DD>-<slug>[-<type>].md
└── allium/                        Allium-owned
```

Per-mode skills create directories lazily on first write.

## Installation

Skills auto-discover by description; the slash-command shim is optional. To register `/spezi`:

`.claude/commands/spezi.md` (project) or `~/.claude/commands/spezi.md` (user):

```markdown
---
description: Spezi — Behavioural Spec Driven Development orchestrator.
argument-hint: "[--elicit|--distill|--tend|--weed|--propagate] [seed...]"
---

Treat `$ARGUMENTS` as the raw Spezi invocation. Load `spezi/SKILL.md`.
```

Place the Spezi tree at `spezi/` (project) or `~/.claude/skills/spezi/` (user). Optionally also install Allium for richer test generation on logic-heavy specs.

## Reference files

- `spezi/reference/feature-file.template.md` — `.feature.md` shape and rules.
- `spezi/reference/decision-record.template.md` — record path convention and shape per session type.
- `spezi/reference/allium-linking.md` — `allium:` block schema and the Allium availability probe.
