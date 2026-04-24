# Spezi State Management

Runtime state for Spezi lives in `.spezi/state.json` at the project root
(the working directory where Spezi is invoked). It caches environment
probes and holds user-supplied config overrides so Spezi does not
re-detect its surroundings on every invocation.

This document is the logic spec. It is agent-agnostic: any skill runtime
that implements file read, file write, and ISO-8601 timestamp generation
can conform to it. No implementation is prescribed — only the behavior.

## Location

- Path: `<project-root>/.spezi/state.json`
- "Project root" is the current working directory when Spezi is invoked.
  If Spezi is invoked inside a git repository, the repository root is
  preferred over the current working directory when they differ.
- The directory `.spezi/` is created lazily on first write.

## Schema

The file is a single JSON object. All fields are optional on read
(missing fields resolve to their defaults); writers produce the full
shape below.

```json
{
  "schemaVersion": 1,
  "allium": {
    "available": false,
    "checkedAt": "2026-04-24T10:00:00Z"
  },
  "config": {},
  "updatedAt": "2026-04-24T10:00:00Z"
}
```

Field semantics:

- `schemaVersion` (integer, required on write): the version of this
  schema the writer understood. Current value: `1`.
- `allium.available` (boolean): whether the Allium plugin was detected
  as installed and callable in the current environment at the time of
  the probe.
- `allium.checkedAt` (ISO 8601 UTC timestamp): when the Allium probe
  was last performed. Used to compute cache freshness.
- `config` (object): user-supplied overrides. Keys and nested shape are
  intentionally open — routers and sub-skills may read known keys and
  ignore the rest. Unknown keys must be preserved on rewrite.
- `updatedAt` (ISO 8601 UTC timestamp): when the file was last written.
  Distinct from `allium.checkedAt` — rewriting `config` updates
  `updatedAt` but not the allium probe timestamp.

Writers must not invent additional top-level fields without bumping
`schemaVersion`. Readers must tolerate and preserve unknown fields so
newer writers do not lose data when an older reader round-trips.

## Read

1. Resolve the state path (see *Location*).
2. If the file does not exist, return the default state:
   ```json
   { "schemaVersion": 1, "allium": null, "config": {}, "updatedAt": null }
   ```
   This is an in-memory default only — do **not** create the file as a
   side effect of reading.
3. If the file exists but does not parse as JSON, or `schemaVersion` is
   missing / greater than the reader's known version, treat the file as
   unreadable: return the default state and surface a single concise
   notice to the user explaining that cached state was ignored. Do not
   delete the file — a later writer will overwrite it cleanly, and
   preserving it lets the user inspect the problem.
4. If `schemaVersion` is lower than the reader's version, migrate
   in-memory by filling defaults for any fields the older writer would
   not have produced. Do not rewrite the file as a side effect of
   reading; the next legitimate write upgrades it on disk.

## Write

1. Compose the full object: known fields plus any unknown fields
   preserved from the prior read. Set `updatedAt` to the current UTC
   timestamp. Set `schemaVersion` to the current version.
2. Ensure `.spezi/` is gitignored (see *Gitignore handling*) **before**
   creating the directory, so the cache never lands in a staged commit.
3. Ensure the directory `<project-root>/.spezi/` exists.
4. Write atomically: serialize to `<project-root>/.spezi/state.json.tmp`,
   then rename to `state.json`. This guarantees readers never observe a
   partial file.
5. A write that only updates `config` must leave `allium.checkedAt` and
   `allium.available` unchanged. A write that records a probe result
   updates both `allium.*` fields and `updatedAt`.

## Invalidate

Invalidation is per-field, not whole-file. The two scenarios:

- **Allium probe is stale.** The allium entry is considered stale when
  any of the following is true:
    1. `allium` is `null` or missing.
    2. `allium.checkedAt` is absent or unparseable.
    3. `now - allium.checkedAt > 7 days`.
    4. The user explicitly asked to re-check (e.g. "recheck allium",
       "allium isn't working", or any signal the router interprets as
       a request to re-probe).
    5. `config.allium.forceRecheck` is `true` — in which case the
       router clears that flag as part of the next write.
  When stale, the router re-probes Allium, then writes the fresh
  result. The TTL (7 days) is a default; a user-supplied
  `config.allium.ttlDays` overrides it.

- **Schema version mismatch.** Treated as unreadable on read (see
  *Read* step 3). The next write replaces the file.

Invalidation never deletes the file silently. If the user explicitly
asks Spezi to "reset state" or equivalent, the router confirms in one
short exchange, then overwrites with the default object.

## Gitignore handling

`.spezi/` holds machine-specific cache. It must never be committed.

Before the first write of `state.json` in a given project:

1. Determine whether the project root is inside a git repository
   (presence of a `.git` directory or `git rev-parse` success). If not,
   skip this section entirely — there is nothing to ignore into.
2. Read `<repo-root>/.gitignore` if it exists.
3. If the file contains a line matching `.spezi/` or `.spezi` (exact
   match, ignoring surrounding whitespace and trailing slash), do
   nothing.
4. Otherwise, append — preserving existing content and trailing newline
   conventions — a block of the form:
   ```
   # Spezi runtime cache
   .spezi/
   ```
   If `.gitignore` does not exist, create it with that block as its
   sole content.
5. Proceed with the write.

The gitignore step is idempotent and must not produce duplicate entries
on repeated invocations. Spezi does not stage, commit, or otherwise
touch git beyond reading repo root and editing `.gitignore`.
