# Plan 005: Resolve codebase maps safely and canonically

> **Executor instructions**: Follow and verify each step. Stop on a STOP
> condition. Update `plans/README.md` when complete.
>
> **Drift check (run first)**:
> `git diff --stat 37ed001..HEAD -- src/files.ts src/prompt-context.ts src/*test.ts docs/CODEBASE_MAP.md docs/agent/CODEBASE_MAP.md README.md memory/project-memory.md`

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: security
- **Planned at**: commit `37ed001`, 2026-07-11

## Why this matters

`--map` currently selects a known-stale generated map instead of the maintained
agent map, and it follows repository-controlled symlinks without checking where
they resolve. An untrusted checkout can therefore inject an outside local file
into a delegated prompt, while this repo itself supplies obsolete architecture
and model defaults to its agents.

## Current state

- `src/files.ts:16-30` searches only these paths, in order:

```ts
resolve(cwd, "docs/CODEBASE_MAP.md"),
resolve(cwd, "CODEBASE_MAP.md"),
resolve(cwd, "docs/ARCHITECTURE.md"),
```

  It uses `readFileSync()` directly, which follows symlinks.
- `memory/project-memory.md:22` says `docs/agent/CODEBASE_MAP.md` is the concise
  current map and `docs/CODEBASE_MAP.md` may lag.
- The selected legacy map claims eight files and `gpt-5.4`; current source has
  additional state/parser/watcher modules and `src/config.ts:5` uses `gpt-5.5`.
- `src/prompt-context.test.ts` is the existing pattern for map discovery tests.

## Commands you will need

| Purpose | Command | Expected on success |
| --- | --- | --- |
| Map tests | `bun test src/prompt-context.test.ts` | all pass |
| Full tests | `bun test` | all pass |
| Build | `bun build src/cli.ts --outdir /tmp/codex-orchestrator-build --target node` | exit 0 |

## Scope

**In scope**:

- `src/files.ts`
- `src/prompt-context.ts` only if result/error shape must change
- `src/prompt-context.test.ts` or a new `src/files.test.ts`
- `docs/CODEBASE_MAP.md`, `docs/agent/CODEBASE_MAP.md`, `README.md`, and
  `memory/project-memory.md` only as required to establish one truthful contract

**Out of scope**:

- Reading any real file outside a temp test directory
- Following an escaping symlink through an opt-out
- Regenerating unrelated agent bootstrap docs
- Changing prompt accounting or task-prompt contents

## Git workflow

- Suggested branch: `advisor/005-safe-map-resolution`
- Commit style: `Resolve codebase maps safely`.
- Do not push.

## Steps

### Step 1: Enforce real-path containment

Resolve the real working-directory path and candidate real path. Accept only a
regular file contained beneath the working directory. Reject symlinks that
escape, and cap map size before reading it. Return a clear non-secret error or
safe no-map result.

**Verify**: temp tests cover a normal map, an internal symlink if intentionally
supported, an escaping symlink, non-file target, and oversized file. The escape
and oversized cases are not read into prompt content.

### Step 2: Establish canonical precedence

Prefer `docs/agent/CODEBASE_MAP.md` when present because this repo designates it
current. Document a deterministic fallback order for projects that only have a
Cartographer map or `docs/ARCHITECTURE.md`.

**Verify**: when both maps exist, the maintained agent map path/content wins;
when it is absent, the documented fallback wins.

### Step 3: Remove map-truth drift

Either refresh the legacy map or clearly mark it generated/non-canonical so no
runtime path treats it as current. Align README and project memory with the
actual precedence.

**Verify**: search docs for contradictory canonical-map claims and obsolete
default model labels in the chosen map; expected result is no contradiction.

## Test plan

- All fixtures must live under a temp directory and contain synthetic text.
- Assert selected path as well as content.
- Assert escaping targets are never returned.
- Keep prompt token-accounting tests green.

## Done criteria

- [ ] Map reads cannot escape the selected working tree.
- [ ] Oversized/non-regular maps are rejected safely.
- [ ] `docs/agent/CODEBASE_MAP.md` wins when both maps exist.
- [ ] Docs state one canonical precedence contract.
- [ ] Full tests and `/tmp` build pass.
- [ ] Only in-scope files and `plans/README.md` changed.

## STOP conditions

- A legitimate supported workflow requires an external map symlink and no
  explicit safe opt-in can preserve it.
- The canonical-map decision conflicts with a newer repo decision document.
- Verification fails twice.

## Maintenance notes

Map files are untrusted input because their contents cross into agent prompts.
Future lookup locations need both precedence tests and real-path containment.

