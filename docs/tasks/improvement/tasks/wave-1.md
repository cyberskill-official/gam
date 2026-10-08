# Wave 1 - core correctness and hardening (P1)

Gate per task: same as wave 0 (lint, vite build, vitest, clippy -D warnings, cargo test, cargo deny).

---

## GAM-010 exec_git timeout and non-interactive git env

Why: exec_git in git_service.rs has no timeout. A known repo on a dead network mount, or any git call that blocks, freezes the app. Git must also never be able to prompt.

Priority P1. Estimate 4 h. Deps: none.

Files: src-tauri/src/git_service.rs.

Steps:

1. Spawn git with a 10 s timeout: spawn, poll with try_wait in a small loop (or wait_timeout crate), kill on expiry, return a "git timed out after 10s in <dir>" error.
2. Always set env GIT_TERMINAL_PROMPT=0 and GIT_ASKPASS=echo on the Command so credential prompts fail fast instead of hanging.
3. Unit test the timeout path with a fake slow binary (test-only script) or gate it behind an integration test using `git -c core.fsmonitor=false` plus a blocking stub on PATH.
4. Surface the timeout error distinctly so the UI can suggest removing the dead repo path (pairs with GAM-016).

Acceptance:

- No git subprocess can block longer than the timeout; prompt-based hangs impossible.
- Existing integration tests unaffected.

Verify: cargo test --lib --locked; manual: add a known repo path on an unreachable mount and confirm the error surfaces instead of a hang.

---

## GAM-011 Null-separated alias parsing

Why: parse_alias_output splits on lines, but git config values may contain newlines. A multiline alias corrupts the parse and can smear one alias into several.

Priority P1. Estimate 4 h. Deps: none.

Files: src-tauri/src/git_service.rs.

Steps:

1. Switch listing to `git config <scope> -z --get-regexp ^alias\.` and parse NUL-separated records (key\nvalue\0 per git docs: key and value separated by newline, records by NUL).
2. Rewrite parse_alias_output for the -z format; keep the old function only if tests need comparison, otherwise delete.
3. Regression tests: alias whose command contains \n, quotes, unicode, leading !; empty output; single alias.
4. Confirm exit-code-1 empty-result handling still works with -z.

Acceptance: a multiline alias round-trips (add via git CLI, read via GAM) without corruption; all parser tests green.

Verify: new unit tests + integration test adding a multiline alias in a temp repo.

---

## GAM-012 Atomic JSON persistence + corruption surfacing

Why: settings_service, group_service, and known_repos_service persist with plain fs::write and silently ignore load errors. A crash mid-write corrupts the file; next launch silently resets to defaults. Users lose groups without any message.

Priority P1. Estimate 6 h. Deps: none.

Files: src-tauri/src/settings_service.rs, group_service.rs, known_repos_service.rs, new src-tauri/src/persist.rs; frontend toast on load warning.

Steps:

1. Add persist.rs with save_json_atomic(path, value): serialize, write to <name>.tmp in the same dir, fsync file, rename over target; and load_json(path) -> Result distinguishing NotFound (fresh start) from Corrupt(err).
2. Use it in all three services. On Corrupt: rename the bad file to <name>.corrupt-<timestamp>, start from defaults, and record a warning the frontend can fetch (extend an existing query or add get_startup_warnings command).
3. Set restrictive permissions on create (0o600 unix).
4. Tests: corrupt-file quarantine, tmp-file cleanup, roundtrip.

Acceptance: kill -9 during save can no longer lose the previous state; corruption is visible in the UI, not silent.

Verify: cargo test --lib --locked (new persist tests); manual corruption test documented in ledger.

---

## GAM-013 update_alias: set-new-then-unset-old with rollback

Why: the rename path unsets the old alias before setting the new one. If the set fails (permissions, invalid value), the user's alias is already gone.

Priority P1. Estimate 2 h. Deps: GAM-011.

Files: src-tauri/src/git_service.rs.

Steps:

1. Reorder: validate, set alias.<new>, then unset alias.<old>; if the unset fails after a successful set, attempt to remove alias.<new> and return the original error (best-effort rollback, both errors reported).
2. When old == new, keep the single-set path.
3. Integration tests: rename success; rename where set fails (invalid target file permissions in a temp repo) leaves the old alias intact.

Acceptance: no sequence of failures deletes an alias without its replacement existing.

Verify: cargo test --lib --locked, new rename-failure test.

---

## GAM-014 Real gitconfig backup + restore

Why: the README promised backups that never existed (removed from docs in GAM-002). Build the real thing: it is genuine protection for a tool whose whole job is mutating gitconfigs.

Priority P1. Estimate 10 h. Deps: GAM-012.

Files: new src-tauri/src/backup_service.rs, git_service.rs hooks, commands.rs, src/components/settings/DataPanel.tsx, tests.

Steps:

1. backup_service: before any mutating git config call, copy the exact target config file (global: git config --global --list --show-origin to resolve the real path, honoring GIT_CONFIG_GLOBAL/XDG; local: <repo>/.git/config) into <app-data>/backups/<sha1-of-path>/<utc-timestamp>.gitconfig; keep the last 10 per target, prune older.
2. Skip backup when the target file does not exist yet (first global alias) but record intent.
3. Commands: list_backups(target), restore_backup(target, id) with a confirm flow; restore writes atomically (GAM-012 helper) after backing up the current state first.
4. DataPanel UI: per-target backup list with timestamps, restore button behind ConfirmDialog.
5. Tests: rotation prunes to 10, restore round-trip in a temp repo, backup precedes every mutating path (add/update/delete).
6. Re-add the README feature row, now true (this closes the loop opened in GAM-002).

Acceptance: every mutating operation is preceded by a backup; restore works from the UI; rotation bounded; docs updated.

Verify: cargo tests; manual UI restore in dev; ledger screenshot optional.

---

## GAM-015 Remove panic paths; deny panic-family lints

Why: lib.rs calls std::env::current_dir().unwrap() while resolving the CLI path argument - a real crash if the cwd was deleted. Nothing stops new unwraps from landing in lib code.

Priority P1. Estimate 3 h. Deps: none.

Files: src-tauri/src/lib.rs, src-tauri/Cargo.toml (lints section) or lib.rs attributes.

Steps:

1. Replace the current_dir().unwrap() with unwrap_or_else falling back to skipping the "." argument, with an eprintln (later: log) note.
2. Add [lints.clippy] unwrap_used = "deny", expect_used = "deny", panic = "deny" for the crate; allow them in #[cfg(test)] modules.
3. Fix every violation the lint surfaces (settings/group/known-repos services use unwrap_or_else already; sweep remaining).

Acceptance: clippy clean with the new denies; deleted-cwd launch no longer panics.

Verify: cd src-tauri && cargo clippy --locked -- -D warnings.

---

## GAM-016 Known-repos hygiene

Why: known-repos.json grows forever, paths are stored un-canonicalized (duplicates via symlinks or trailing slashes), KnownReposService::remove is dead code, and there is no way to clear app data. "All scopes" scans every stored path.

Priority P1. Estimate 6 h. Deps: GAM-012.

Files: src-tauri/src/known_repos_service.rs, commands.rs, lib.rs (register), src/components/settings/PrivacyPanel.tsx and DataPanel.tsx, src/tauri-bridge.ts, types.

Steps:

1. Canonicalize paths in add(); dedupe existing entries on load; only store paths that contain a .git dir or are bare repos (git rev-parse --is-inside-work-tree check via exec_git).
2. New commands: list_known_repos, forget_known_repo(path), clear_app_data(kind: backups|history-cache|known-repos|all) with confirm in UI.
3. PrivacyPanel: show what is stored and where (paths list), with forget buttons; DataPanel: clear-all-data.
4. Tests: canonicalization dedupe, forget persists, clear removes files.

Acceptance: users can see and delete everything GAM stores; no duplicate scans.

Verify: cargo + vitest suites; manual check of app-data dir after clear.

---

## GAM-017 Single-instance plugin

Why: two GAM processes race on the same JSON stores (last writer wins, and GAM-012 atomicity does not serialize logical updates).

Priority P1. Estimate 2 h. Deps: none.

Files: src-tauri/Cargo.toml, src-tauri/src/lib.rs, src-tauri/capabilities/default.json.

Steps:

1. Add tauri-plugin-single-instance (desktop only), focusing the existing window when a second launch happens; forward the CLI path argument from the second invocation to the running instance (set_local_path) if present.
2. Capability entry if the plugin requires one.

Acceptance: second launch focuses the first instance and applies its path argument; no second process remains.

Verify: manual double-launch test (documented in ledger); cargo gates green.

---

## GAM-018 Async commands + bounded concurrent repo scan

Why: every command is synchronous on the IPC thread; get_aliases("all") runs one git subprocess per known repo serially. With GAM-010's timeout the worst case is bounded but still additive. tokio is already a dependency and unused.

Priority P1. Estimate 8 h. Deps: GAM-010.

Files: src-tauri/src/commands.rs, git_service.rs, lib.rs.

Steps:

1. Make git-touching commands async fn; run subprocess work in tauri::async_runtime::spawn_blocking (keep RwLock usage outside await points, or switch AppState locks to tokio::sync where held across awaits).
2. In the "all"/no-path listing, scan known repos concurrently with a bound of 4 (futures buffered or a semaphore), merging results in stable order.
3. Frontend unchanged (invoke is already promise-based); verify no UI regression in loading states.
4. Ranking refresh moves inside spawn_blocking too (large history files off the IPC thread).

Acceptance: UI stays responsive during a scan of many repos including one timing out; results identical to serial scan.

Verify: cargo tests; manual test with 5+ known repos, one on a slow path; clippy clean (async fns in generate_handler are supported).

---

## GAM-019 Dependency hygiene

Why: tokio was declared but unused (GAM-018 now uses it - keep only the features needed: rt, macros; add sync if used). @cyberskill/shared sits in dependencies but nothing in src/ imports it; it backs the cyberskill dev CLI only.

Priority P1. Estimate 1 h. Deps: GAM-018.

Files: src-tauri/Cargo.toml, package.json.

Steps:

1. Trim tokio features to what GAM-018 actually uses; cargo tree confirm no duplicate runtimes.
2. Move @cyberskill/shared to devDependencies; pnpm install; confirm vite build output unchanged and the cyberskill scripts still run.

Acceptance: no unused runtime dependency remains in either manifest.

Verify: pnpm vite:build + pnpm lint (cyberskill CLI proves itself); cargo build.

---

## GAM-020 Blocking supply-chain gates for npm

Why: pnpm audit and dependency-review run with continue-on-error, so they alert nobody. Rust has a license policy (cargo-deny); npm has none.

Priority P1. Estimate 3 h. Deps: none.

Files: .github/workflows/check.yml, package.json (script), optional audit allowlist file.

Steps:

1. Remove continue-on-error from pnpm audit; set --audit-level=high; document the exception process: temporary ignores live in an .audit-allowlist.json consumed by a small wrapper script (or use pnpm audit --json + jq filter) with expiry dates.
2. Dependency-review: keep continue-on-error only until the Dependency Graph is enabled (GAM-005 runbook); flip it blocking in the same PR that confirms the setting.
3. Add a license gate for npm deps (license-checker-rseidelsohn or pnpm licenses list --json + script) allowing the same license set as deny.toml; fail on unknown/copyleft.

Acceptance: a high-severity npm advisory or a GPL transitive dep fails CI; exceptions are explicit and dated.

Verify: run the new scripts locally; intentionally add a fake disallowed license entry in a test run (revert).

---

## GAM-021 Structured logging + crash-log rotation

Why: logging is bare eprintln; the panic hook appends to crash.log forever. Support needs leveled, rotating file logs; disks need bounds.

Priority P1. Estimate 4 h. Deps: none.

Files: src-tauri/Cargo.toml, lib.rs, sweep of eprintln call sites.

Steps:

1. Add tauri-plugin-log: file target in app-log dir, rotation at 1 MB keeping 5 files, level info (debug via env var GAM_LOG=debug), plus stdout in dev.
2. Replace eprintln calls with log macros; panic hook writes through the logger and also keeps the plain crash.log but rotates it (truncate-at-1MB guard).
3. Never log alias command bodies or shell-history content (assert in review; add a code comment policy note).

Acceptance: logs rotate; levels work; no sensitive payloads logged.

Verify: run dev, generate logs, confirm rotation config; grep new code for command/history interpolation.

---

## GAM-022 pnpm supply-chain cooldown

Why: npm-ecosystem worm attacks spread through freshly published versions; a minimum-age cooldown means GAM never installs a version younger than the window, giving the ecosystem time to catch compromises.

Priority P1. Estimate 1 h. Deps: none.

Files: .npmrc or pnpm-workspace.yaml (pnpm settings block).

Steps:

1. Set pnpm minimumReleaseAge to 4320 (3 days, minutes unit) with an exceptions list if any first-party package must bypass (none expected; @cyberskill/shared is pinned exact).
2. Document the setting and the override procedure in CONTRIBUTING.
3. Confirm pnpm install still resolves the current lockfile untouched.

Acceptance: fresh dependency bumps resolve only to versions older than the cooldown; lockfile unchanged by the setting itself.

Verify: pnpm install clean; pnpm config get / file review in CI logs.
