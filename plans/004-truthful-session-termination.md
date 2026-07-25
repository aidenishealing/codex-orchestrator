# Plan 004: Make session termination truthful

> **Executor instructions**: Follow the plan and verification gates exactly.
> Stop on a STOP condition. Update `plans/README.md` when complete.
>
> **Drift check (run first)**:
> `git diff --stat 37ed001..HEAD -- src/jobs.ts src/tmux.ts src/*test.ts`

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: MED
- **Depends on**: `plans/003-serialize-job-persistence.md`
- **Category**: bug
- **Planned at**: commit `37ed001`, 2026-07-11

## Why this matters

The CLI can report an agent killed or deleted even when tmux termination fails,
then archive the metadata needed to control the still-running process. It also
marks any vanished running tmux session completed without confirmed successful
exit. Lifecycle status must reflect proven process outcomes, especially because
workers may continue modifying a workspace.

## Current state

- `src/jobs.ts:700-712` ignores `killSession()` failure before archiving.
- `src/jobs.ts:777-790` marks a job killed and returns success without confirming
  session absence.
- `src/jobs.ts:947-959` treats a missing session as successful completion:

```ts
if (!sessionExists(job.tmuxSession)) {
  job.status = "completed";
  job.completedAt = new Date().toISOString();
  // load whatever log exists, then save
}
```

- `src/tmux.ts:324-338` already exposes a truthful boolean termination result.
- The PRD requires terminal `COMPLETED`, `FAILED`, and `CANCELLED` states to be
  trustworthy; preserve its vocabulary.

## Commands you will need

| Purpose | Command | Expected on success |
| --- | --- | --- |
| Lifecycle tests | `bun test src/jobs-cleanup.test.ts src/tmux.test.ts src/state.test.ts` | all pass |
| Full tests | `bun test` | all pass |
| Build | `bun build src/cli.ts --outdir /tmp/codex-orchestrator-build --target node` | exit 0 |

## Scope

**In scope**:

- `src/jobs.ts`
- `src/tmux.ts`
- lifecycle/cleanup tests and a small fake process/session harness

**Out of scope**:

- Killing unmanaged tmux sessions
- Deleting Codex sessions, transcripts, or history
- Changing the public state vocabulary
- Adding force-kill behavior beyond the existing explicit kill command

## Git workflow

- Suggested branch: `advisor/004-truthful-termination`
- Commit style: `Make session termination truthful`.
- Do not push.

## Steps

### Step 1: Confirm kill outcomes

For running jobs, require `killSession()` success and a follow-up absence check
before saving a terminal killed state or archiving metadata. On failure, keep
the job and index intact, record a recoverable error, and return failure.

**Verify**: fake tmux failure leaves metadata present and both kill/delete APIs
report failure; success removes the session and only then updates/archives.

### Step 2: Distinguish clean exit from unexplained disappearance

When a running session vanishes, trust an already-persisted completion-hook
outcome. If no confirmed exit status exists, do not infer success from absence;
record a failed/unknown terminal outcome with the available log evidence.

**Verify**: tests cover exit 0, nonzero exit, manual disappearance, and kill.
Only exit 0 becomes `COMPLETED`.

### Step 3: Preserve recovery evidence

Ensure error/status output tells the operator what was confirmed, what remains
controllable, and the safe next action. Never remove the only job ID/log after
failed termination.

**Verify**: JSON and human status contract tests assert the terminal state,
error, and recommended action for each outcome.

## Test plan

- Mock/fake tmux kill success and failure.
- Simulate completion-hook exit 0/nonzero.
- Simulate session disappearance without hook evidence.
- Assert no history outside the job artifact root is touched.

## Done criteria

- [ ] Kill/delete success requires confirmed session absence.
- [ ] Failed termination preserves metadata and returns failure.
- [ ] Unexplained disappearance never becomes `COMPLETED`.
- [ ] Exit 0 and nonzero are persisted distinctly.
- [ ] Full tests and `/tmp` build pass.
- [ ] Only in-scope files and `plans/README.md` changed.

## STOP conditions

- The completion hook cannot persist an exit result through the job store.
- Supporting a platform requires deleting external session/history data.
- Tests cannot distinguish unmanaged from managed tmux sessions.

## Maintenance notes

Session absence is evidence that a process is gone, not evidence of success.
Keep that distinction explicit in future cleanup and recovery paths.

