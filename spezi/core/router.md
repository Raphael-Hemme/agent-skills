# Spezi Router

The router is Spezi's entry point. It has three stages:

1. **Argument parsing** — turn the raw invocation into a structured
   intent. (This document — step below.)
2. **Routing** — decide which sub-skill(s) to run, in which mode.
   *(Not yet implemented. Deferred to a later step.)*
3. **Dialogue** — confirm intent with the user before acting.
   *(Not yet implemented. Deferred to a later step.)*

Only step 1 is specified here. A conforming implementation must not
silently introduce step 2 or 3 behavior into the parser.

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
