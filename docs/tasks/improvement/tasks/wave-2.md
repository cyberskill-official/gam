# Wave 2 - tests, errors, privacy surface (P1-P2)

Gate per task: standard (see wave 0).

---

## GAM-030 Stable test ids; delete self-skipping e2e guards

Why: two of three e2e specs wrap assertions in `if ((await el.count()) > 0)` - they silently pass when the UI element disappears. Selectors are fragile class/placeholder guesses.

Priority P1. Estimate 3 h. Deps: none.

Files: e2e/alias-crud.spec.ts, touched components (TitleBar, SearchBar, SettingsDropdown, Toolbar, AliasList, ConfirmDialog).

Steps:

1. Add data-testid to the core interactive elements (title bar, search input, settings button and menu, add-alias button, alias rows, confirm dialog buttons).
2. Rewrite the three specs to assert unconditionally on testids; a missing element fails the test.
3. Extend the suite: open create form, validation error on bad name, cancel flow (still browser-mode; real IPC lands in GAM-031).

Acceptance: no conditional guards remain; suite fails if any target element is removed.

Verify: pnpm test:e2e locally (vite server); grep for "count()) > 0" returns nothing.

---

## GAM-031 Real-runtime e2e with tauri-driver

Why: current e2e runs the frontend in Chrome against Vite - it never exercises Tauri IPC, the Rust core, or the WebView. The crown-jewel path (alias CRUD against real git) is untested end to end.

Priority P1. Estimate 10 h. Deps: GAM-030.

Files: new e2e-tauri/ (WebdriverIO config + specs), .github/workflows/check.yml job, docs/CONTRIBUTING.md note.

Steps:

1. Add a webdriverio + tauri-driver setup per the Tauri v2 WebDriver docs; Linux CI job (ubuntu, webkit2gtk-driver, xvfb-run) building the debug binary.
2. Isolate the environment: HOME set to a temp dir with a scratch gitconfig and a temp repo so tests never touch the runner's real config; pass the repo path as the CLI arg.
3. Specs: add alias (assert via git config CLI), edit, delete, dangerous-command warning, import/export round-trip using the dialog-free code path (invoke export with a path if needed - acceptable to add a test-only command behind #[cfg(debug_assertions)]).
4. Keep the fast Playwright suite for UI logic; document which suite covers what.

Acceptance: CI runs the real binary; a Rust-side regression in alias CRUD fails CI.

Verify: green job on the branch; intentionally break add_alias locally and watch the suite fail (note in ledger, revert).

---

## GAM-032 Typed errors over IPC

Why: error.rs defines GamError with stable codes but every command returns bare strings; the frontend string-matches errors and cannot localize (blocks GAM-079).

Priority P2. Estimate 6 h. Deps: none.

Files: src-tauri/src/error.rs, commands.rs, git_service.rs (return types), src/types/index.ts, src/tauri-bridge.ts, toast/error display sites.

Steps:

1. Change IpcResult.error to { code: string, message: string }; services return GamError instead of String (From impls exist).
2. Map known git failures (not found, timeout, permission) to codes; unknown stays GIT_FAILED with the stderr as message.
3. Frontend: central errorMessage(code, message) helper; components stop string-matching.
4. Tests both sides (Rust serialization shape; frontend mapping).

Acceptance: every command returns coded errors; error.rs has no dead items.

Verify: cargo + vitest green; grep frontend for raw .error usage.

---

## GAM-033 Rust coverage + combined upload

Why: the 70% gate covers TypeScript only; Rust coverage is unmeasured, and no PR shows coverage movement.

Priority P2. Estimate 4 h. Deps: none.

Files: .github/workflows/check.yml, README badge, codecov.yml (or equivalent).

Steps:

1. Add cargo-llvm-cov to the check job (ubuntu only is fine) producing lcov; upload frontend lcov (switch vitest coverage reporter to also emit lcov) and Rust lcov to Codecov with flags frontend/rust.
2. Configure PR comment + patch coverage target 80 informational, project threshold hold-the-line.
3. Plan (comment in config) the ratchet: frontend lines 70 to 80 over the quarter.

Acceptance: PRs show coverage for both halves; gate remains deterministic locally.

Verify: CI run shows both flags; local pnpm test -- --coverage unchanged behavior.

---

## GAM-034 Command-layer tests, property tests, fuzzing

Why: commands.rs (the IPC surface) has zero direct tests; the two untrusted-input parsers (import JSON, shell history) deserve adversarial testing.

Priority P2. Estimate 8 h. Deps: GAM-032.

Files: src-tauri/src/commands.rs tests, src-tauri/fuzz/ (cargo-fuzz targets), proptest dev-dependency, CI optional nightly fuzz job.

Steps:

1. Unit-test command wrappers with an AppState fixture (temp dirs): success + error shape per command family.
2. proptest: validate_alias_name never panics and accepts/rejects per the documented charset; parse_alias_output(-z) total on arbitrary bytes.
3. cargo-fuzz targets: FileService::import_aliases on arbitrary bytes; history parsers on arbitrary text. Run 5 minutes in a scheduled weekly workflow (not per-PR).
4. Fix anything the fuzzers find (record in ledger).

Acceptance: parsers proven panic-free on arbitrary input; command surface has direct tests.

Verify: cargo test; cargo fuzz run each target briefly; scheduled workflow added.

---

## GAM-035 Windows history parsing test in CI

Why: PowerShell history parsing only executes on Windows and is currently exercised by no CI test; regressions would ship silently.

Priority P2. Estimate 3 h. Deps: none.

Files: src-tauri/src/ranking_service.rs tests.

Steps:

1. Refactor refresh_history_cache to take the history root path as a parameter (default: real location) so tests can point at a fixture dir on any OS.
2. Add fixture-based tests for the PSReadLine format; keep an OS-gated test asserting the real default path resolution on Windows runners.
3. check.yml already runs the matrix on windows-latest; confirm the new tests execute there.

Acceptance: PowerShell parsing has cross-platform unit coverage plus a Windows-runner path test.

Verify: CI matrix green on all three OSes.

---

## GAM-036 Ranking precision and large-history safety

Why: the zsh plain-line regex also matches timestamped lines (double count); `git\s+(\w+)` matches "git" mid-line (echo git push counts); whole-file read_to_string re-parses on a 5 s TTL - slow on 100 MB histories.

Priority P2. Estimate 4 h. Deps: none.

Files: src-tauri/src/ranking_service.rs.

Steps:

1. Anchor patterns: timestamped zsh lines consume the prefix; plain matcher requires start-of-line (^|;|&&) before git; add \b.
2. Parse each line with one matcher, never two (count once).
3. Tail-read: read only the last 512 KB of each history file (seek from end, align to newline); document the window.
4. Unit tests for double-count fix, mid-line non-match, tail-window behavior.

Acceptance: score for a fixture history matches hand-computed counts; parse cost bounded regardless of file size.

Verify: cargo test with new fixtures.

---

## GAM-037 PRIVACY.md and data inventory

Why: procurement asks "what does it read, store, send". The honest answer is excellent (gitconfigs; shell history only with consent; nothing leaves the machine except the updater HTTPS call) but it exists nowhere as a document.

Priority P1. Estimate 3 h. Deps: none.

Files: PRIVACY.md (root), README link, docs/MANUAL.md link.

Steps:

1. Inventory: reads (global/local gitconfig, shell history files with consent listed by exact path per shell), stores (app-data dir contents: settings.json, groups.json, known-repos.json, backups/, logs), transmits (updater check to the two GitHub URLs; nothing else; no telemetry).
2. State consent model (history ranking default and toggle), retention (user-controlled; clear-all-data), and jurisdiction posture (no personal data leaves the device; Vietnam PDPD and GDPR stance follows).
3. Keep it one page, plain language.

Acceptance: every path and URL in the doc verified against code; linked from README.

Verify: cross-check against source (updater endpoint in tauri.conf.json, history paths in ranking_service.rs, app-data files).

---

## GAM-038 Export-diagnostics bundle

Why: support without logs is guesswork; users should not hand-collect files.

Priority P2. Estimate 4 h. Deps: GAM-021.

Files: commands.rs (export_diagnostics), backend zip assembly, AboutPanel button.

Steps:

1. Command assembles a zip in a user-chosen location (save dialog): app version, OS, git --version output, settings.json (theme + toggles only), rotated logs, known-repos count (not paths), backups listing (names only). Never alias bodies, never shell-history content, never full repo paths.
2. AboutPanel: "Export diagnostics" button + toast with the chosen path.
3. Test: bundle contents allowlist-checked in a unit test (no forbidden files).

Acceptance: one click produces a support-ready bundle containing nothing sensitive.

Verify: unit test on the allowlist; manual open of a produced zip.

---

## GAM-039 IPC payload caps + plaintext-export warning

Why: no size bounds on alias name/command over IPC (cheap DoS insurance); exports are plaintext JSON while alias commands sometimes embed tokens (shell aliases with curl headers).

Priority P2. Estimate 3 h. Deps: none.

Files: git_service.rs (validate), commands.rs, ExportModal/DataPanel copy, file_service.rs.

Steps:

1. Enforce name <= 64 chars, command <= 8 KB at the command boundary with coded errors; mirror limits in the form UI.
2. Export flow: warning line "Exports are plaintext. Review for secrets before sharing." shown in the dialog area of the UI (Tauri save dialog itself cannot carry it, so surface in the confirm step or toast before opening the dialog).
3. Import: enforce the same field caps during validation (file cap 10 MB already exists).

Acceptance: oversized inputs rejected both sides; user sees the plaintext warning before an export.

Verify: unit tests for caps; vitest for form validation; manual export shows warning.

---

## GAM-040 Tighten CSP and HTTP headers

Why: the CSP is good but can pin down more: no object/base/form/frame directives are set.

Priority P2. Estimate 2 h. Deps: none.

Files: src-tauri/tauri.conf.json.

Steps:

1. Append to the CSP: object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'; worker-src 'self'.
2. Add the app.security.headers block (Tauri 2 HTTP headers) with X-Content-Type-Options nosniff and Referrer-Policy no-referrer.
3. Run the app in dev and release-build smoke to confirm fonts/images/updater still load (connect-src list unchanged).

Acceptance: stricter CSP ships with no functional regression.

Verify: manual smoke (themes render, update check runs); config diff reviewed.

---

## GAM-041 Narrow Tauri capabilities

Why: capabilities grant core:default wholesale; the app uses a small core subset. Least privilege should be literal.

Priority P2. Estimate 2 h. Deps: none.

Files: src-tauri/capabilities/default.json.

Steps:

1. Replace core:default with the specific core permissions the frontend uses (event listen/emit, window default set as needed, app default) - derive the set by grepping @tauri-apps/api usage in src/ and expanding until the app functions.
2. Keep dialog:default, updater:default, process:default (or narrow process to relaunch/exit actually used by UpdateModal).
3. Document the final permission set in docs/CODEBASE.md.

Acceptance: app fully functional with the narrowed set; capability diff reviewed.

Verify: dev smoke of every feature (aliases, groups, dialogs, update check, restart-after-update path mocked); release build boots.
