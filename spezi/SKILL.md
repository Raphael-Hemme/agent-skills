---
name: spezi
description: Meta-skill for Behavioral Spec Driven Development (BSDD). Orchestrates a Gherkin sub-skill and the Allium plugin through a dialogical workflow where specs are the source of truth. Invoke via `/spezi [flags] [seed]` — the router parses, resolves environment availability, and dispatches. See `spezi/core/router.md` for the full invocation grammar.
---

# Spezi

Behavioral Spec Driven Development orchestrator. Spezi is a thin
coordinator: it owns the parser, the router, and the environment
probe cache. Actual authoring and execution live in the sub-skills.

## Principles

These hold across every file in `spezi/`:

- **Agent-agnostic core.** Everything under `spezi/core/`,
  `spezi/gherkin/`, and `spezi/tdd/` is prose that any conforming
  agent can follow. No harness-specific paths, tool names, or APIs.
- **Thin adapter layer.** Harness integration lives only in
  `spezi/adapters/<harness>.md`. Adding a new harness means adding an
  adapter, never touching the core.
- **Dialogical workflow.** No significant decision is made silently.
  Ambiguity → grouped flag list. Environment fallback →
  `degradationNotices`. Destructive or meaning-changing action →
  explicit confirmation.
- **Specs are the source of truth.** Implementation, tests, and
  Allium artifacts follow `.feature.md`. When they diverge, the spec
  update step is the *only* path back into the feature file.
- **Agent Skills open standard.** Each SKILL.md carries YAML front
  matter with `name` and `description`. Sub-skills are discoverable
  by their frontmatter, not by hard-coded imports.

## Invocation reference

```
/spezi [flags] [seed]
/spezi <subcommand> [flags]
```

**Flags** (boolean, long-form only):

| Flag | Meaning | Specified in |
|------|---------|--------------|
| `--elicit` | Author a new spec | `spezi/gherkin/SKILL.md` §Elicit mode |
| `--distill` | Tighten an existing spec | `spezi/gherkin/SKILL.md` §Distill mode |
| `--read` | Read-only spec reference | `spezi/tdd/SKILL.md` §`--read` mode |
| `--red` | TDD red phase: alignment + test generation | `spezi/tdd/SKILL.md` §`--red` mode |
| `--green` | TDD green phase: implementation guidance | `spezi/tdd/SKILL.md` §`--green` mode |
| `--refactor` | TDD refactor phase | Deferred |
| `--allium` | Target Allium sub-skill | `spezi/core/router.md` §2 |
| `--gherkin` | Target Gherkin sub-skill | `spezi/core/router.md` §2 |
| `--both` | Target both sub-skills | `spezi/core/router.md` §2 |
| `--recheck` | Force re-probe of cached Allium availability | `spezi/core/router.md` §2 |

**Subcommands** (positional, first non-flag token):

| Subcommand | Meaning | Specified in |
|------------|---------|--------------|
| `check` | Probe and report environment | `spezi/core/router.md` §2.A |
| `status` | Report cached state | `spezi/core/router.md` §2.B |

Any text that isn't a recognized flag or subcommand is the **seed** —
free-form context the router hands to the sub-skills. Parser grammar,
whitespace rules, and the full test matrix: `spezi/core/router.md` §1.

### Examples

```
/spezi --elicit add SSO to login
/spezi --distill sso-login
/spezi --read sso-login
/spezi --red empty-cart checkout
/spezi --green sso-login happy path
/spezi check
/spezi check --allium
/spezi status
/spezi --recheck --elicit add rate limiting
```

### Quick routing summary

- Target flag explicit (`--allium`, `--gherkin`, `--both`) → honored.
- No target flag:
  - Authoring modes (`--elicit`, `--distill`, `--read`) → Gherkin.
  - TDD phases (`--red`, `--green`, `--refactor`) → both.
  - No mode either → clarification asked.
- Allium unavailable: implicit fallback drops Allium with a notice;
  explicit Allium intent escalates to clarification.

Full decision tree, every edge case: `spezi/core/router.md` §2.

## Directory layout

```
spezi/
├── SKILL.md                   (this file)
├── core/
│   ├── router.md              argument parser + routing logic
│   └── state.md               .spezi/state.json schema + I/O
├── gherkin/
│   └── SKILL.md               spec authoring: elicit + distill modes,
│                              shared concerns, file formats
├── tdd/
│   └── SKILL.md               --read, --red, --green protocol;
│                              read-only contract; /update-spec
└── adapters/
    └── claude-code.md         Claude Code integration (only)
```

Produced artifacts (in the project consuming Spezi):

```
specs/
├── gherkin/
│   ├── <slug>.feature.md              spec files
│   └── decisions/
│       └── <date>-<slug>[-<mode>].md  decision records per session
└── allium/                             Allium artifacts (Allium owns this)

.spezi/
└── state.json                 runtime cache (gitignored)
```

The `.spezi/` directory is added to `.gitignore` automatically on
first state write — see `spezi/core/state.md` §Gitignore handling.

## Sub-skill index

| Component | Path | Role |
|-----------|------|------|
| **Core router** | `spezi/core/router.md` | Parse the invocation; resolve Allium availability (read/write `.spezi/state.json`); emit a `RoutingDecision`. No user turns. |
| **Core state** | `spezi/core/state.md` | Runtime cache contract: schema, read/write, invalidation, gitignore handling. |
| **Gherkin sub-skill** | `spezi/gherkin/SKILL.md` | Spec authoring (`--elicit`, `--distill`), `.feature.md` and decision-record formats, Allium assessment hook, linking conventions. |
| **TDD protocol** | `spezi/tdd/SKILL.md` | Cross-sub-skill TDD phases (`--read`, `--red`, `--green`). Read-only enforcement contract. `/update-spec` escape hatch. |
| **Allium** | external plugin | Test generation (`--red`), execution (`--green`), and Allium-side artifacts. Not part of this repo. Spezi probes for it; it is otherwise opaque to the core. |

## Adapter layer

Environment-specific integration is factored out of the core and
lives under `spezi/adapters/`.

| Adapter | Path | Status |
|---------|------|--------|
| Claude Code | `spezi/adapters/claude-code.md` | Current. |

Each adapter implements the contract the core expects
(`probeAllium()`, optional `describeAllium()`, state I/O, dispatch,
rendering of structured router output) and nothing else. The adapter
file is **the only** place that may name a specific harness — the
core and sub-skill files must remain harness-agnostic.

A new harness (e.g. a JetBrains plugin, a CI runner, a different
CLI) is added by writing a new file under `spezi/adapters/` and a
matching slash-command / trigger; no changes to the core are
required.

## Extension points (not yet implemented)

The following are named here so future work can slot in without
disturbing existing files:

- **Dialogue stage** — structured handling of `clarificationNeeded`
  and `degradationNotices`. Currently, adapters render them as
  plain text and re-enter the router on the user's reply. See
  `spezi/core/router.md`.
- **`--refactor` mode** — TDD refactor phase. Named in the flag
  table; unspecified.
- **Persistent config** — `state.json` already carries a `config`
  object (see `spezi/core/state.md`). No keys are defined yet; user
  preferences like `allium.ttlDays` or a persistent Allium
  opt-out land here when specified.
- **Additional adapters** — any non-Claude-Code harness.

## Where to start reading

- Curious about the invocation surface → this file's *Invocation
  reference*.
- Implementing or auditing the parser/router → `spezi/core/router.md`.
- Running an elicit or distill session → `spezi/gherkin/SKILL.md`.
- Running a TDD phase → `spezi/tdd/SKILL.md`.
- Wiring Spezi into Claude Code → `spezi/adapters/claude-code.md`.
- Understanding the runtime cache → `spezi/core/state.md`.
