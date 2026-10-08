# GAM improvement program

This directory is the executable backlog that takes GAM from a polished OSS tool to an enterprise-grade product. It was derived from the internal enterprise-grade audit of 2026-07-06 (findings F1-F22, recommendations R1-R72; the audit itself is an internal document and is not part of the repo). Every task here is self-contained: an agent or contributor can execute it without reading the audit.

## Layout

| File                                    | Purpose                                                                       |
| --------------------------------------- | ----------------------------------------------------------------------------- |
| `README.md`                             | This file: how the program works.                                             |
| `BACKLOG.md`                            | Master index: all tasks, decisions, waves, estimates, live status.            |
| `tasks/wave-0.md` ... `tasks/wave-5.md` | Detailed task cards (context, steps, acceptance, verification).               |
| `LEDGER.md`                             | Append-only execution log. Every completed task gets an evidence entry.       |
| `PROMPT.md`                             | Prompt A (trigger agent implementation) and Prompt B (human review protocol). |

## How a wave runs

1. Stephen triggers Prompt A from `PROMPT.md`, naming a wave (or accepting the default: lowest incomplete wave).
2. The agent works on a dedicated branch `auto/improve-gam-w<N>`, executes tasks in dependency order, and updates `BACKLOG.md` status plus `LEDGER.md` evidence after each task.
3. Every task must pass the quality gate before its commit: `pnpm lint`, `pnpm vite:build` (type-checks), `pnpm test`, and in `src-tauri/`: `cargo clippy --locked -- -D warnings`, `cargo test --lib --locked`, `cargo deny check`.
4. The agent never pushes and never touches key material. It stops at genuine forks and records them as BLOCKED in the ledger.
5. Stephen reviews with Prompt B, pushes, opens the PR, and flips statuses to `verified`.

## Status vocabulary

`todo` -> `in_progress` -> `done` (agent, gates green) -> `verified` (human review passed). Side states: `blocked(<reason>)`, `skipped(<reason>)`, `gated(D<n>)` for tasks waiting on a human decision.

## Rules that bind every task

- No task may weaken an existing gate, delete a test to go green, or add a dependency without recording the reason in the ledger.
- Anything involving secrets, certificates, or signing keys is split: the agent writes config and docs; only Stephen creates, moves, or enters key material.
- Public-repo discipline: nothing in `docs/` may reference internal infrastructure, credentials, or unpublished business plans. This backlog is written to be publishable.
- Conventional commits, one commit per task, task id in the body (for example `Implements GAM-012.`).

## Current shape

6 waves, 65 tasks, 7 human decisions, roughly 349 hours of agent work. See `BACKLOG.md` for the live index.
