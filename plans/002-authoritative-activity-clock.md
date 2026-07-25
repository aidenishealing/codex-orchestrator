# Plan 002: Use one authoritative activity clock

> **Executor instructions**: Follow each step and verification gate. Stop on a
> STOP condition; do not improvise. Update `plans/README.md` when complete.
>
> **Drift check (run first)**:
> `git diff --stat 37ed001..HEAD -- src/state.ts src/jobs.ts src/state.test.ts src/jobs-json.test.ts src/cli-contract.test.ts`

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: MED
- **Depends on**: none
- **Category**: bug
- **Planned at**: commit `37ed001`, 2026-07-11

## Why this matters

State derivation ignores live log activity while the JSON field
`last_activity_at` prefers it. A job can therefore report `STALE` alongside a
recent activity timestamp, causing parent agents and `await-turn` to abandon an
active worker. The same split now produces six failing tests because active
fixtures hard-code May 2026 dates.

## Current state

- `src/jobs.ts:356-373` correctly uses log mtime for inactivity timeout.
- `src/jobs.ts:426-429` reports log mtime as outward activity.
- `src/jobs.ts:545-559` derives orchestration state before applying that log
  timestamp:

```ts
const derived = deriveJobView(job, { staleAfterMs: getStaleAfterMs() });
// ...
last_activity_at: getLastActivityAt(job, derived.lastActivityAt),
```

- `src/state.ts:213-223` compares only persisted lifecycle timestamps with
  `Date.now()`.
- `src/state.test.ts:113-140` is the existing exemplar for injecting `nowMs`.

## Commands you will need

| Purpose | Command | Expected on success |
| --- | --- | --- |
| State tests | `bun test src/state.test.ts src/jobs-json.test.ts src/cli-contract.test.ts` | all pass |
| Full tests | `bun test` | all pass; no six stale-related failures |
| Build | `bun build src/cli.ts --outdir /tmp/codex-orchestrator-build --target node` | exit 0 |

## Scope

**In scope**:

- `src/state.ts`
- `src/jobs.ts`
- `src/state.test.ts`
- `src/jobs-json.test.ts`
- `src/cli-contract.test.ts`

**Out of scope**:

- Changing the configured 60-minute timeout
- Changing terminal-state precedence
- Adding a daemon, heartbeat service, or new public schema field

## Git workflow

- Suggested branch: `advisor/002-activity-clock`
- Imperative commit style: `Unify job activity state`.
- Do not push.

## Steps

### Step 1: Define one activity value

Compute the authoritative last activity once per job from log mtime when it
exists, otherwise the latest persisted lifecycle timestamp. Pass that exact
value into staleness derivation and return it as `last_activity_at`.

**Verify**: add a unit case with ancient lifecycle timestamps and a current log
mtime; expect non-STALE state and the same current `last_activity_at`.

### Step 2: Make clocks deterministic in contracts

Inject `nowMs` at the state/view boundary or generate active fixture times from
a controlled test clock. Preserve an explicit old timestamp case for `STALE`.
Do not merely replace May 2026 with a later hard-coded date.

**Verify**: state, JSON, and CLI contract tests pass regardless of wall clock.

### Step 3: Cover idle and active semantics

Add cases for fresh-log WORKING, fresh-log WAITING, old-log STALE, and terminal
COMPLETED (never stale). Assert `actions.recommended_next` agrees with state.

**Verify**: targeted tests pass, then `bun test` passes.

## Test plan

- Use synthetic temp job/log files only.
- Assert state, `last_activity_at`, and recommended action as one contract.
- Retain direct `deriveJobView(..., { nowMs, staleAfterMs })` coverage.

## Done criteria

- [ ] One timestamp drives both staleness and `last_activity_at`.
- [ ] Fresh log activity prevents STALE classification.
- [ ] Truly inactive jobs still become STALE.
- [ ] All six previously failing tests pass.
- [ ] Full tests and `/tmp` build pass.
- [ ] Only in-scope files and `plans/README.md` changed.

## STOP conditions

- The intended semantics require a new external heartbeat source.
- A change would make terminal jobs stale.
- Verification fails twice.

## Maintenance notes

Any future activity source must feed this same boundary; do not add another
display-only timestamp that state derivation cannot see.

