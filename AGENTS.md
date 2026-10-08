# AGENTS.md — gam

gam is a Tauri desktop application: a Vite + React/TypeScript front end in `src/` and a Rust back end in `src-tauri/`, managed with pnpm.

**Status (2026-10-08):** legacy CyberOS v1 has been removed from this repository. The SDD kit from CyberOS is to be adopted here next (CyberOS decision DEC-106); its arrival protocol will replace this file. Until then:

- Open work is the improvement program in `docs/tasks/improvement/` (BACKLOG.md indexes GAM-001 … GAM-065; it will be carried into SDD tasks at adoption).
- Changes go through pull requests opened by Stephen; never push to `main`, deploy, publish a release or touch signing keys without his explicit instruction.

Commands (from `package.json`):

| Purpose | Command |
|---|---|
| Run the app | `pnpm dev` |
| Unit tests | `pnpm test` |
| End-to-end tests | `pnpm test:e2e` |
| Lint | `pnpm lint` |
| Front-end build (type-checks) | `pnpm vite:build` |
| Rust checks | `cd src-tauri && cargo clippy --locked -- -D warnings && cargo test --lib --locked` |
