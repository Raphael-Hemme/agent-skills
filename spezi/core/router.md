# Spezi Router

The router is Spezi's entry point. It has three stages:

1. **Argument parsing** — turn the raw invocation into a structured
   intent.
2. **Routing** — resolve environment availability and decide which
   sub-skill(s) run, in which mode.
3. **Dialogue** — confirm intent and degradations with the user before
   acting. *(Not yet implemented. Deferred to a later step.)*

Stages 1 and 2 are pure description: they produce structured outputs
and never speak to the user. All user-visible prose — prompts,
confirmations, degradation notices — belongs to stage 3. A conforming
implementation must not smuggle stage 3 behavior into stages 1 or 2.

## 1. Argument Parsing

### Interface

```
parse(invocation: string) -> ParsedInvocation
```

Input: a single raw string, exactly what the user (or upstream adapter)
passed to Spezi. It may be empty.

Output: a `ParsedInvocation` object with the shape:

```json
{
  "subcommand": "check" | "status" | null,
  "flags": {
    "elicit":   false,
    "distill":  false,
    "allium":   false,
    "gherkin":  false,
    "both":     false,
    "read":     false,
    "green":    false,
    "red":      false,
    "refactor": false,
    "recheck":  false
  },
  "seed": "",
  "unknown": []
}
```

- `subcommand`: one of the recognized subcommands, else `null`.
- `flags`: every known flag is present with a boolean. Omitted flags
  default to `false`. No `undefined`/`null` values for known flags.
- `seed`: the free-form residual text, whitespace-normalized, trimmed.
  Empty string when there is no residual.
- `unknown`: a list of tokens that began with `--` but did not match
  any known flag. Order-preserving, duplicates preserved. Parsing does
  not fail on unknowns — they are surfaced so the router (a later
  stage) can decide what to do.

The parser is pure: same input → same output. It performs no I/O, no
state reads, no user prompts. It produces no side effects.

### Recognized tokens

Flags (boolean, long-form only):

| Flag          | Meaning (parser treats as opaque) |
|---------------|-----------------------------------|
| `--elicit`    | elicit mode                       |
| `--distill`   | distill mode                      |
| `--allium`    | select Allium target              |
| `--gherkin`   | select Gherkin target             |
| `--both`      | select both targets               |
| `--read`      | read mode                         |
| `--green`     | TDD green phase                   |
| `--red`       | TDD red phase                     |
| `--refactor`  | TDD refactor phase                |
| `--recheck`   | force re-probe of cached state    |

Subcommands (positional, case-sensitive, lowercase):

| Subcommand | Meaning (parser treats as opaque) |
|------------|-----------------------------------|
| `check`    | environment check                 |
| `status`   | report current state              |

The parser records these verbatim. It does **not** interpret their
effects (e.g. `--both` is not expanded into `--allium --gherkin` here;
that is a routing concern).

### Grammar and rules

1. **Tokenization.** Split the trimmed input on runs of ASCII
   whitespace (` `, `\t`, `\n`). No shell-style quoting. A user who
   needs to preserve internal whitespace in the seed may rely on the
   fact that single spaces are reinserted when the seed is rejoined;
   multiple spaces are collapsed.

2. **Flag classification.** A token is a *flag candidate* iff it
   begins with `--`. If the candidate exactly matches a key in the
   flags table, set that flag to `true`. Otherwise, append the
   verbatim token to `unknown`. Flag candidates are never contributed
   to `seed`.

3. **No flag values.** No flag in the current set takes a value. A
   token like `--recheck=true` does not match `--recheck` and is
   recorded in `unknown`. This keeps the grammar uniform and avoids
   silent coercion.

4. **Duplicate flags.** A flag that appears more than once stays
   `true`. Duplicates are not errors and are not reported.

5. **Subcommand detection.** After flag classification, walk the
   remaining (non-flag) tokens in original order. The *first*
   non-flag token is eligible to become the `subcommand`, and only if
   it exactly matches a subcommand name. If it does, it is consumed
   and does not contribute to `seed`. Any later token that happens to
   match a subcommand name is **not** treated as a subcommand — it
   stays in `seed`. This makes invocations unambiguous: subcommands
   are positional, not keywords.

6. **Seed assembly.** Remaining non-flag, non-subcommand tokens are
   joined with a single space, in original order. The result is
   trimmed. Empty residual produces `""`, not `null`.

7. **Casing.** All matching is case-sensitive. `--Recheck`, `CHECK`,
   or `Status` are not recognized — `--Recheck` lands in `unknown`;
   `CHECK`/`Status` land in `seed`.

8. **Empty input.** An empty or whitespace-only invocation yields
   the zero value: `subcommand: null`, all flags `false`, `seed: ""`,
   `unknown: []`.

9. **Determinism.** Parsing is stable under reordering of flag
   tokens. It is **not** stable under reordering of non-flag tokens,
   because only the first non-flag token can be a subcommand.

### Test cases

Each case lists the input on the left and the non-default fields of
the resulting `ParsedInvocation` on the right. Any field not listed
holds its zero value (`false` for flags, `null` for subcommand,
`""` for seed, `[]` for unknown).

**Zero and singletons**

| Input | Result |
|-------|--------|
| `` (empty) | — |
| `check` | `subcommand=check` |
| `status` | `subcommand=status` |
| `--elicit` | `flags.elicit=true` |
| `--distill` | `flags.distill=true` |
| `--allium` | `flags.allium=true` |
| `--gherkin` | `flags.gherkin=true` |
| `--both` | `flags.both=true` |
| `--read` | `flags.read=true` |
| `--green` | `flags.green=true` |
| `--red` | `flags.red=true` |
| `--refactor` | `flags.refactor=true` |
| `--recheck` | `flags.recheck=true` |

**Subcommand + flag**

| Input | Result |
|-------|--------|
| `status --recheck` | `subcommand=status`, `flags.recheck=true` |
| `check --both` | `subcommand=check`, `flags.both=true` |
| `check --allium --recheck` | `subcommand=check`, `flags.allium=true`, `flags.recheck=true` |

**Flag order independence**

| Input | Result |
|-------|--------|
| `--recheck status` | `subcommand=status`, `flags.recheck=true` |
| `--allium check --recheck` | `subcommand=check`, `flags.allium=true`, `flags.recheck=true` |

**Free-form seed**

| Input | Result |
|-------|--------|
| `the user should be able to log in with SSO` | `seed="the user should be able to log in with SSO"` |
| `check authentication flow` | `subcommand=check`, `seed="authentication flow"` |
| `--distill users must reset their password every 90 days` | `flags.distill=true`, `seed="users must reset their password every 90 days"` |
| `--elicit --allium add 2fa to the login page` | `flags.elicit=true`, `flags.allium=true`, `seed="add 2fa to the login page"` |

**TDD-phase flags with seed**

| Input | Result |
|-------|--------|
| `--red write a failing test for empty cart checkout` | `flags.red=true`, `seed="write a failing test for empty cart checkout"` |
| `--green` | `flags.green=true` |
| `--refactor extract the rate limiter` | `flags.refactor=true`, `seed="extract the rate limiter"` |
| `--read review the current auth specs` | `flags.read=true`, `seed="review the current auth specs"` |

**Duplicates**

| Input | Result |
|-------|--------|
| `--red --red` | `flags.red=true` |
| `--recheck status --recheck` | `subcommand=status`, `flags.recheck=true` |

**Subcommand-name tokens that are not subcommands**

| Input | Result |
|-------|--------|
| `foo check` | `seed="foo check"` *(first non-flag token is `foo`, not a subcommand)* |
| `check status` | `subcommand=check`, `seed="status"` *(second `status` is seed)* |
| `please check the login` | `seed="please check the login"` |

**Unknown flags**

| Input | Result |
|-------|--------|
| `--unknown` | `unknown=["--unknown"]` |
| `--recheck --foo check` | `subcommand=check`, `flags.recheck=true`, `unknown=["--foo"]` |
| `--recheck=true` | `unknown=["--recheck=true"]` *(values not supported)* |
| `--Recheck` | `unknown=["--Recheck"]` *(case-sensitive)* |

**Casing of non-flag tokens**

| Input | Result |
|-------|--------|
| `CHECK` | `seed="CHECK"` |
| `Status --recheck` | `flags.recheck=true`, `seed="Status"` |

**Whitespace**

| Input | Result |
|-------|--------|
| `   ` (whitespace only) | — |
| `  --elicit   a   seed  ` | `flags.elicit=true`, `seed="a seed"` |
| `check\t--both\nlogin` (tab/newline) | `subcommand=check`, `flags.both=true`, `seed="login"` |

## 2. Routing

The routing stage consumes a `ParsedInvocation`, resolves environment
availability through the adapter, and produces a `RoutingDecision` —
a complete, pure description of what Spezi intends to do. It performs
the state I/O required to cache the Allium probe, but it never speaks
to the user. Any required user turn is flagged in the decision via
`clarificationNeeded` or `degradationNotices` for the dialogue stage
to render.

### Interface

```
route(parsed: ParsedInvocation, adapter: Adapter) -> RoutingDecision
```

The router reads and writes `.spezi/state.json` per `core/state.md`.
Those are the only side effects. No sub-skill is invoked here — the
decision merely names targets.

### Adapter interface (environment probes)

The router depends on a thin, agent-agnostic adapter contract. Each
runtime (Claude Code, other harnesses) supplies its own implementation
in `spezi/adapters/`. The router treats these as black boxes.

```
adapter.probeAllium() -> { available: boolean, detail?: string }
```

- `available`: true iff Allium can be invoked as a sub-skill from the
  current environment.
- `detail`: short opaque string the adapter may supply for diagnostic
  output (e.g. plugin version, path). The router passes this through
  to `check` reports but does not parse it.

Probe implementations MUST be read-only with respect to the target
environment — detection only, no install, no configuration change.

### Allium availability resolution

On every invocation (except `status`, see below), the router resolves
Allium availability through this exact sequence:

1. Load `.spezi/state.json` per `core/state.md` §Read. Hold the
   result as `state`.
2. Decide whether to probe now. Probe iff **any** of:
   - `parsed.flags.recheck` is `true`;
   - `parsed.subcommand == "check"`;
   - `state.allium` is stale per `core/state.md` §Invalidate.
3. If probing: call `adapter.probeAllium()`, then write back via
   `core/state.md` §Write with updated `allium.available`,
   `allium.checkedAt = now`. If `config.allium.forceRecheck` was
   set, clear it in the same write.
4. If not probing: use `state.allium.available` as-is.

The resolved boolean is `alliumAvailable`. A `probeDetail` (string or
null) accompanies it when a probe occurred; otherwise null.

`status` is exempt from step 2: it reports cached state only and
never triggers a probe. `status --recheck` is a contradiction — see
the `status` branch below.

### RoutingDecision shape

```json
{
  "kind": "invoke" | "check" | "status" | "clarify",
  "targets": [
    { "skill": "gherkin" | "allium", "mode": "elicit|distill|read|green|red|refactor|null" }
  ],
  "seed": "",
  "environment": {
    "alliumAvailable": false,
    "probed": false,
    "probeDetail": null
  },
  "degradationNotices": [],
  "clarificationNeeded": null,
  "diagnostic": null
}
```

Field semantics:

- `kind`:
    - `invoke` — proceed to run the listed `targets`.
    - `check` — diagnostic only; `targets` is `[]`; `diagnostic` is
      populated.
    - `status` — read-only report of cached state; `targets` is `[]`;
      `diagnostic` is populated.
    - `clarify` — do not invoke anything; the dialogue stage must ask
      the user. `clarificationNeeded` is populated.
- `targets`: ordered, deduplicated. `gherkin` precedes `allium` when
  both are present. `mode` may be `null` when no mode flag was set
  but a target was explicit — sub-skills that need a mode will prompt
  for one themselves; the router does not invent one.
- `seed`: verbatim `parsed.seed`. The router does not edit it.
- `environment.probed`: true iff step 3 above ran during this call.
- `degradationNotices`: short human-readable strings describing any
  degradation applied (e.g. Allium unavailable). Empty when nothing
  degraded. The dialogue stage decides whether/how to surface them.
- `clarificationNeeded`: `null` unless `kind == "clarify"`, then a
  structured object `{ reason, choices }` (see *Clarifications*).
- `diagnostic`: `null` for `invoke`/`clarify`; populated for `check`
  and `status` (see their branches).

### Decision tree

The router selects a branch in priority order. Once a branch matches,
later branches are not considered.

**A. Subcommand `check`** — diagnostic, never invokes sub-skills.
  Always probes Allium (step 3 above is forced). Returns
  `kind: "check"`, `targets: []`, and a `diagnostic` payload:
    ```json
    {
      "allium": { "available": true|false, "detail": "..." },
      "gherkin": { "available": true },
      "state": { "path": "<project-root>/.spezi/state.json",
                 "schemaVersion": 1,
                 "updatedAt": "..." },
      "config": { ... }  // passthrough of state.config
    }
    ```
  `--allium`, `--gherkin`, `--both` scope the diagnostic to the
  requested targets. Mode flags are not applicable — they are
  recorded as `degradationNotices` entries of the form
  `"--<flag> ignored under \`check\`"`. A non-empty `seed` under
  `check` is similarly ignored with a notice.

**B. Subcommand `status`** — read-only report, never probes.
  Returns `kind: "status"`, `targets: []`, and a `diagnostic` built
  from the cached `state` only. `environment.probed` is always
  `false`. Freshness of the allium entry is computed and reported
  (`fresh | stale | missing`). `status --recheck` is a contradiction:
  the router does **not** probe; it records a
  `degradationNotices` entry
  `"--recheck has no effect under \`status\`; use \`check\` to
  re-probe"` and proceeds with the cached report. Target and mode
  flags are ignored with analogous notices.

**C. Main branch** — no subcommand. Determine targets and mode:

  1. **Count mode flags set.** Mode flags = {elicit, distill, read,
     green, red, refactor}.
     - `0 mode flags`: `mode = null`.
     - `1 mode flag`: `mode = <that flag>`.
     - `≥2 mode flags`: `kind = clarify`, reason
       `"multiple-mode"`, choices = the set of mode flags that were
       passed. Skip the rest.

  2. **Resolve target set.**
     - `--both` → `{gherkin, allium}`.
     - `--allium` and `--gherkin` both set (without `--both`) →
       `{gherkin, allium}`; record a
       `degradationNotices` entry `"--allium and --gherkin are
       equivalent to --both"`.
     - `--allium` only → `{allium}`.
     - `--gherkin` only → `{gherkin}`.
     - No target flag:
       - mode ∈ {elicit, distill, read} → `{gherkin}`.
       - mode ∈ {green, red, refactor} → `{gherkin, allium}`.
       - mode == null:
         - `seed` non-empty → `kind = clarify`, reason
           `"no-mode-no-target"`, choices =
           `[gherkin+elicit, gherkin+distill, gherkin+read,
             both+red, both+green, both+refactor]`.
         - `seed` empty and no flags set at all → `kind = clarify`,
           reason `"empty-invocation"`, choices = same as above
           plus `check`, `status`.

  3. **Apply Allium availability** (see *Graceful degradation*).

  4. If the target set survives as non-empty, emit
     `kind: "invoke"` with ordered targets, `mode` attached to each
     target entry, and `seed` passed through.

### Graceful degradation

Applied after the main branch resolves a target set, before emitting
the final decision.

- If `alliumAvailable == true`: no-op.
- If `alliumAvailable == false` and `allium` is **not** in the target
  set: no-op.
- If `alliumAvailable == false` and `allium` **is** in the target set:
    - If `allium` was *implicit* (added by default mapping for mode
      ∈ {green, red, refactor}): drop `allium` from the set, append a
      `degradationNotices` entry
      `"Allium unavailable; proceeding with Gherkin only"`. If
      `gherkin` remains, emit `kind: "invoke"`. If the set is now
      empty, escalate to
      `kind: "clarify"`, reason `"allium-unavailable-fallback"`,
      choices = `[gherkin-only, abort, install-allium]`.
    - If `allium` was *explicit* (via `--allium` or `--both`):
      escalate to `kind: "clarify"`, reason
      `"allium-unavailable-explicit"`, choices = `[gherkin-only,
      abort, install-allium]`. Do not silently drop the explicit
      request — the user named it.

Under `check`, degradation does not apply: `check` reports what is,
rather than making a plan. An unavailable Allium simply lands in the
diagnostic as `available: false`.

### The `--recheck` modifier

`--recheck` is orthogonal to every non-`status` branch. It forces
step 3 of *Allium availability resolution* to run, regardless of
cache freshness, and updates the persisted state with the fresh
result. After resolution, routing proceeds exactly as it would
without `--recheck` — the flag is not itself a target or a mode.
Combined with `status` it is a contradiction (see branch B).

### The `check` flow (summary)

1. Parse.
2. Force-probe Allium via adapter; write state.
3. Build the diagnostic object from: probe result, cached
   `config`, resolved state path, schema version, timestamps.
4. Scope to requested targets if `--allium` / `--gherkin` / `--both`
   was passed; otherwise include everything.
5. Return `kind: "check"` with notices for any ignored mode/seed
   input. No sub-skill is invoked.

### Clarifications

`clarificationNeeded` structure:

```json
{
  "reason": "multiple-mode" |
            "no-mode-no-target" |
            "empty-invocation" |
            "allium-unavailable-fallback" |
            "allium-unavailable-explicit",
  "choices": ["<opaque-id-1>", "<opaque-id-2>", ...],
  "context": { ... }  // optional structured payload the dialogue
                      // stage may render (e.g. the offending flags)
}
```

The router does not draft prose, does not pick a default, and does
not apply a timeout. The dialogue stage (later) is responsible for
turning this into a user turn and, once resolved, re-invoking the
router with an amended `ParsedInvocation`.

### Routing test cases

Each case shows the invocation on the left and the salient fields of
the resulting `RoutingDecision` on the right. Omitted fields hold
zero/default values. `A=available`, `A=unavailable` abbreviates the
assumed Allium availability for that case.

**A available**

| Invocation | Decision |
|------------|----------|
| `--elicit add SSO` | `kind=invoke`, `targets=[gherkin(elicit)]`, `seed="add SSO"` |
| `--distill` | `kind=invoke`, `targets=[gherkin(distill)]` |
| `--read` | `kind=invoke`, `targets=[gherkin(read)]` |
| `--green checkout happy path` | `kind=invoke`, `targets=[gherkin(green), allium(green)]`, `seed="checkout happy path"` |
| `--red empty cart` | `kind=invoke`, `targets=[gherkin(red), allium(red)]`, `seed="empty cart"` |
| `--refactor rate limiter` | `kind=invoke`, `targets=[gherkin(refactor), allium(refactor)]` |
| `--allium seed` | `kind=invoke`, `targets=[allium(null)]`, `seed="seed"` |
| `--gherkin seed` | `kind=invoke`, `targets=[gherkin(null)]` |
| `--both --elicit x` | `kind=invoke`, `targets=[gherkin(elicit), allium(elicit)]` |
| `--allium --gherkin x` | `kind=invoke`, `targets=[gherkin(null), allium(null)]`, `degradationNotices=["--allium and --gherkin are equivalent to --both"]` |

**A unavailable**

| Invocation | Decision |
|------------|----------|
| `--green x` (implicit allium) | `kind=invoke`, `targets=[gherkin(green)]`, `degradationNotices=["Allium unavailable; proceeding with Gherkin only"]` |
| `--allium x` (explicit allium) | `kind=clarify`, `reason="allium-unavailable-explicit"` |
| `--both --elicit x` | `kind=clarify`, `reason="allium-unavailable-explicit"` |
| `--elicit x` (no target flag) | `kind=invoke`, `targets=[gherkin(elicit)]` (no degradation — Allium never entered the set) |

**Mode and target ambiguity**

| Invocation | Decision |
|------------|----------|
| `--elicit --distill x` | `kind=clarify`, `reason="multiple-mode"`, `choices=["elicit","distill"]` |
| `add SSO to login` | `kind=clarify`, `reason="no-mode-no-target"` |
| `` (empty) | `kind=clarify`, `reason="empty-invocation"` |
| `--red --green` | `kind=clarify`, `reason="multiple-mode"`, `choices=["red","green"]` |

**`--recheck` modifier**

| Invocation | Decision |
|------------|----------|
| `--recheck --elicit x` | `kind=invoke`, `targets=[gherkin(elicit)]`, `environment.probed=true` |
| `--recheck` (no mode/seed) | `kind=clarify`, `reason="empty-invocation"`, `environment.probed=true` |

**`check` subcommand**

| Invocation | Decision |
|------------|----------|
| `check` | `kind=check`, `environment.probed=true`, `diagnostic` covers gherkin and allium |
| `check --allium` | `kind=check`, diagnostic scoped to allium |
| `check --both --recheck` | `kind=check`, scoped to both, probed |
| `check --elicit x` | `kind=check`, `degradationNotices=["--elicit ignored under \`check\`", "seed ignored under \`check\`"]` |

**`status` subcommand**

| Invocation | Decision |
|------------|----------|
| `status` | `kind=status`, `environment.probed=false`, diagnostic from cache |
| `status --recheck` | `kind=status`, `environment.probed=false`, `degradationNotices=["--recheck has no effect under \`status\`; use \`check\` to re-probe"]` |
| `status --allium` | `kind=status`, scoped to allium in diagnostic, notice for unused flag semantics if any |

**Unknown flags (carried from parser)**

The router does not reject unknowns. It appends one `degradationNotices`
entry per unknown token of the form
`"unknown flag: <token> (ignored)"` and then routes the rest.

| Invocation | Decision |
|------------|----------|
| `--elicit --foo x` | `kind=invoke`, `targets=[gherkin(elicit)]`, `degradationNotices=["unknown flag: --foo (ignored)"]` |

