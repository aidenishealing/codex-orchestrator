# Plan 003: Serialize job persistence

> **Executor instructions**: Execute and verify in order. Stop and report on a
> STOP condition. Update `plans/README.md` on completion.
>
> **Drift check (run first)**:
> `git diff --stat 37ed001..HEAD -- src/jobs.ts src/watcher.ts src/notify-hook.ts src/tmux.ts src/*test.ts`

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: MED
- **Depends on**: `plans/002-authoritative-activity-clock.md`
- **Category**: tech-debt
- **Planned at**: commit `37ed001`, 2026-07-11

## Why this matters

Parallel agents are the product, yet separate processes rewrite whole job and
index JSON files without serializing the read-modify-write transaction. Atomic
rename prevents partial bytes but not lost updates. Concurrent starts,
notifications, and completions can drop active index entries, turn counts, last
messages, or terminal status.

## Current state

- `src/jobs.ts:202-210` atomically renames an index temp file.
- `src/jobs.ts:230-239` still performs an unlocked read/change/rewrite of the
  entire shared index.
- `src/jobs.ts:294-304` is the central normalized `saveJob` path.
- `src/watcher.ts:81-109` bypasses that path and rewrites job JSON directly:

```ts
const job = JSON.parse(readFileSync(jobPath, "utf-8"));
// mutate fields
writeFileSync(jobPath, JSON.stringify(job, null, 2));
```

- `src/tmux.ts:162-191` embeds another whole-file job/index update in the exit
  hook. Preserve crash recovery and append-oriented history.

## Commands you will need

| Purpose | Command | Expected on success |
| --- | --- | --- |
| Persistence tests | `bun test src/jobs-json.test.ts src/jobs-cleanup.test.ts` | all pass |
| Full tests | `bun test` | all pass |
| Build | `bun build src/cli.ts --outdir /tmp/codex-orchestrator-build --target node` | exit 0 |

## Scope

**In scope**:

- a new `src/job-store.ts` plus `src/job-store.test.ts`, if useful
- `src/jobs.ts`
- `src/watcher.ts`
- `src/notify-hook.ts`
- `src/tmux.ts` completion-hook integration
- directly related tests

**Out of scope**:

- Database or network storage
- Deleting or rewriting old job/session history
- Public JSON schema changes
- Decomposing unrelated CLI/job presentation code

## Git workflow

- Suggested branch: `advisor/003-serialize-job-store`
- Commit style: `Serialize job persistence`.
- Do not push.

## Steps

### Step 1: Extract the storage boundary

Centralize path validation, reads, normalized atomic writes, and field updates.
Every job mutation must merge against the latest record inside this boundary.
Use an OS-appropriate lock or conflict-aware retry that works across Bun
processes; an atomic rename alone is insufficient.

**Verify**: unit tests reject traversal IDs and preserve unrelated fields during
an update.

### Step 2: Serialize shared-index transactions

Guard the entire read-modify-write of `index.json`. Make starts/completions of
different jobs commutative so no entry disappears.

**Verify**: spawn multiple test processes that add/remove distinct job IDs;
after all exit, every expected active entry exists and every terminal entry is
absent.

### Step 3: Route watcher and completion writes through the store

Replace direct JSON rewrites in `watcher.ts` and the inline completion hook.
Keep the hook small; it may invoke a dedicated Bun module/command instead of
duplicating persistence logic as a string.

**Verify**: race a turn-complete update with terminal completion. The final job
retains correct terminal state, incremented turn count, last message, usage,
and index removal.

## Test plan

- Multi-process index-add/index-remove test.
- Same-job turn/completion race test.
- Crash/leftover-temp recovery test.
- Existing cleanup and JSON contract tests remain green.

## Done criteria

- [ ] All job/index writes use one validated persistence boundary.
- [ ] No direct job JSON rewrite remains in `watcher.ts` or inline shell code.
- [ ] Concurrent updates preserve all expected fields and entries.
- [ ] Full tests and `/tmp` build pass.
- [ ] No user history was deleted or rewritten for migration.
- [ ] Only in-scope files and `plans/README.md` changed.

## STOP conditions

- The proposed lock is not cross-process on supported macOS and Linux.
- The change requires a database or destructive migration.
- A race test remains nondeterministic after two correction attempts.

## Maintenance notes

Future lifecycle fields belong in the job-store update contract. Reviewers
should look specifically for read-modify-write code outside that module.

