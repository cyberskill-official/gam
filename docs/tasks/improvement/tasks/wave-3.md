# Wave 3 - signing, distribution, IT enablement (P1-P2)

Gate per task: standard. Tasks marked gated(D<n>) start only after Stephen closes the decision; the agent must skip them cleanly and note the skip in the ledger.

---

## GAM-050 macOS Developer ID signing + notarization [gated D1]

Why: unsigned .dmg means Gatekeeper "damaged app" friction; managed Macs cannot run GAM at all. The env-key scaffolding already exists as comments in release.yml.

Priority P0 (gated). Estimate 6 h. Deps: D1 (Apple Developer Program + six APPLE\_\* repo secrets set by Stephen).

Files: .github/workflows/release.yml, docs/RELEASING.md, tauri.conf.json (macOS signing options if needed).

Steps:

1. Restore the six APPLE\_\* env keys on the tauri-action step exactly as the removed block documented (certificate, cert password, signing identity, apple id, app password, team id).
2. Enable hardened runtime (Tauri default with signing) and confirm entitlements suffice for the updater relaunch.
3. Cut a prerelease tag on a test branch; verify: spctl -a -vv accepts the app, stapler validate passes, auto-update from the previous version still verifies (minisign key unchanged).
4. Update RELEASING.md secret table and the release-notes template (remove the right-click-Open note).
5. Agent handles workflow and docs only; Stephen enters all secret values.

Acceptance: released .dmg installs with zero Gatekeeper warnings on a clean Mac; update path intact.

Verify: spctl/stapler output pasted into ledger from Stephen's Mac (agent cannot notarize locally).

---

## GAM-051 Windows Authenticode via Azure Artifact Signing [gated D2]

Why: unsigned .exe/.msi trips SmartScreen and most corporate allowlisting. Microsoft's managed service (renamed from Trusted Signing to Artifact Signing; Basic SKU) signs via HSM in the pipeline - no cert file to protect.

Priority P0 (gated). Estimate 6 h. Deps: D2 (Azure account, Artifact Signing account + identity validation done by Stephen).

Files: .github/workflows/release.yml, src-tauri/tauri.conf.json (bundle > windows > signCommand), docs/RELEASING.md.

Steps:

1. Configure tauri.conf.json windows signCommand to invoke the Artifact Signing client (trusted-signing-cli or Invoke-AzTrustedSigning path documented by the service) with account/profile from env.
2. Add the Azure login step (OIDC federated credential preferred over client secrets) to the windows matrix leg only.
3. Test on a prerelease tag: signtool verify /pa passes on .exe and .msi; SmartScreen reputation note added to docs (signing removes the hard block; reputation builds).
4. Update RELEASING.md secrets/roles table.

Acceptance: released Windows artifacts are Authenticode-signed with a timestamp; verify step green in CI.

Verify: signtool verify output in CI log; ledger link.

---

## GAM-052 Homebrew tap cask [gated D7]

Why: the release workflow's Homebrew job is written and gated behind ENABLE_HOMEBREW, but the tap repo does not exist; README already advertises the install line.

Priority P1 (gated). Estimate 3 h. Deps: D7 (Stephen creates cyberskill-official/homebrew-tap + HOMEBREW_GITHUB_TOKEN secret).

Files: Casks/gam.rb in the tap repo (authored here, applied there), release.yml (no change expected), docs/README.md.

Steps:

1. Author Casks/gam.rb: version, sha256, aarch64 + intel dmg urls (livecheck optional), app stanza "GAM.app", zap stanza removing app-data dirs.
2. Hand Stephen the exact repo-var and secret steps; after creation, set ENABLE_HOMEBREW=true.
3. Dry-run the bump job logic against the latest release (the sed replacements) on a fork/branch of the tap.

Acceptance: brew install --cask cyberskill-official/tap/gam works on a clean Mac; release job bumps it automatically on the next tag.

Verify: brew install output from Stephen's Mac in ledger; next release run green.

---

## GAM-053 winget manifest + automation [gated D7]

Why: winget is the default corporate install path on Windows; absence means manual downloads.

Priority P1 (gated). Estimate 4 h. Deps: D7 (publisher account), GAM-051 (signed artifacts strongly preferred before submission).

Files: manifests for microsoft/winget-pkgs (CyberSkill.GAM), .github/workflows/release.yml (winget bump job).

Steps:

1. Author the three-file manifest set (version, installer for the x64 .exe or .msi with silent switches, locale) under publisher CyberSkill; validate with winget validate + winget install --manifest locally on Windows (or in CI).
2. Add a release job using vedantmgoyal9/winget-releaser or wingetcreate update in CI to PR the new version on each tag (token scoped to a fork).
3. Document the first-submission review lag in RELEASING.md.

Acceptance: winget install CyberSkill.GAM resolves and installs silently; version bumps automated.

Verify: winget-pkgs PR link; install log.

---

## GAM-054 IT deployment guide

Why: pilots die when IT has to reverse-engineer silent installs and firewall rules.

Priority P1. Estimate 6 h. Deps: none (references signed artifacts once D1/D2 land; write now, mark signing-dependent lines).

Files: docs/enterprise/DEPLOYMENT.md (new), README link.

Steps:

1. Windows: msiexec /i GAM_x.x.x_x64_en-US.msi /qn ALLUSERS=... note (verify the WiX default install scope and document per-user vs per-machine reality; if per-user only, state it and file a follow-up), Intune Win32 packaging steps (.intunewin, install/uninstall commands, detection rule by MSI product code), uninstall string.
2. macOS: dmg -> app copy, Jamf policy outline, first-run notes; MDM notes for the updater policy (GAM-055 keys).
3. Linux: deb/rpm silent flags, AppImage note.
4. Network: the exact two updater URLs to allowlist and the statement that nothing else is contacted.
5. Keep every claim executable: each command tested once, evidence in ledger.

Acceptance: an admin can deploy silently on all three OSes from this page alone.

Verify: commands executed in ledger evidence (Windows steps on a runner or Stephen's VM).

---

## GAM-055 MDM-managed update policy

Why: enterprises must be able to disable self-update or pin a channel fleet-wide; today the updater always checks on launch.

Priority P1. Estimate 8 h. Deps: none.

Files: src-tauri/src/settings_service.rs or new policy_service.rs, lib.rs, useUpdater.ts, docs/enterprise/DEPLOYMENT.md.

Steps:

1. Read-only policy sources, checked before any update check: macOS managed preferences (CFPreferences/plist at /Library/Managed Preferences/<bundle-id>.plist), Windows registry HKLM\SOFTWARE\Policies\CyberSkill\GAM, Linux /etc/gam/policy.json.
2. Keys: updates.enabled (bool), updates.channel (string), historyRanking.allowed (bool - lets IT force the consent off).
3. Policy wins over user settings; UI shows "managed by your organization" state on affected toggles.
4. Tests: policy file fixtures per OS (path injection for tests); UI disabled-state test.
5. Document keys + example profiles in DEPLOYMENT.md.

Acceptance: with policy present, no update network call is made and the UI says why; without it, behavior unchanged.

Verify: fixture tests; manual run with a local policy file; network assertion via logs.

---

## GAM-056 Update channels + manual check

Why: everyone updates on launch from one channel today; there is no beta ring, no "check now", no skip-version.

Priority P2. Estimate 8 h. Deps: GAM-055.

Files: tauri.conf.json (endpoint templating), useUpdater.ts, UpdateModal.tsx, AboutPanel.tsx, release.yml (publish beta latest.json on prerelease tags), docs/RELEASING.md.

Steps:

1. Endpoints: stable keeps latest.json; beta tags (v*-beta.*) publish beta-latest.json; updater endpoint chosen at runtime from settings/policy (updater builder allows endpoint override at check time).
2. Settings: channel picker (stable default, hidden behind policy if pinned); AboutPanel "Check for updates" button; skip-this-version persisted.
3. Periodic re-check every 24 h while running (timer), silent unless found.
4. Tests: endpoint selection logic; skip-version suppression.

Acceptance: beta ring works end to end on a prerelease tag; manual check and skip behave.

Verify: prerelease dry run; vitest for logic.

---

## GAM-057 SHA256SUMS + verification docs

Why: attestation and .sig files exist, but there is no one-command checksum path and no user-facing verification doc.

Priority P2. Estimate 2 h. Deps: none.

Files: release.yml (checksums job), docs/README.md or docs/enterprise/DEPLOYMENT.md section.

Steps:

1. After the release job, download all artifacts, produce SHA256SUMS, upload to the release (mirror of the sbom job pattern).
2. Document: sha256sum -c line, and gh attestation verify <file> --repo cyberskill-official/gam for provenance.

Acceptance: every release carries SHA256SUMS; docs show both verification paths.

Verify: next tag run; command outputs in ledger.

---

## GAM-058 OpenSSF Scorecard + badges

Why: Scorecard is the metric procurement teams and OSS insight tools actually read; most of its checks are already satisfied and unclaimed.

Priority P2. Estimate 3 h. Deps: none.

Files: .github/workflows/scorecard.yml, README badges, docs note.

Steps:

1. Add ossf/scorecard-action (pinned) on schedule + branch-protection trigger, publishing results (security-events: write, id-token: write for the badge API).
2. Add the Scorecard badge and the CI badge to README.
3. Create the openssf best-practices (bestpractices.dev) application checklist as docs/improvement/runbooks/openssf-best-practices.md with the passing/gap answers prefilled; Stephen submits the form.

Acceptance: scorecard runs green with a score recorded in ledger; badges render.

Verify: workflow run + badge URL.

---

## GAM-059 Updater key custody + compromise runbook

Why: the signing key has already been rotated twice under pressure; the procedure lives in commit messages and memory, not in a runbook. Key custody must survive people forgetting.

Priority P2. Estimate 2 h. Deps: none.

Files: docs/improvement/runbooks/updater-key.md (internal-safe: no secrets, procedure only), RELEASING.md link.

Steps:

1. Document: where the key lives (password manager entry + the two GitHub secrets, nowhere else), passphrase rules (no shell-special characters - documented lesson), generation command, how to verify the secret pair matches before tagging (local sign test), rotation procedure, and the compromise playbook (rotate, pin new pubkey, ship a manual-reinstall release, announce).
2. Add a pre-release checklist line: verify signer pair with a dry sign.

Acceptance: a maintainer who has never touched the key can rotate it safely from this page.

Verify: dry-read walkthrough; no secret material appears in the doc.

---

## GAM-060 Adopt the Tauri isolation pattern

Why: defense in depth for the IPC bridge: the isolation iframe intercepts and can sanitize every message from the main frontend before it reaches Rust; recommended by Tauri for production apps.

Priority P2. Estimate 6 h. Deps: none (after GAM-041 to avoid re-testing capabilities twice).

Files: tauri.conf.json (app > security > pattern), new isolation dir (index.html + hook script), vite config if bundling needs it.

Steps:

1. Configure pattern: isolation with a minimal hook that validates payload shapes for the known commands (name/command length, scope enum echo of GAM-001/GAM-039) and passes through.
2. Verify dev + release builds and all commands still function (dialogs, updater).
3. Document the pattern and its guarantees in docs/CODEBASE.md.

Acceptance: isolation active in release builds; full feature smoke passes.

Verify: manual smoke matrix in ledger; e2e suites green.
