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
   whitespace (` `, `\t`, `\n`). No shell-style quoting. Multiple
   spaces collapse to one when the seed is rejoined.
2. **Flag classification.** A token beginning with `--` that matches
   a flags-table key sets that flag to `true`. Otherwise it goes to
   `unknown` verbatim. Flag candidates never contribute to `seed`.
3. **No flag values.** No flag takes a value. `--recheck=true` does
   not match `--recheck` and lands in `unknown`.
4. **Duplicate flags.** A flag appearing more than once stays `true`.
   Duplicates are not errors.
5. **Subcommand detection.** After flag classification, the *first*
   remaining non-flag token becomes `subcommand` iff it exactly
   matches a subcommand name; later matching tokens stay in `seed`.
   Subcommands are positional, not keywords.
6. **Seed assembly.** Remaining non-flag, non-subcommand tokens are
   joined with a single space in original order, then trimmed. Empty
   residual produces `""`, not `null`.
7. **Casing.** All matching is case-sensitive. `--Recheck` → `unknown`;
   `CHECK` / `Status` → `seed`.
8. **Empty input.** Empty or whitespace-only input yields the zero
   value: `subcommand: null`, all flags `false`, `seed: ""`, `unknown: []`.
9. **Determinism.** Parsing is stable under reordering of flag tokens;
   it is *not* stable under reordering of non-flag tokens (only the
   first non-flag token can be a subcommand).

### Test cases

One case per parser rule. Each row's omitted fields hold zero values
(`false` for flags, `null` for subcommand, `""` for seed, `[]` for
unknown).

| Rule exercised | Input | Result |
|----------------|-------|--------|
| Zero value (empty input, rule 8) | `` | — |
| Whitespace-only → zero value (rule 8) | `   ` | — |
| Singleton subcommand | `check` | `subcommand=check` |
| Singleton flag | `--elicit` | `flags.elicit=true` |
| Subcommand + flag | `check --both` | `subcommand=check`, `flags.both=true` |
| Flag order independence (rule 9) | `--recheck status` | `subcommand=status`, `flags.recheck=true` |
| Free-form seed preserved | `--elicit --allium add 2fa to the login page` | `flags.elicit=true`, `flags.allium=true`, `seed="add 2fa to the login page"` |
| TDD-phase flag + seed | `--red write a failing test for empty cart checkout` | `flags.red=true`, `seed="write a failing test for empty cart checkout"` |
| Duplicate flag (rule 4) | `--red --red` | `flags.red=true` |
| First non-flag wins subcommand (rule 5) | `foo check` | `seed="foo check"` |
| Later subcommand-name token is seed (rule 5) | `check status` | `subcommand=check`, `seed="status"` |
| Unknown flag surfaced (rule 2) | `--recheck --foo check` | `subcommand=check`, `flags.recheck=true`, `unknown=["--foo"]` |
| Flag values not supported (rule 3) | `--recheck=true` | `unknown=["--recheck=true"]` |
| Case-sensitive flag (rule 7) | `--Recheck` | `unknown=["--Recheck"]` |
| Case-sensitive non-flag (rule 7) | `CHECK` | `seed="CHECK"` |
| Whitespace normalization (rule 1) | `  --elicit   a   seed  ` | `flags.elicit=true`, `seed="a seed"` |
| Tab/newline treated as whitespace (rule 1) | `check\t--both\nlogin` | `subcommand=check`, `flags.both=true`, `seed="login"` |

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
Those are the only side effects. No sub-skill is invoked here.

### Adapter interface

The router treats each harness adapter as a black box exposing:

```
adapter.probeAllium() -> { available: boolean, detail?: string }
```

`available` is `true` iff Allium can be invoked from the current
environment. `detail` is an opaque diagnostic string (e.g. plugin
version, path) that the router passes through to `check` reports but
does not parse. Probe implementations MUST be read-only — detection
only, no install, no configuration change.

### Allium availability resolution

On every invocation (except `status`):

1. Load state per `core/state.md` §Read.
2. Probe iff **any** of: `parsed.flags.recheck == true`;
   `parsed.subcommand == "check"`; `state.allium` is stale per
   `core/state.md` §Invalidate.
3. If probing: call `adapter.probeAllium()`, write back with
   `allium.available` and `allium.checkedAt = now` per `core/state.md`
   §Write. Clear `config.allium.forceRecheck` if it was set.
4. Otherwise use `state.allium.available` as-is.

The resolved boolean is `alliumAvailable`; `probeDetail` accompanies
it when a probe occurred, else `null`. `status` reports cached state
only and never probes — `status --recheck` is a contradiction (see
branch B).

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

- `kind`: `invoke` → run the listed `targets`; `check` / `status` →
  diagnostic only, `targets: []`, `diagnostic` populated; `clarify` →
  do not invoke, `clarificationNeeded` populated.
- `targets`: ordered, deduplicated. `gherkin` precedes `allium` when
  both present. `mode` may be `null` when a target was explicit but
  no mode flag was set — sub-skills prompt for one themselves; the
  router does not invent one.
- `seed`: verbatim `parsed.seed`.
- `environment.probed`: `true` iff the probe step ran.
- `degradationNotices`: short human-readable strings for any applied
  degradation; empty when none. The dialogue stage renders them.
- `clarificationNeeded`: structured `{ reason, choices }` (see
  *Clarifications*) when `kind == "clarify"`, else `null`.
- `diagnostic`: populated for `check` and `status`; `null` otherwise.

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

**B. Subcommand `status`** — read-only report, never probes. Returns
  `kind: "status"`, `targets: []`, diagnostic built from cached state
  only, `environment.probed = false`. Allium-entry freshness is
  reported as `fresh | stale | missing`. `status --recheck` is a
  contradiction: no probe runs; a `degradationNotices` entry
  `"--recheck has no effect under \`status\`; use \`check\` to
  re-probe"` is recorded and the cached report proceeds. Target and
  mode flags are similarly ignored with analogous notices.

**C. Main branch** — no subcommand. Mode flags = {elicit, distill,
  read, green, red, refactor}:

  1. **Mode.** 0 set → `mode = null`. 1 set → `mode = <that flag>`.
     ≥2 set → `kind = clarify`, reason `"multiple-mode"`, choices =
     the passed mode flags; skip the rest.
  2. **Target set.** `--both` → `{gherkin, allium}`. `--allium` +
     `--gherkin` (without `--both`) → `{gherkin, allium}` plus a
     `"--allium and --gherkin are equivalent to --both"` notice.
     `--allium` only → `{allium}`. `--gherkin` only → `{gherkin}`.
     No target flag: mode ∈ {elicit, distill, read} → `{gherkin}`;
     mode ∈ {green, red, refactor} → `{gherkin, allium}`; mode ==
     null with non-empty seed → `kind = clarify`, reason
     `"no-mode-no-target"`, choices `[gherkin+elicit, gherkin+distill,
     gherkin+read, both+red, both+green, both+refactor]`; mode ==
     null with empty seed and no flags → `kind = clarify`, reason
     `"empty-invocation"`, choices = same set plus `check`, `status`.
  3. **Apply Allium availability** (see *Graceful degradation*).
  4. **Emit.** If the target set survives non-empty, emit
     `kind: "invoke"` with ordered targets, `mode` attached to each,
     and `seed` passed through.

### Graceful degradation

Applied after the main branch resolves a target set, before the final
decision. No-op if `alliumAvailable == true` or if `allium` is not in
the target set. Otherwise:

| Allium was... | Outcome |
|---------------|---------|
| Implicit (default mapping for `green`/`red`/`refactor`) | Drop `allium` from the set; append `"Allium unavailable; proceeding with Gherkin only"` to `degradationNotices`. If `gherkin` remains, emit `kind: "invoke"`; if the set is now empty, escalate to `kind: "clarify"`, reason `"allium-unavailable-fallback"`, choices `[gherkin-only, abort, install-allium]`. |
| Explicit (via `--allium` or `--both`) | Escalate to `kind: "clarify"`, reason `"allium-unavailable-explicit"`, choices `[gherkin-only, abort, install-allium]`. Never silently drop an explicit request — the user named it. |

Under `check`, degradation does not apply: `check` reports what is,
rather than making a plan. Unavailable Allium lands in the
diagnostic as `available: false`.

### The `--recheck` modifier

`--recheck` forces the probe step of availability resolution
regardless of cache freshness, and is orthogonal to every non-`status`
branch — it is not itself a target or a mode. Combined with `status`
it is a contradiction (see branch B).

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

One case per routing branch and per degradation path. `A=available`
vs `A=unavailable` names the assumed Allium probe outcome. Omitted
`RoutingDecision` fields hold zero/default values.

| Branch exercised | Invocation | Decision (A available unless noted) |
|------------------|------------|-------------------------------------|
| Default mapping: elicit → gherkin only | `--elicit add SSO` | `kind=invoke`, `targets=[gherkin(elicit)]`, `seed="add SSO"` |
| Default mapping: TDD phase → gherkin+allium | `--red empty cart` | `kind=invoke`, `targets=[gherkin(red), allium(red)]`, `seed="empty cart"` |
| Explicit `--allium` only | `--allium seed` | `kind=invoke`, `targets=[allium(null)]`, `seed="seed"` |
| Explicit `--both` composes with mode | `--both --elicit x` | `kind=invoke`, `targets=[gherkin(elicit), allium(elicit)]` |
| `--allium` + `--gherkin` ≡ `--both` (with notice) | `--allium --gherkin x` | `kind=invoke`, `targets=[gherkin(null), allium(null)]`, `degradationNotices=["--allium and --gherkin are equivalent to --both"]` |
| Implicit allium degrades silently-with-notice | `--green x` (A=unavailable) | `kind=invoke`, `targets=[gherkin(green)]`, `degradationNotices=["Allium unavailable; proceeding with Gherkin only"]` |
| Explicit allium unavailable → clarify | `--allium x` (A=unavailable) | `kind=clarify`, `reason="allium-unavailable-explicit"` |
| Multiple mode flags → clarify | `--elicit --distill x` | `kind=clarify`, `reason="multiple-mode"`, `choices=["elicit","distill"]` |
| No mode, non-empty seed → clarify | `add SSO to login` | `kind=clarify`, `reason="no-mode-no-target"` |
| Empty invocation → clarify | `` | `kind=clarify`, `reason="empty-invocation"` |
| `--recheck` is orthogonal (forces probe) | `--recheck --elicit x` | `kind=invoke`, `targets=[gherkin(elicit)]`, `environment.probed=true` |
| `check` always probes | `check` | `kind=check`, `environment.probed=true`, `diagnostic` covers gherkin+allium |
| `check` ignores mode/seed with notice | `check --elicit x` | `kind=check`, `degradationNotices=["--elicit ignored under \`check\`", "seed ignored under \`check\`"]` |
| `status` never probes | `status` | `kind=status`, `environment.probed=false`, diagnostic from cache |
| `status --recheck` is a contradiction | `status --recheck` | `kind=status`, `environment.probed=false`, `degradationNotices=["--recheck has no effect under \`status\`; use \`check\` to re-probe"]` |
| Unknown flag → notice, routing proceeds | `--elicit --foo x` | `kind=invoke`, `targets=[gherkin(elicit)]`, `degradationNotices=["unknown flag: --foo (ignored)"]` |

The router does not reject unknown flags. Each unknown token from the
parser produces one `degradationNotices` entry of the form
`"unknown flag: <token> (ignored)"`; routing then proceeds as if the
token were absent.

