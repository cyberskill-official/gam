# Wave 0 - truth and closable holes (P0)

Gate for every task: `pnpm lint && pnpm vite:build && pnpm test` green, and in `src-tauri/`: `cargo clippy --locked -- -D warnings && cargo test --lib --locked && cargo deny check`. One conventional commit per task with the task id in the body.

---

## GAM-001 Whitelist git config scope as an enum

Why: `git_service.rs` builds the config flag with `format!("--{}", scope)` in `add_alias`, `update_alias`, and `delete_alias`, and `get_aliases` accepts free strings. The scope arrives raw over IPC, so `file=/some/path`, `system`, `worktree`, or `blob:...` become valid single git arguments: arbitrary config-file writes from a compromised webview.

Priority P0. Estimate 2 h. Deps: none.

Files: `src-tauri/src/git_service.rs`, `src-tauri/src/commands.rs`.

Steps:

1. Add `pub enum Scope { Global, Local }` with `TryFrom<&str>` accepting exactly `"global"` and `"local"` (and `"all"` only where listing is legal), returning a descriptive error otherwise.
2. Change `add_alias`, `update_alias`, `delete_alias`, `get_aliases` signatures to parse into `Scope` at the command boundary before any argument construction; derive the `--global`/`--local` flag from the enum, never from the input string.
3. Keep the IPC string parameter for compatibility; conversion failure returns the standard error shape.
4. Add unit tests: `file=/tmp/x`, `system`, `worktree`, `blob:HEAD:.gitconfig`, empty string, `Global` (case), all rejected; `global`, `local` accepted.

Acceptance:

- No `format!("--{}", ...)` remains on user-controlled input anywhere in `src-tauri/`.
- New rejection tests pass; existing integration tests still pass.

Verify: `grep -rn 'format!("--' src-tauri/src/` returns nothing scope-derived; `cargo test --lib --locked`.

---

## GAM-002 README and docs truth pass

Why: the README promises an auto-backup of `~/.gitconfig` before every write and copy-based usage tracking; neither exists in the code. It also points Homebrew at `zintaen/tap` while release.yml uses `cyberskill-official/tap`, and Node engine statements disagree (package.json >=22, RELEASING.md >=24, CI 24.12.0). For a tool that edits git configs, documentation accuracy is a security property.

Priority P0. Estimate 3 h. Deps: none. Related later work: GAM-014 implements the real backup.

Files: `docs/README.md`, `docs/MANUAL.md`, `docs/RELEASING.md`, `docs/CODEBASE.md`, `package.json`.

Steps:

1. Remove or mark "planned (tracked as GAM-014)" the Auto-Backup feature row; remove the "usage is tracked when you copy" sentence (ranking reads shell history with consent, nothing else).
2. Fix the Homebrew instruction to `cyberskill-official/tap/gam` and note it goes live only after the tap exists (D7).
3. Align Node engines: set package.json `engines.node` to `>=24.0.0` to match CI and RELEASING.md, or lower both docs to 22 - pick one and make all three agree (CI currently uses 24.12.0; prefer 24).
4. Sweep remaining `zintaen` references in `docs/` where they mean the org (keep true history in CHANGELOG and the Buy Me a Coffee link only if Stephen wants it; flag it in the ledger for his call).
5. Re-verify every feature row in the README against the code; adjust wording where the code does less (or more) than claimed.

Acceptance:

- No documented feature lacks an implementation; no stale namespace or engine contradiction remains.
- `grep -rn "zintaen" docs/` output only contains deliberate historical references listed in the ledger entry.

Verify: manual diff review; `grep -rniE "auto-backup|backs up" docs/` shows only the planned-feature note.

---

## GAM-003 SECURITY.md, SUPPORT.md, issue templates, CODEOWNERS

Why: there is no vulnerability disclosure channel, no support statement, no issue templates, and no ownership map. These are the cheapest professionalism signals an enterprise checks first.

Priority P0. Estimate 3 h. Deps: none.

Files: `SECURITY.md` (repo root), `docs/SUPPORT.md`, `.github/ISSUE_TEMPLATE/bug_report.yml`, `.github/ISSUE_TEMPLATE/task.yml`, `.github/ISSUE_TEMPLATE/config.yml`, `.github/CODEOWNERS`.

Steps:

1. SECURITY.md: report privately via GitHub private vulnerability reporting or security@cyberskill.world; supported versions = latest release; acknowledgment target 72 h, fix target 30 d for high severity; no bounty program; credit policy.
2. SUPPORT.md: community support via GitHub issues/discussions, no SLA for the free tier; link SECURITY.md for vulnerabilities.
3. Issue forms (YAML): bug (version, OS, git version, repro steps, logs hint) and feature; config.yml disables blank issues and links SECURITY.md for security reports.
4. CODEOWNERS: `* @cyberskill-official/maintainers` or Stephen's handle; `/.github/ @<owner>` and `/src-tauri/ @<owner>` explicitly.

Acceptance: files render correctly on GitHub; security policy tab picks up SECURITY.md.

Verify: GitHub UI check after push (ledger note); YAML lint clean (`pnpm lint` covers repo lint config if applicable, otherwise actionlint/manual).

---

## GAM-004 CodeQL and gitleaks workflows

Why: Rust is gated by clippy, cargo-audit, and cargo-deny, but the TypeScript half has lint only - no SAST. There is also no history-wide secret scanning in CI (the updater key incident makes this worth automating).

Priority P0. Estimate 3 h. Deps: none.

Files: `.github/workflows/codeql.yml`, `.github/workflows/gitleaks.yml`.

Steps:

1. CodeQL: `github/codeql-action` (init/analyze) pinned by SHA, language `javascript-typescript`, on pull_request to main + weekly schedule; default queries.
2. gitleaks: `gitleaks/gitleaks-action` pinned by SHA, full-history scan on schedule (weekly) and diff scan on pull_request; add `.gitleaks.toml` allowlisting the updater public key (it is public by design) so the scan stays signal-only.
3. Both workflows `permissions:` minimal (`security-events: write` for CodeQL, `contents: read`).

Acceptance: both workflows pass on the branch; gitleaks full-history run is clean (or findings triaged in the ledger).

Verify: `gh run list` after push; local `gitleaks detect --source .` if the binary is available.

---

## GAM-005 Repo-settings hardening runbook

Why: rulesets, tag protection, secret scanning with push protection, private vulnerability reporting, and the Dependency Graph are GitHub settings, not files - an agent cannot flip them, but it can make them a 10-minute checklist instead of tribal knowledge. Tag protection matters most: releases are cut from `v*` tags, so tag creation must be maintainer-only or the release pipeline is the weakest link.

Priority P0. Estimate 2 h. Deps: none.

Files: `docs/improvement/runbooks/repo-settings.md` (new).

Steps:

1. Write the checklist with exact `gh api` commands where the API supports it: branch ruleset on main (require PR, require the four check.yml jobs, no force push), tag ruleset on `v*` (restrict creation), enable secret scanning + push protection, enable private vulnerability reporting, enable Dependency Graph (unblocks the currently non-blocking dependency-review job), require 2FA on the org.
2. Mark each item with its verification command (`gh api repos/cyberskill-official/gam/rulesets` etc.).
3. Cross-link from SECURITY.md maintainers section.

Acceptance: runbook covers all six settings with verification commands; Stephen can complete it in one sitting and tick each line.

Verify: dry-read of the runbook against GitHub's current settings UI naming; `gh api` commands syntax-checked with `--help` locally where possible.
