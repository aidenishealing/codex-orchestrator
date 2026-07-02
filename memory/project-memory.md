# Project Memory

Use this file for durable project context that should survive session resets.
Treat this as append-oriented working memory for the software factory.

## Identity
- Project: codex-orchestrator
- Repo path: /Users/HP/dev/10_active/codex-orchestrator
- Product surface: Bun/TypeScript CLI for launching and tracking Codex agent jobs through tmux.
- Source ticket / message: Deploy MAX Agents hallucination remediation on 2026-07-02.
- Delivery target: Local CLI repo and GitHub remote.

## Tool Slots
- Source control: GitHub remote `https://github.com/aidenishealing/codex-orchestrator.git`.
- Deploy: Local CLI build through `npm run build`.
- Data: Local job artifacts under the configured Codex-agent jobs directory.
- Observability: CLI status/capture/output commands plus persisted job JSON and tmux logs.
- Security gate: Not a web app; use code review, TypeScript/Bun build checks, and path validation for job artifacts.
- Identity and secrets: No repo secrets required; do not hardcode credentials in job prompts, logs, or config.

## Decisions
- 2026-07-02: Treat `docs/agent/CODEBASE_MAP.md` as the concise current map. The older `docs/CODEBASE_MAP.md` remains a deeper generated reference but may lag newer files.

## Append-Only Log
- Timestamp:
  - Agent:
  - Change:
  - Evidence:
- Timestamp: 2026-07-02 12:09 EDT
  - Agent: Codex
  - Change: Replaced the generated web-app placeholder in `docs/agent/CODEBASE_MAP.md` with the actual Bun/TypeScript CLI structure and call paths, and filled this project-memory identity from verified repo files/remotes.
  - Evidence: Verified current files under `src/`, `bin/codex-agent`, `package.json`, and the older `docs/CODEBASE_MAP.md`; removed the false generated web-app map entries.

## Active Workstreams
- Hallucination-report remediation for generated agent docs.

## Open Questions
- None for the documentation path correction.

## Risks
- Worker jobs launched through this CLI can capture sensitive prompt/output content; keep artifacts local unless intentionally shared.

## References
- Related tickets: Deploy MAX Agents hallucination audit, row R055.
- Related docs: `docs/agent/CODEBASE_MAP.md`, `docs/CODEBASE_MAP.md`, `README.md`.
