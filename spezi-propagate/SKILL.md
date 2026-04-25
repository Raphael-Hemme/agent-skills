---
name: spezi-propagate
description: Generate tests from a `.feature.md`. Hand off to Allium when the spec links to an Allium artifact or the feature is logic-heavy and Allium is available; otherwise detect the repo's BDD test framework, confirm with the user, and generate Gherkin step definitions / scenario tests. Use when the user wants to propagate tests, generate test files from a spec, write tests for a behavioural specification, produce property-based tests, or check test coverage against scenarios.
---

# spezi-propagate

Turn a `.feature.md` into runnable tests. Two paths: **hand off to Allium** (logic-heavy specs with formal rules) or **generate framework-native tests** (Gherkin step definitions + scenarios in the repo's BDD framework). Decide which per spec; confirm the user before generating.

## Output

- New test files in the repo's test layout (path depends on path: `tests/features/`, `features/`, `cypress/integration/`, etc. — confirm with user).
- Decision record: `specs/gherkin/decisions/<YYYY-MM-DD>-<slug>-propagate[-<n>].md` — base + Propagate section per `spezi/reference/decision-record.template.md`.
- Updated `.feature.md` `allium:` block if Allium handoff occurred (status: `pending` → `active` after Allium confirms).

## Boundaries

- Generates tests; doesn't run them. Running tests is the user's harness or a follow-up `/spezi-weed --update-code` flow.
- Doesn't modify production code. Generated tests reference fixtures and helpers; if those don't exist, scaffold them as TODOs in the test file.
- Doesn't write a new `.feature.md`. If the spec needs work, exit and run `/spezi-tend` or `/spezi-elicit`.

## Step 1 — Resolve the seed

Seed → one feature file (slug, path, or free-form). Confirm with one line:

```
Propagating tests for <slug>, <N> scenarios (<H> happy, <E> edge, <I> invariants). Continue? (yes / abort)
```

## Step 2 — Decide handoff vs. native

Probe Allium availability per `spezi/reference/allium-linking.md` §Availability probe (read-only — never invoke Allium during the probe). Then:

| Condition | Action |
|---|---|
| Spec has an `allium:` entry with `kind: tests` and Allium is available | Propose Allium handoff. |
| Spec has no Allium link, Allium is available, and the spec is **logic-heavy** (≥2 invariants, or ≥1 cross-scenario state-machine spanning 3+ scenarios, or temporal triggers) | Propose Allium handoff with a one-line rationale. |
| Otherwise | Propose framework-native test generation. |

Always confirm before proceeding:

```
This spec is logic-heavy (3 invariants, state machine across 5 scenarios) and Allium is available.
Hand off to Allium for richer property-based testing? (recommended)

Reply `allium` / `native` / `abort`.
```

If the user picks `native` despite a recommendation for Allium, do not push back twice. Their call.

## Step 3a — Allium handoff path

Invoke Allium with `{ mode: "post-gherkin", seed: <slug>, featureFile: <path> }` per `spezi/reference/allium-linking.md` §Hook. Pre-prompt notice if any `stale`, missing-file, or mismatched-slug `allium:` entries exist on the spec.

After Allium returns control:
- Record the Allium-side artifact paths in the `.feature.md` `allium:` block (`status: active`, `linkedAt: now`).
- Persist the decision record with a `## Allium handoff` section listing the artifacts.
- Skip Step 3b.

## Step 3b — Framework-native generation path

### Detect the test framework

Walk the repo for one of these markers, in order:

| Language | Probe path | Framework hint |
|---|---|---|
| JS/TS | `package.json` → `dependencies` / `devDependencies` | `cucumber`, `@cucumber/cucumber`, `cypress-cucumber-preprocessor`, `playwright-bdd` |
| Python | `pyproject.toml`, `setup.cfg`, `requirements*.txt` | `pytest-bdd`, `behave` |
| Ruby | `Gemfile` | `cucumber`, `cucumber-rails` |
| Go | `go.mod` | `github.com/cucumber/godog` |
| Java/Kotlin | `pom.xml`, `build.gradle*` | `io.cucumber:cucumber-java`, `io.cucumber:cucumber-kotlin` |
| Rust | `Cargo.toml` | `cucumber` |
| C# | `*.csproj`, `packages.config` | `SpecFlow`, `Reqnroll` |

If exactly one framework is found and it's a Gherkin BDD runner, propose it. If multiple are found (e.g. cucumber-js + playwright-bdd), surface them and ask. If none is found, ask the user to name the framework or to install one before continuing.

### Confirm

```
Detected: pytest-bdd in pyproject.toml.
Test layout: tests/features/<feature>.feature + tests/step_defs/<feature>_steps.py?

Reply `yes` / `change framework: <name>` / `change layout: <path>` / `abort`.
```

Don't generate without explicit confirmation.

### Generate

For each `## Scenario:` block in the `.feature.md`:
- Emit a corresponding `Scenario:` block in the framework's `.feature` file (same Gherkin syntax — most Gherkin frameworks read identical syntax).
- Emit a step-definition stub (one function per Given/When/Then step) in the language's idiomatic style. Each stub raises `NotImplementedError` (or framework equivalent) so it fails loudly. Add a docstring quoting the spec step verbatim.
- For invariants: emit one assertion helper per invariant, called from the step that needs it. Add a TODO if the invariant cannot be expressed without a runtime hook.

For ambiguous mappings (a step mentions "the user" but the test framework needs a fixture), emit a TODO and a flag to the user list rather than guessing.

### Flag list before writing

```
Flagged for your attention (Propagate — <slug>):
1. [scenario: Happy path — SSO login] step "When the assertion is POSTed to /sso/callback" needs a test client fixture; none found in tests/conftest.py
2. [invariant: audit log] no runtime hook found for audit log assertion; emit TODO?

Reply `1: stub`, `1: skip`, `1: defer`. Silence = stub all (with TODOs).
```

After resolving, write all test files and the record.

## Step 4 — Update the spec's `allium:` block (Allium path only)

Only on the Allium-handoff path. Native-test generation does not touch `allium:` (those tests aren't Allium artifacts).

## Reusing existing tests

Before generating, search the test layout for existing tests that already cover spec scenarios. Patterns:

- Test names that match `## Scenario:` titles.
- Steps that quote the spec verbatim.
- Coverage by step text (a `Given a user with an Okta IdP assertion` step elsewhere in the suite).

If found, list them in the flag list as `1: [reuse] tests/features/sso.feature::Scenario: SSO login already covers this`. User picks `accept` to skip generation for that scenario or `regenerate` to overwrite.

## Generator awareness

If the framework supports property-based testing (Hypothesis for pytest-bdd, fast-check for cucumber-js, ScalaCheck for Cucumber-JVM), offer to generate property-based assertions for invariants:

```
Detected Hypothesis available. Generate property-based tests for invariants? (yes / no)
```

Default to `no` (additive, not core).

## Limitations

Generated tests are starting points. They will fail until step-definition stubs are filled in. The skill never claims a test is passing — it claims a test scaffold matches the spec's shape.
