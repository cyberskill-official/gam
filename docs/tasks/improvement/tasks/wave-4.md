# Wave 4 - architecture, performance, reach (P2-P3)

Gate per task: standard.

---

## GAM-070 Watch gitconfig files + manual refresh

Why: aliases edited in a terminal while GAM is open do not appear until an implicit refetch; there is no refresh affordance at all.

Priority P2. Estimate 6 h. Deps: GAM-018.

Files: src-tauri (notify crate watcher service), lib.rs, event emit to frontend, useAliases.ts, Toolbar refresh button.

Steps:

1. Watch the resolved global config path and each known repo's .git/config with the notify crate (debounced 500 ms); on change emit aliases-changed to the webview.
2. Frontend listens and refetches; add a manual refresh button with a spinner state as the fallback affordance.
3. Watcher lifecycle: rebuild on known-repos changes; drop on exit; failure to watch degrades silently to manual refresh.
4. Tests: watcher unit (temp file touch triggers debounce once); frontend event handling.

Acceptance: git config --global alias.zz whatever in a terminal appears in GAM within ~1 s; refresh button always works.

Verify: manual terminal-edit test; suites green.

---

## GAM-071 Event-driven ranking refresh

Why: history reparses on a 5 s TTL inside get_aliases; wasteful and still stale-feeling.

Priority P3. Estimate 2 h. Deps: GAM-036.

Files: ranking_service.rs, lib.rs/frontend focus event.

Steps:

1. Refresh history cache on window focus events and after alias mutations; raise TTL to 60 s as a backstop.
2. Keep the consent gate exactly as is.

Acceptance: identical scores with far fewer parses (log-verified).

Verify: debug logs show refresh points; tests unchanged.

---

## GAM-072 Typed IPC via tauri-specta

Why: I_GitAlias and friends are hand-mirrored from Rust; drift is silent until runtime.

Priority P2. Estimate 6 h. Deps: GAM-032 (error shape stable first).

Files: src-tauri (specta derives + build step), generated src/bindings.ts, tauri-bridge.ts consuming it.

Steps:

1. Add specta + tauri-specta; derive Type on the IPC structs; generate bindings in dev/build (checked-in file, CI check that it is current).
2. Replace hand-written invoke signatures in tauri-bridge.ts with the generated client (keep the I_AppAPI facade so components do not churn).
3. CI step: regenerate and git diff --exit-code.

Acceptance: a Rust struct change without regenerated bindings fails CI; runtime shapes guaranteed.

Verify: intentional field rename dry run (revert); suites green.

---

## GAM-073 Extract gam-core crate

Why: services are Tauri-free already in spirit (plain structs + std); making that literal enables the CLI (GAM-094), isolated fuzzing, and reuse.

Priority P2. Estimate 10 h. Deps: GAM-018.

Files: src-tauri/Cargo.toml becomes a workspace: crates/gam-core (git_service, ranking, groups, settings, known-repos, backup, persist, error), src-tauri depends on it.

Steps:

1. Create the workspace; move service modules to gam-core with no tauri dependency (dialog usage stays in the shell; dirs crate stays in core behind a config-dir parameter).
2. App-data dir passed into core constructors (kills the 4x hardcoded dir string; pairs with GAM-080).
3. Keep tauri.conf/bundling untouched; cargo-deny + clippy cover the workspace.
4. All existing tests move with their modules; CI paths updated.

Acceptance: src-tauri compiles as a thin shell; gam-core builds and tests standalone with no tauri in its tree (cargo tree proof).

Verify: cargo tree -p gam-core | grep -c tauri == 0; full gates green.

---

## GAM-074 Version-sync guard

Why: three files carry the version; releases bypassing scripts/release.js could drift them.

Priority P3. Estimate 1 h. Deps: none.

Files: .github/workflows/release.yml (prepare job step) or check.yml.

Steps:

1. Script compares package.json, tauri.conf.json, Cargo.toml versions and, on tag builds, the tag itself; mismatch fails fast before any build minutes burn.

Acceptance: mismatched tag dies in prepare with a clear message.

Verify: unit-run the script with a forced mismatch locally.

---

## GAM-075 Renovate tuning

Why: default config creates PR noise; unattended patch bumps are safe here given the gate.

Priority P3. Estimate 1 h. Deps: none.

Files: renovate.json.

Steps:

1. Weekly schedule, grouped minor devDependencies, automerge patch-level devDependencies on green checks, lockFileMaintenance monthly, security alerts immediate.
2. Keep runtime deps manual-review.

Acceptance: config validates (renovate-config-validator); policy documented inline.

Verify: validator run; first scheduled PR observed post-merge.

---

## GAM-076 Accessibility gate

Why: no a11y checks exist; glassmorphism themes carry contrast risk; enterprises ask for a11y posture (VPAT eventually).

Priority P2. Estimate 8 h. Deps: GAM-030.

Files: playwright a11y spec (axe-core), component fixes, docs/ACCESSIBILITY.md stub.

Steps:

1. Add @axe-core/playwright scans of the main screen, form modal, settings, in default + one light theme; fail on serious/critical.
2. Keyboard-only e2e: full CRUD without a mouse (tab order, focus trap in modals, Escape behavior - shortcuts already exist).
3. Fix findings: focus rings, aria labels on icon buttons, dialog roles, contrast tokens where cheap; theme-wide contrast sweep can be follow-up-listed in the doc.
4. ACCESSIBILITY.md: current conformance statement + known gaps.

Acceptance: axe passes at serious+ on scanned surfaces; keyboard path proven in CI.

Verify: pnpm test:e2e with the new specs.

---

## GAM-077 Theme visual regression

Why: 10 theme CSS files, zero automated eyes; refactors regress themes silently.

Priority P3. Estimate 4 h. Deps: GAM-030.

Files: playwright screenshot spec + checked-in baselines per theme (dashboard view, fixed viewport, animations disabled).

Steps:

1. Deterministic rendering: freeze animations (prefers-reduced-motion + CSS override), fixed seed data via a mock bridge state.
2. toHaveScreenshot per theme with a small maxDiffPixelRatio; update-snapshot workflow documented.

Acceptance: a stray CSS change to any theme fails CI with a visual diff artifact.

Verify: run twice for stability; deliberate 1px change dry run (revert).

---

## GAM-078 Startup-time budget

Why: startup is logged ("Startup completed in ...") but never asserted; regressions creep.

Priority P3. Estimate 2 h. Deps: none.

Files: check.yml (linux job step) or the tauri-driver suite.

Steps:

1. In the real-runtime e2e (GAM-031 infra), measure launch-to-window-ready; assert < 2000 ms on the CI runner (generous; catches order-of-magnitude regressions only).
2. Record the number per run in the job summary for trend eyeballing.

Acceptance: a pathological startup regression fails CI; trend visible.

Verify: three consecutive runs stable.

---

## GAM-079 i18n framework, EN + VI

Why: hardcoded English strings everywhere; VI is the home-market differentiator and an enterprise checkbox. GAM-032's error codes make backend messages translatable.

Priority P2. Estimate 10 h. Deps: GAM-032.

Files: react-i18next setup, src/locales/en.json + vi.json, component sweep, language picker in settings, docs note.

Steps:

1. Wire i18next with detection order: settings > OS locale > en; extract all UI strings (components, toasts, confirm dialogs, warnings list from validate_command codes).
2. Translate VI natively (not machine-flavored); keep keys semantic.
3. Map backend error codes + dangerous-command warning codes to localized strings (backend sends codes after GAM-032; move warning text to codes as part of this task).
4. Settings picker + persistence; vitest snapshot of both catalogs having identical key sets (CI check).

Acceptance: full UI usable in VI; key-set parity enforced in CI; no hardcoded user-facing strings left (lint rule or grep check).

Verify: key-parity test; manual VI walkthrough screenshots in the ledger.

---

## GAM-080 App-dir constant + bundle-id migration [gated D3]

Why: the app-data directory name com.github.zintaen.gam is hardcoded in four files; the bundle identifier still carries the personal namespace. If D3 says migrate, it must be one deliberate release with data migration; if D3 says keep, centralize the constant anyway.

Priority P1 (gated). Estimate 6 h. Deps: D3, GAM-073 preferred (config-dir already parameterized).

Files: gam-core config-dir plumbing, lib.rs migration step, tauri.conf.json identifier, docs.

Steps:

1. Either outcome: one APP_DIR constant / parameter, zero duplicated literals.
2. If migrating: on first launch of the new identifier, copy (not move) settings/groups/known-repos/backups from the old dir if present and mark migrated; keep the old dir untouched for rollback; updater continuity verified (endpoint + minisign key unchanged - identifier change does not break minisign, but macOS treats it as a new app: document that update-in-place will not occur and ship it as a manual-download release with clear notes).
3. Release notes template for the migration release; MANUAL.md data-location update.

Acceptance: fresh install, upgraded install, and rollback all keep user data; no duplicated dir literals remain.

Verify: three-scenario manual matrix in ledger; grep for the old literal returns only the migration shim.

---

## GAM-081 Windows ARM64 target

Why: ARM Windows fleets (Surface and successors) are no longer exotic; Tauri supports the target.

Priority P2. Estimate 3 h. Deps: GAM-051 (sign the new artifact too).

Files: release.yml matrix (windows-latest, target aarch64-pc-windows-msvc), release-notes table row, winget manifest arch entry (GAM-053 follow-up).

Steps:

1. Add the matrix leg with rust_target and tauri args; confirm NSIS/MSI bundling supports the target in current tauri-action; publish artifacts.
2. Extend the release-body table and checksums job.

Acceptance: signed ARM64 installer published alongside x64 on the next tag.

Verify: release run green; artifact installs on ARM hardware or VM (Stephen/ledger).
