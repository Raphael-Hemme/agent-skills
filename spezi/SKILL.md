---
name: spezi
description: Meta-skill for Behavioral Spec Driven Development (BSDD). Parses `/spezi` invocations, resolves Allium availability, routes to the sibling sub-skills (`spezi-gherkin`, `spezi-tdd`), and — after a successful Gherkin session — asks whether to invoke Allium. No dialogue logic, no phase logic: those live in the sub-skills.
---

# Spezi

Spezi is the orchestrator. It owns the parser, the router, the
runtime state cache, the Allium availability probe, the linking
convention between `.feature.md` and `.allium` files, and the
post-Gherkin Allium assessment hook. Everything else — spec
authoring, TDD phase protocol, test generation — lives in sibling
skills.

## Principles

These hold across every file under `spezi/` and across the sibling
sub-skills:

- **Agent-agnostic core.** The logic under `spezi/core/` and the
  body of each sibling sub-skill is prose any conforming agent can
  follow. No harness-specific paths, tool names, or APIs.
- **Thin adapter layer.** Harness integration lives only in
  `spezi/adapters/<harness>.md`. Adding a new harness means adding
  an adapter file; no core change required.
- **Dialogical workflow.** No significant decision is made silently.
  Ambiguity → grouped flag list. Environment fallback →
  degradation notice. Destructive or meaning-changing action →
  explicit confirmation.
- **Specs are the source of truth.** Implementation, tests, and
  Allium artifacts follow `.feature.md`. The only path that writes
  a `.feature.md` during a TDD session is the spec update step.
- **Agent Skills open standard.** Each SKILL.md carries YAML front
  matter with `name` and `description`. Sub-skills are discoverable
  by frontmatter, not by hard-coded imports.

## Invocation

```
/spezi [flags] [seed]
/spezi <subcommand> [flags]
```

**Flags:**

| Flag | Meaning | Specified in |
|------|---------|--------------|
| `--elicit` | Author a new spec | `spezi-gherkin/SKILL.md` §Elicit mode |
| `--distill` | Tighten an existing spec | `spezi-gherkin/SKILL.md` §Distill mode |
| `--read` | Read-only spec reference | `spezi-tdd/SKILL.md` §`--read` mode |
| `--red` | TDD red phase | `spezi-tdd/SKILL.md` §`--red` mode |
| `--green` | TDD green phase | `spezi-tdd/SKILL.md` §`--green` mode |
| `--refactor` | TDD refactor phase | Deferred |
| `--allium` | Target Allium only | `spezi/core/router.md` §2 |
| `--gherkin` | Target the gherkin-side skill only (no Allium hook) | `spezi/core/router.md` §2, `spezi/core/allium.md` §Router flag interpretation |
| `--both` | Target gherkin-side skill + Allium | `spezi/core/router.md` §2, `spezi/core/allium.md` §Router flag interpretation |
| `--recheck` | Force re-probe of cached Allium availability | `spezi/core/router.md` §2 |

**Subcommands** (positional, first non-flag token):

| Subcommand | Meaning | Specified in |
|------------|---------|--------------|
| `check` | Probe and report environment | `spezi/core/router.md` §2.A |
| `status` | Report cached state | `spezi/core/router.md` §2.B |

Free-form residual text after flag/subcommand parsing is the **seed**
— context the router passes to the target skill.

### Examples

```
/spezi --elicit add SSO to login
/spezi --distill sso-login
/spezi --read sso-login
/spezi --red empty-cart checkout
/spezi --green sso-login happy path
/spezi check
/spezi status
/spezi --recheck --elicit add rate limiting
```

## Architecture

Four skills sit at the same directory level:

```
spezi/                          orchestrator
├── SKILL.md                    (this file)
├── core/
│   ├── router.md               parser + routing
│   ├── state.md                .spezi/state.json schema + I/O
│   ├── allium.md               linking schema, assessment hook,
│   │                           decline memory, reconciliation
│   ├── patterns.md             shared dialogical patterns
│   │                           (escape hatches, batch Q&A,
│   │                           ambiguity flags, full-section
│   │                           rewrite, session state)
│   └── decision-records.md     decision-record schema: base +
│                               distill and TDD extensions
└── adapters/
    └── claude-code.md          Claude Code integration

spezi-gherkin/                  spec authoring
└── SKILL.md                    elicit + distill, file format

spezi-tdd/                      TDD phase runner
└── SKILL.md                    --read / --red / --green, read-only
                                contract, /update-spec

allium/                         external plugin (not in this repo)
```

Produced artifacts in the consuming project:

```
specs/
├── gherkin/
│   ├── <slug>.feature.md              specs
│   └── decisions/
│       └── <date>-<slug>[-<mode>].md  decision records per session
└── allium/                             Allium-owned

.spezi/
└── state.json                 runtime cache (gitignored; see
                               spezi/core/state.md)
```

## Orchestration

Spezi invokes at most one gherkin-side sub-skill per invocation,
plus (optionally) Allium. The gherkin-side selection is by mode:

| Mode | Gherkin-side target |
|------|---------------------|
| `elicit`, `distill` | `spezi-gherkin` |
| `read`, `red`, `green`, `refactor` | `spezi-tdd` |

The router's `RoutingDecision.targets` uses the logical name
`gherkin`; the dispatcher (adapter) resolves that to the sub-skill
above based on mode. This preserves the router's target vocabulary
while letting the physical split live at the file-system level.

After a gherkin-side sub-skill returns control, Spezi inspects the
just-written `.feature.md`. If it carries `status: complete` and at
least one scenario, the Allium assessment hook is eligible (unless
suppressed by `--gherkin` or `--both`). See `spezi/core/allium.md`
for full rules.

The interface between Spezi and the sub-skills is **implicit**: the
sub-skill writes its artifacts and exits; Spezi reads the result
from disk. There is no explicit handoff signal.

## Sub-skill index

| Component | Path | Role |
|-----------|------|------|
| Core router | `spezi/core/router.md` | Parse; resolve availability; emit `RoutingDecision`. No user turns. |
| Core state | `spezi/core/state.md` | Runtime cache contract: schema, read/write, invalidation, gitignore handling. |
| Core Allium | `spezi/core/allium.md` | `.feature.md` ↔ `.allium` linking schema, assessment hook, decline memory, reconciliation. |
| Core patterns | `spezi/core/patterns.md` | Escape hatches, batch Q&A, ambiguity flags, full-section rewrite, session state. Canonical for every sub-skill. |
| Core records | `spezi/core/decision-records.md` | Decision-record path convention, base schema, distill and TDD extensions. |
| Gherkin authoring | `spezi-gherkin/SKILL.md` | `--elicit`, `--distill`. Feature-file format. Mode-specific flow only; patterns and records live in core. |
| TDD phases | `spezi-tdd/SKILL.md` | `--read`, `--red`, `--green`. Read-only contract, `/update-spec`. Patterns and records live in core. |
| Allium | external plugin | Test generation, execution, Allium-side artifacts. Opaque to Spezi. |

## Adapter layer

| Adapter | Path | Status |
|---------|------|--------|
| Claude Code | `spezi/adapters/claude-code.md` | Current |

Only adapter files name a specific harness. Adding a new harness =
new adapter file; no change to `spezi/core/` or the sibling
sub-skills.

## Extension points (deferred)

- **Dialogue stage.** Structured handling of `clarificationNeeded`
  and `degradationNotices`; adapters currently render them as plain
  text and re-enter the router.
- **`--refactor` mode.** Named in the flag table; unspecified.
- **Persistent config.** `state.json` has a `config` object; no keys
  are defined yet. User preferences like `allium.ttlDays` or a
  persistent Allium opt-out land here when specified.
- **Additional adapters.** Any non-Claude-Code harness.

## Where to start reading

- Invocation surface → this file's *Invocation* section.
- Parser/router → `spezi/core/router.md`.
- Elicit / distill → `spezi-gherkin/SKILL.md`.
- TDD phase → `spezi-tdd/SKILL.md`.
- Shared dialogical patterns → `spezi/core/patterns.md`.
- Decision-record schema → `spezi/core/decision-records.md`.
- Allium hook, linking, decline memory → `spezi/core/allium.md`.
- Claude Code wiring → `spezi/adapters/claude-code.md`.
- Runtime cache → `spezi/core/state.md`.
