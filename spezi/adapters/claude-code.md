# Claude Code Adapter

This is the thin integration layer between Spezi's agent-agnostic core
(`spezi/core/`) and its sibling sub-skills (`spezi-gherkin/`,
`spezi-tdd/`) and Claude Code's invocation model. Nothing here
re-specifies routing, parsing, dialogue, or sub-skill behaviour —
those live in the core files and this adapter delegates to them.

This file is **the only** place in Spezi that names Claude Code
specifics. A sibling adapter for a different harness would replace it
entirely without touching the core.

## What the adapter owns

1. The `/spezi` slash command registration.
2. The entry-point handoff: turn Claude Code's invocation into a raw
   string the core router can parse.
3. The environment probes the core expects from any adapter:
   `probeAllium()` and (optional) `describeAllium()`.
4. Dispatch: invoke the Gherkin sub-skill (in-repo) and the Allium
   plugin (installed separately) per the `RoutingDecision` the core
   emits.
5. Rendering: turn the `RoutingDecision`'s structured fields
   (`diagnostic`, `degradationNotices`, `clarificationNeeded`) into
   plain-text turns until the stage-3 dialogue protocol is specified.

## What the adapter does not own

- Argument parsing — see `spezi/core/router.md` §1.
- Routing decisions — see `spezi/core/router.md` §2.
- Sub-skill behaviour — see `spezi-gherkin/SKILL.md` and
  `spezi-tdd/SKILL.md`.
- State file format — see `spezi/core/state.md`.
- Gitignore handling — see `spezi/core/state.md` §Gitignore handling.
- Allium orchestration (linking schema, assessment hook, decline
  memory, reconciliation) — see `spezi/core/allium.md`.
- Shared dialogical patterns (escape hatches, batch Q&A, ambiguity
  flags, full-section rewrite, session state) — see
  `spezi/core/patterns.md`.
- Decision-record schema — see `spezi/core/decision-records.md`.

The adapter must not introduce new rules for any of the above. If a
rule is missing from the core, add it to the core — not here.

## Slash command registration

Install the slash command at project scope:
`.claude/commands/spezi.md`.

For user scope (available in every project), install at:
`~/.claude/commands/spezi.md`.

Minimum contents:

```markdown
---
description: Spezi — Behavioral Spec Driven Development orchestrator.
argument-hint: "[flags] [seed...]"
---

Treat the string `$ARGUMENTS` as the raw Spezi invocation.

Load `spezi/SKILL.md`. Follow its invocation reference. Parse the
arguments per `spezi/core/router.md` §1. Resolve Allium availability
via this adapter (`spezi/adapters/claude-code.md`). Route per
`spezi/core/router.md` §2. Dispatch to the named sub-skills per their
own SKILL.md files.

Do not invent additional rules. If the input resolves to
`kind: "clarify"`, render the clarification block and await the user's
reply, then re-enter the router with the amended invocation.
```

A project that vendors Spezi under a non-default path must adjust the
`Load` line; everything else stays the same.

## Invocation flow

When the user types `/spezi <args>` inside a Claude Code session:

1. Claude Code resolves the slash command file and substitutes
   `$ARGUMENTS` with `<args>` verbatim (including empty).
2. The command file directs Claude Code to load `spezi/SKILL.md`.
3. Claude Code (acting as the agent) executes the protocol described
   by the core router: parse → resolve availability → route → dispatch.
4. Dispatched sub-skills read their own SKILL.md and run.
5. Any clarification or diagnostic surfaces back to the user as a
   plain-text turn. The user's next message is the amended
   invocation (or a reply to the clarification), and step 3 runs
   again with that input.

Nothing in this flow is special to Claude Code except the slash-command
trigger itself. The argument string, the core logic, and the sub-skill
invocations are harness-independent.

## Adapter contract — implementations

### `probeAllium()`

Required. Read-only. Returns
`{ available: boolean, detail?: string }`.

Detection strategy for Claude Code (first success wins, in order):

1. **Plugin manifest.** Look for an Allium skill under a conventional
   plugin location. Canonical candidates, checked in this order:
   - `~/.claude/plugins/allium/SKILL.md`
   - `~/.claude/skills/allium/SKILL.md`
   - `.claude/plugins/allium/SKILL.md` (project-scope)
   - `.claude/skills/allium/SKILL.md` (project-scope)
   If found, `detail` is the matching path.
2. **Slash command presence.** If an `/allium` slash command is
   registered (`.claude/commands/allium*.md` or the user-scope
   equivalent), treat Allium as available. `detail` is the command
   path.
3. **Negative.** If none of the above match, return
   `{ available: false, detail: "allium not found in plugin or
   command paths" }`.

The probe must **not** invoke `/allium`, run Allium code, or
trigger Allium's own initialization. Detection only.

### `describeAllium()` — optional

When Allium is available, return a one-line role summary for the
assessment-hook prompt (see `spezi/core/allium.md` §Post-Gherkin
assessment hook). Recommended: the first non-empty line of the
description field in the located Allium `SKILL.md`'s YAML front
matter, truncated to 80 characters. Do not fabricate. If
unimplemented, Spezi falls back to `"continue with the Allium-side
workflow"`.

### State I/O

Use Claude Code's Read/Write tools to honor `spezi/core/state.md`.
Atomic write: serialize to `state.json.tmp`, then overwrite
`state.json` (or rename, if the harness exposes it). Do not embed
Claude Code specifics in `state.json` content — only schema-defined
fields.

## Dispatch

The adapter consumes the `RoutingDecision.kind` and acts:

| `kind` | Action |
|--------|--------|
| `invoke` | For each entry in `targets`, invoke the named sub-skill with `{ mode, seed, parsed }` plus (for `allium`) any `featureFile` already fixed by a prior Gherkin pass. Ordering: the gherkin-side skill runs before Allium when both are listed. Spezi waits for the sub-skill to exit, then inspects the just-written `.feature.md` to decide whether to invoke Allium (see `spezi/core/allium.md` §Post-Gherkin assessment hook). No explicit handoff signal — the interface is implicit via disk state. |
| `check` | Render `diagnostic` as a plain-text report; append each entry of `degradationNotices` on its own line prefixed `note:`. No sub-skill invoked. |
| `status` | Same rendering pattern as `check`, from the cached state only. `environment.probed` is always `false`; do not re-probe here. |
| `clarify` | Render `clarificationNeeded` as a numbered choice list with the structured reason shown inline. Await the user's next message. When it arrives, re-enter the router with an amended invocation — either the user's reply concatenated to the prior invocation, or a fresh invocation if the user rephrased wholesale. |

### Invoking the gherkin-side sub-skill

The router's `skill: "gherkin"` target resolves by mode:

- `elicit`, `distill` → `spezi-gherkin/SKILL.md`
- `read`, `red`, `green`, `refactor` → `spezi-tdd/SKILL.md`

Invocation loads the chosen SKILL.md, passes `{ mode, seed, parsed }`,
and follows its flow. The adapter does not mix the two skills or
inject logic between them.

### Invoking Allium

Call the registered Allium entry-point (slash command or plugin hook
— whichever the probe found) with `{ mode, seed, featureFile }`.
Under `--both`, the gherkin-side skill runs to completion first,
then Allium is invoked with the just-written feature file. There is
no explicit handoff signal — Spezi infers completion by the sub-skill
returning control and by inspecting the `.feature.md`'s `status`
field (see `spezi/core/allium.md` §Post-Gherkin assessment hook).

### Degradation

When `RoutingDecision.degradationNotices` is non-empty, surface each
entry to the user before acting on `kind`. Never fold a notice into
the sub-skill's prose silently.

## Installation checklist

1. Place the Spezi tree at `spezi/` (project scope) or
   `~/.claude/skills/spezi/` (user scope).
2. Create `.claude/commands/spezi.md` or `~/.claude/commands/spezi.md`
   with the minimum contents above.
3. Install the Allium plugin per its own instructions — Spezi does
   **not** install it.
4. Verify with `/spezi check`. Either outcome (Allium available or
   unavailable) is a valid configuration.

## Non-goals

- No auto-install of Allium.
- No modification of the user's Claude Code settings beyond the
  slash-command file.
- No telemetry, no remote calls, no background tasks.
- No behavioral divergence from the core. If Claude Code forces a
  compromise, document it explicitly here; do not drift the core to
  accommodate it.
