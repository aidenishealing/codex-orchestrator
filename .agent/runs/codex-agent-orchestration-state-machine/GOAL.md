# Codex Agent Orchestration State Machine

Goal ID: `codex-agent-orchestration-state-machine`
Started: 2026-05-28T02:08:14Z
Parent goal: none
Mode: full
Ledger path: `.agent/runs/codex-agent-orchestration-state-machine/`

## Objective

Implement Oracle-backed codex-agent state machine, JSON output contract, await-turn semantics, prompt/context accounting, token usage parsing, sandbox truthfulness, and safe cleanup.

## Non-Negotiable User Requirements

Original request:

> Okay. Lets get all 6 done. Create a master PRD, scope out the work then execute it to completion with $goal please

Acceptance proofs:

- [done] Master PRD exists at `docs/prds/codex-agent-orchestration-state-machine.md`.
- [done] Work is scoped into owned implementation lanes before product edits.
- [done] `$goal` ledger is current in `implementation-notes.html` and `events.jsonl`.
- [done] State derivation and terminal normalization are implemented and tested.
- [done] JSON output contract is implemented and tested.
- [done] State-aware `await-turn` is implemented and tested.
- [done] Prompt/context accounting is implemented and tested.
- [done] Codex token usage parsing is implemented, persisted, exposed, and tested.
- [done] Sandbox truthfulness and safe cleanup are implemented, remediated, reviewed, and tested.
- [done] Final validation passes and final review blockers were remediated.

## Goal Mode Coupling

When creating or updating the matching `/goal`, include this ledger pointer in the goal objective:

`Maintain the agent-owned ledger at /Users/saint/Dev/tools/codex-agent/.agent/runs/codex-agent-orchestration-state-machine/ and keep implementation-notes.html current at checkpoints, before compaction, and before final handoff.`

## Finishing Criteria

- [done] `bun test` passes with 38 tests.
- [done] Targeted state/contract/accounting/usage/safety tests pass.
- [done] A real smoke path proves `WAITING` after a completed turn.
- [done] Keep `implementation-notes.html` current with status, decisions, tradeoffs, changes, validation, and next action.
- [done] Link large proof artifacts from `evidence/` when they are too bulky for the HTML notes.

## Escape Hatch

Pause, ask Saint, or mark a scoped item `[blocked]` / `[incomplete]` if:
- validation contradicts the goal
- the goal requires a scope change
- the agent is looping without measurable progress
- the next step risks deleting or rewriting durable memory
- the PRD and actual repo disagree
- the ledger itself contaminates validation
