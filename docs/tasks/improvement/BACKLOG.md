# GAM improvement backlog - master index

Source: internal enterprise-grade audit 2026-07-06 (F1-F22 findings, R1-R72 recommendations). Status legend: `todo | in_progress | done | verified | blocked(...) | skipped(...) | gated(D<n>)`. Detailed cards live in `tasks/wave-<n>.md`.

## Decision register (human-only, block the tasks that cite them)

| ID  | Decision                                                                                                                                                                | Blocks             | Status |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ | ------ |
| D1  | Enroll Apple Developer Program (99 USD/yr) and provide the six APPLE\_\* repo secrets                                                                                   | GAM-050            | open   |
| D2  | Create Azure subscription + Artifact Signing account (Basic SKU), provide signing identity                                                                              | GAM-051            | open   |
| D3  | Bundle identifier: keep `com.github.zintaen.gam` or migrate to `world.cyberskill.gam` (recommendation: migrate in one deliberate release before the install base grows) | GAM-080            | open   |
| D4  | Open-core boundary: which enterprise features (packs, policy, audit, CLI) stay MIT vs become commercial                                                                 | GAM-090..094 scope | open   |
| D5  | Telemetry stance: keep "nothing leaves the machine except the updater" or add opt-in crash/usage reporting                                                              | GAM-099            | open   |
| D6  | Product scope: best-in-class alias manager vs grow into a git environment manager (identities, hooks, ignores)                                                          | wave-5 shape       | open   |
| D7  | Create org accounts: `cyberskill-official/homebrew-tap` repo + HOMEBREW_GITHUB_TOKEN, winget publisher, npm org 2FA enforcement                                         | GAM-052, GAM-053   | open   |

## Wave 0 - truth and closable holes (P0, 13 h)

| ID      | Task                                                                                                              | Prio | Est | Deps | Status |
| ------- | ----------------------------------------------------------------------------------------------------------------- | ---- | --- | ---- | ------ |
| GAM-001 | Whitelist git config scope as an enum (kill `--file=` injection surface)                                          | P0   | 2h  | -    | todo   |
| GAM-002 | README and docs truth pass (remove phantom features, fix namespaces and engine drift)                             | P0   | 3h  | -    | todo   |
| GAM-003 | SECURITY.md, SUPPORT.md, issue templates, CODEOWNERS                                                              | P0   | 3h  | -    | todo   |
| GAM-004 | CodeQL (js-ts) and gitleaks workflows                                                                             | P0   | 3h  | -    | todo   |
| GAM-005 | Repo-settings hardening runbook + gh commands (rulesets, tag protection, secret scanning, private vuln reporting) | P0   | 2h  | -    | todo   |

## Wave 1 - core correctness and hardening (P1, 54 h)

| ID      | Task                                                                                  | Prio | Est | Deps    | Status |
| ------- | ------------------------------------------------------------------------------------- | ---- | --- | ------- | ------ |
| GAM-010 | exec_git timeout + non-interactive git env                                            | P1   | 4h  | -       | todo   |
| GAM-011 | Null-separated alias parsing (`git config -z`) + multiline regression tests           | P1   | 4h  | -       | todo   |
| GAM-012 | Atomic JSON persistence + corruption surfacing and quarantine                         | P1   | 6h  | -       | todo   |
| GAM-013 | update_alias: set-new-then-unset-old with rollback                                    | P1   | 2h  | GAM-011 | todo   |
| GAM-014 | Real gitconfig backup (rotating) + restore UI - makes the README claim true           | P1   | 10h | GAM-012 | todo   |
| GAM-015 | Remove panic paths; deny unwrap/expect/panic lints in lib code                        | P1   | 3h  | -       | todo   |
| GAM-016 | Known-repos hygiene: canonicalize, forget-repo UI, clear-all-data                     | P1   | 6h  | GAM-012 | todo   |
| GAM-017 | Single-instance plugin                                                                | P1   | 2h  | -       | todo   |
| GAM-018 | Async commands via spawn_blocking + bounded concurrent repo scan                      | P1   | 8h  | GAM-010 | todo   |
| GAM-019 | Dependency hygiene: tokio justified-or-removed, @cyberskill/shared to devDependencies | P1   | 1h  | GAM-018 | todo   |
| GAM-020 | Make pnpm audit and dependency-review blocking; add npm license gate                  | P1   | 3h  | -       | todo   |
| GAM-021 | Structured logging (tauri-plugin-log) + crash-log rotation                            | P1   | 4h  | -       | todo   |
| GAM-022 | npm supply-chain settings: pnpm minimumReleaseAge cooldown                            | P1   | 1h  | -       | todo   |

## Wave 2 - tests, errors, privacy surface (P1-P2, 52 h)

| ID      | Task                                                                           | Prio | Est | Deps    | Status |
| ------- | ------------------------------------------------------------------------------ | ---- | --- | ------- | ------ |
| GAM-030 | Stable data-testid attributes; delete self-skipping e2e guards                 | P1   | 3h  | -       | todo   |
| GAM-031 | Real-runtime e2e: tauri-driver + WebdriverIO CRUD path on Linux CI             | P1   | 10h | GAM-030 | todo   |
| GAM-032 | Wire GamError codes over IPC + frontend error mapping                          | P2   | 6h  | -       | todo   |
| GAM-033 | Rust coverage (cargo-llvm-cov) + coverage upload and PR diff gate              | P2   | 4h  | -       | todo   |
| GAM-034 | commands.rs tests, proptest validators, cargo-fuzz import parser               | P2   | 8h  | GAM-032 | todo   |
| GAM-035 | Windows PowerShell history parsing test in CI                                  | P2   | 3h  | -       | todo   |
| GAM-036 | Ranking precision: word boundaries, no double count, tail-read large histories | P2   | 4h  | -       | todo   |
| GAM-037 | PRIVACY.md: full data inventory and network-call statement                     | P1   | 3h  | -       | todo   |
| GAM-038 | Export-diagnostics bundle (logs, versions, env; never alias bodies)            | P2   | 4h  | GAM-021 | todo   |
| GAM-039 | IPC payload caps + plaintext-export warning                                    | P2   | 3h  | -       | todo   |
| GAM-040 | Tighten CSP + Tauri HTTP response headers                                      | P2   | 2h  | -       | todo   |
| GAM-041 | Narrow capabilities from core:default to the specific set used                 | P2   | 2h  | -       | todo   |

## Wave 3 - signing, distribution, IT enablement (P1-P2, 54 h)

| ID      | Task                                                                      | Prio | Est | Deps        | Status    |
| ------- | ------------------------------------------------------------------------- | ---- | --- | ----------- | --------- |
| GAM-050 | macOS Developer ID signing + notarization in release.yml                  | P0   | 6h  | D1          | gated(D1) |
| GAM-051 | Windows Authenticode via Azure Artifact Signing signCommand               | P0   | 6h  | D2          | gated(D2) |
| GAM-052 | Homebrew tap cask + enable release job                                    | P1   | 3h  | D7          | gated(D7) |
| GAM-053 | winget manifest + release automation                                      | P1   | 4h  | D7, GAM-051 | gated(D7) |
| GAM-054 | IT deployment guide (silent MSI, perMachine, Intune, Jamf, firewall URLs) | P1   | 6h  | -           | todo      |
| GAM-055 | MDM-managed update policy (disable/pin) honored at startup                | P1   | 8h  | -           | todo      |
| GAM-056 | Update channels (stable/beta) + manual check + skip-version               | P2   | 8h  | GAM-055     | todo      |
| GAM-057 | SHA256SUMS per release + attestation verification docs                    | P2   | 2h  | -           | todo      |
| GAM-058 | OpenSSF Scorecard workflow + badges; Best Practices badge checklist       | P2   | 3h  | -           | todo      |
| GAM-059 | Updater key custody + compromise runbook                                  | P2   | 2h  | -           | todo      |
| GAM-060 | Adopt the Tauri isolation pattern                                         | P2   | 6h  | -           | todo      |

## Wave 4 - architecture, performance, reach (P2-P3, 59 h)

| ID      | Task                                                                   | Prio | Est | Deps    | Status    |
| ------- | ---------------------------------------------------------------------- | ---- | --- | ------- | --------- |
| GAM-070 | Watch gitconfig files for external edits + manual refresh control      | P2   | 6h  | GAM-018 | todo      |
| GAM-071 | Event-driven ranking refresh (focus/mutation) instead of TTL polling   | P3   | 2h  | GAM-036 | todo      |
| GAM-072 | Typed IPC: generate TS types from Rust (tauri-specta)                  | P2   | 6h  | GAM-032 | todo      |
| GAM-073 | Extract gam-core crate (services without Tauri deps)                   | P2   | 10h | GAM-018 | todo      |
| GAM-074 | Version-sync guard in release CI                                       | P3   | 1h  | -       | todo      |
| GAM-075 | Renovate tuning (patch automerge, schedule, lockfile maintenance)      | P3   | 1h  | -       | todo      |
| GAM-076 | Accessibility gate: axe-core in Playwright + keyboard-only e2e + fixes | P2   | 8h  | GAM-030 | todo      |
| GAM-077 | Theme visual-regression screenshots (10 themes)                        | P3   | 4h  | GAM-030 | todo      |
| GAM-078 | Startup-time budget smoke in CI                                        | P3   | 2h  | -       | todo      |
| GAM-079 | i18n framework (react-i18next), EN + VI catalogs                       | P2   | 10h | GAM-032 | todo      |
| GAM-080 | Single app-dir constant + bundle-id migration with data copy           | P1   | 6h  | D3      | gated(D3) |
| GAM-081 | Windows ARM64 release target                                           | P2   | 3h  | GAM-051 | todo      |

## Wave 5 - enterprise tier and reach (P2-P3, 117 h)

| ID      | Task                                                                          | Prio | Est | Deps             | Status    |
| ------- | ----------------------------------------------------------------------------- | ---- | --- | ---------------- | --------- |
| GAM-090 | Team alias packs: signed bundle format, import from URL/repo, drift detection | P1   | 24h | GAM-073, D4      | gated(D4) |
| GAM-091 | Policy engine: allow/deny alias patterns via managed policy file              | P2   | 16h | GAM-090          | gated(D4) |
| GAM-092 | Config-as-code sync mode (designated git repo, optional read-only enforce)    | P2   | 12h | GAM-090          | gated(D4) |
| GAM-093 | Local audit trail of alias mutations + export                                 | P2   | 8h  | GAM-012          | todo      |
| GAM-094 | gam CLI companion on gam-core (apply/export/lint)                             | P2   | 16h | GAM-073, D4      | gated(D4) |
| GAM-095 | Docs site + product page (gam.cyberskill.world) with enterprise section       | P1   | 12h | GAM-054          | todo      |
| GAM-096 | Support + versioning policy docs (SemVer, fix window, security SLA)           | P2   | 2h  | -                | todo      |
| GAM-097 | THREAT-MODEL.md                                                               | P3   | 3h  | GAM-037          | todo      |
| GAM-098 | Linux reach: Flathub + AUR                                                    | P2   | 6h  | -                | todo      |
| GAM-099 | Opt-in crash/usage telemetry with published schema                            | P2   | 12h | D5               | gated(D5) |
| GAM-100 | Microsoft Store (MSIX) evaluation spike                                       | P3   | 4h  | GAM-051          | todo      |
| GAM-101 | Offline/air-gapped install documentation                                      | P3   | 2h  | GAM-055, GAM-057 | todo      |

## Totals

65 tasks. Wave 0: 13 h. Wave 1: 54 h. Wave 2: 52 h. Wave 3: 54 h. Wave 4: 59 h. Wave 5: 117 h. Program total: ~349 h of agent work plus 7 human decisions.

Recommendation coverage: R1-R72 all map to a task or a decision except R67/R71 (pure strategy, held as D4/D6) and R2's doc sweep which also absorbs F21/F22 consistency fixes.
