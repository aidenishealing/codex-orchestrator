# Plan 001: Protect prompts and job artifacts

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before continuing. If a
> STOP condition occurs, stop and report; do not improvise. When done, update
> the status row in `plans/README.md`.
>
> **Drift check (run first)**:
> `git diff --stat 37ed001..HEAD -- src/jobs.ts src/tmux.ts src/watcher.ts src/notify-hook.ts src/*test.ts`
> If an in-scope file changed, compare the current code with the excerpts below.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: MED
- **Depends on**: none
- **Category**: security
- **Planned at**: commit `37ed001`, 2026-07-11

## Why this matters

Jobs persist full prompts, output, and lifecycle metadata. On the audited Mac,
the jobs directory was mode `0755` and existing artifacts were mode `0644`, so
other local accounts could read delegated code context or accidental secrets.
The full prompt is also expanded into the Codex process argument. The storage
and launch boundary should treat prompts as confidential by default.

## Current state

- `src/jobs.ts:294` persists the complete `Job`, including `prompt`, without an
  explicit restrictive file mode:

```ts
writeFileSync(jobPath, JSON.stringify(normalized, null, 2));
```

- `src/tmux.ts:127` writes a second prompt copy without a mode:

```ts
fs.writeFileSync(promptFile, options.prompt);
```

- `src/tmux.ts:206` injects that file's contents into the command argument:

```ts
const codexCmd = `codex ${codexArgs} "$(cat ${shellQuote(promptFile)})"`;
```

- `src/watcher.ts:34,96,109` also writes signal/job artifacts directly.
- Preserve the existing invariant in `src/jobs.ts:83-95`: validate job IDs and
  keep artifacts under `config.jobsDir`.
- Never reproduce any real prompt or secret in tests. Use synthetic fixtures.

## Commands you will need

| Purpose | Command | Expected on success |
| --- | --- | --- |
| Targeted tests | `bun test src/jobs-cleanup.test.ts src/tmux.test.ts` | exit 0 |
| Full tests | `bun test` | all tests pass |
| Build | `bun build src/cli.ts --outdir /tmp/codex-orchestrator-build --target node` | exit 0 |

## Scope

**In scope**:

- `src/jobs.ts`
- `src/tmux.ts`
- `src/watcher.ts`
- `src/notify-hook.ts` only if the supported non-argv transport requires it
- adjacent `src/*.test.ts` files needed for this security contract

**Out of scope**:

- Reading, migrating, or copying real prompt contents
- Deleting any existing job/session/history artifact
- Changing Codex model, reasoning, sandbox, approval, or notification semantics
- Encrypting artifacts or introducing a secrets service

## Git workflow

- Suggested branch: `advisor/001-protect-prompts`
- Match the repo's imperative commit style, e.g. `Protect job artifacts`.
- Do not push or open a PR unless instructed.

## Steps

### Step 1: Centralize private artifact creation

Create one storage helper that ensures `config.jobsDir` and `.trash` are mode
`0700`, and creates job, prompt, log, signal, index-temp, and completion files
as mode `0600`. Existing files may be tightened with `chmod`; never delete or
rewrite their contents merely to change permissions.

**Verify**: add a temp-HOME test that creates each artifact and asserts directory
mode `0700` and file mode `0600`; run the targeted tests and expect exit 0.

### Step 2: Remove the full prompt from process arguments

Use a Codex-supported input path that does not expose the prompt through process
arguments. Prefer stdin if the current CLI contract accepts a prompt from stdin;
otherwise use the smallest supported file/input mechanism. Preserve immediate
non-interactive start, tmux logging, and the notify hook.

**Verify**: a fake `codex` executable must record its argv and stdin. Assert the
synthetic prompt is absent from argv and arrives exactly once through the chosen
input channel.

### Step 3: Tighten existing artifacts safely

At startup, tighten modes only inside the validated jobs directory. Skip
symlinks and anything outside that root. Failure to tighten an old artifact
must produce a clear warning without printing its contents.

**Verify**: tests cover a legacy `0644` file, a symlink, and a permission error;
the legacy file becomes `0600`, the symlink target is untouched, and no content
appears in output.

## Test plan

- Add synthetic temp-HOME tests for all artifact modes.
- Add a fake-runner launch test proving prompt absence from argv.
- Add symlink and migration-failure tests.
- Model structure on `src/jobs-cleanup.test.ts` and argument assertions on
  `src/tmux.test.ts`.

## Done criteria

- [ ] Jobs directories are created/tightened to `0700`.
- [ ] Job, prompt, log, signal, and index artifacts are `0600`.
- [ ] Synthetic prompts do not appear in the Codex process argv.
- [ ] No real user artifact content is read into tests or output.
- [ ] `bun test` passes.
- [ ] The `/tmp` build command exits 0.
- [ ] Only in-scope files and `plans/README.md` changed.

## STOP conditions

- The installed Codex CLI has no supported non-argv input path.
- Correct permissions require deleting or recreating real user history.
- A required change would weaken sandbox or approval settings.
- Any verification fails twice after a reasonable correction.

## Maintenance notes

All new job artifacts must use the same private storage helper. Reviewers should
inspect the actual spawned argv and filesystem modes, not only unit mocks.

