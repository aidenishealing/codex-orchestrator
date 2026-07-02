# Codebase Map

Update this file when architecture changes.

## Core Paths
- `bin/codex-agent`: shell entrypoint for the installed CLI.
- `src/cli.ts`: parses commands/options and routes job, session, capture, output, cleanup, and health commands.
- `src/jobs.ts`: owns job lifecycle, persisted job JSON, artifact archival, status refresh, output access, and job cleanup.
- `src/tmux.ts`: owns tmux session creation, messaging, capture, session cleanup, and process-state checks.
- `src/state.ts`: normalizes process/turn state into orchestration status values.
- `src/prompt-context.ts`: builds prompt/file context and token accounting for delegated jobs.
- `src/session-parser.ts`: locates and parses Codex session files for job evidence.
- `src/output-cleaner.ts`: strips ANSI/Codex TUI noise from captured output.
- `src/usage-parser.ts`: parses Codex token-usage output.
- `src/watcher.ts`: handles turn-complete signal files for `await-turn`.
- `tools/eslint-plugin-agent-guardrails.cjs`: local architecture guardrail plugin.

## Critical Flows
- Start job: `bin/codex-agent` -> `src/cli.ts` -> `startJob()` in `src/jobs.ts` -> `createSession()` in `src/tmux.ts`.
- Send input: `src/cli.ts` -> `sendToJob()` in `src/jobs.ts` -> `sendMessage()` in `src/tmux.ts`.
- Capture output: `src/cli.ts` -> `getJobOutput()` / `getJobFullOutput()` in `src/jobs.ts` -> tmux capture helpers -> optional `cleanTerminalOutput()`.
- Status view: `src/cli.ts` -> `refreshJobStatus()` / JSON builders in `src/jobs.ts` -> `deriveJobView()` in `src/state.ts`.
- Cleanup: `src/cli.ts` -> `cleanupOldJobs()` in `src/jobs.ts` -> archive job artifacts plus tmux cleanup helpers.

## Ownership
- CLI command contract: `src/cli.ts`.
- Job persistence/state: `src/jobs.ts` and `src/state.ts`.
- Tmux process boundary: `src/tmux.ts`.
- Prompt/file context: `src/prompt-context.ts` and `src/files.ts`.

## Notes
- This repo is a TypeScript/Bun CLI, not a web app with a separate server/client/lib split.
- Keep this file short and factual.
- Prefer file paths and call chains over prose.
