# Wave 5 - enterprise tier and reach (P2-P3)

Gate per task: standard. D4 (open-core boundary) shapes GAM-090/091/092/094: build them in-repo if D4 says MIT, or scaffold interfaces here + implementation in the commercial module if D4 draws the line there. D6 (product scope) may add tasks; it does not change these.

---

## GAM-090 Team alias packs [gated D4]

Why: the enterprise value proposition. Teams want one blessed alias set, versioned, verifiable, and drift-visible - not 40 developers with 40 configs.

Priority P1 (gated). Estimate 24 h. Deps: GAM-073, D4.

Files: gam-core pack module (format, verify, diff), commands, PackPanel UI, docs/enterprise/PACKS.md.

Steps:

1. Pack format: JSON with schema version, metadata (name, version semver, publisher), aliases[], optional groups, and a detached minisign signature file; document the format publicly.
2. Sources: import from file, URL, or git repo path (reuse exec_git for clone/pull into app-data cache); signature verification against publisher keys the user/policy trusts (trust-on-first-use with fingerprint display; policy file can pin keys - ties into GAM-055 keys).
3. Apply: preview diff (add/change/delete vs current scope), selective apply, record pack provenance per alias (extend group assignments storage pattern).
4. Drift detection: on load, compare live aliases against applied pack version; badge drifted aliases with "differs from <pack>@<version>" and offer re-apply/keep.
5. Tests: format round-trip, signature verify (good/bad/missing), drift matrix, URL fetch mocked.
6. PACKS.md: authoring guide + signing guide for team leads.

Acceptance: a team lead can publish a signed pack at a URL; a developer imports, applies, and sees drift when they diverge; bad signatures refuse loudly.

Verify: full-cycle integration test in temp repos; ledger walkthrough.

---

## GAM-091 Policy engine [gated D4]

Why: orgs need to forbid foot-gun aliases (force-push wrappers) fleet-wide, not warn politely.

Priority P2 (gated). Estimate 16 h. Deps: GAM-090 (shares policy plumbing with GAM-055).

Files: gam-core policy module, validate/add/update integration, PrivacyPanel-adjacent managed-state UI, docs/enterprise/POLICY.md.

Steps:

1. Policy file (same managed locations as GAM-055): denyPatterns[] / allowPatterns[] (regex on command), dangerousPatterns override list, enforcement mode warn|block.
2. Enforce at add/update in core (not only UI); blocked attempts return a coded error naming the rule; existing violating aliases flagged in the list view.
3. The hardcoded DANGEROUS_PATTERNS list becomes the built-in default set, overridable by policy.
4. Tests: block/warn matrices, malformed-policy resilience (fail closed to warn+log, never crash).

Acceptance: with a block policy, a forbidden alias cannot be created via UI or IPC and the reason is shown; audit of existing aliases lists violations.

Verify: core tests + real-runtime e2e case.

---

## GAM-092 Config-as-code sync mode [gated D4]

Why: platform teams keep golden configs in git; GAM should follow a repo, not fight it.

Priority P2 (gated). Estimate 12 h. Deps: GAM-090.

Files: gam-core sync module, settings UI, docs/enterprise/SYNC.md.

Steps:

1. Designate a sync source (git URL + path to pack file); scheduled pull (on launch + manual) applies the pack; read-only enforce mode (policy-settable) reverts local alias edits to the pack with notice.
2. Offline tolerance: last-fetched pack cached; failures degrade to cached state with a status indicator.
3. Tests: apply/revert cycles, offline fallback, enforce-mode revert.

Acceptance: pointing GAM at a repo keeps aliases converged with it; enforce mode survives local tampering.

Verify: integration test with a local bare repo as origin.

---

## GAM-093 Local audit trail

Why: compliance reviews ask "who changed what, when" even for local tools; also the natural hook for future central reporting.

Priority P2. Estimate 8 h. Deps: GAM-012.

Files: gam-core audit module (append-only JSONL in app-data), mutation hooks, DataPanel export, docs.

Steps:

1. Append one JSONL record per mutation (ts, action, scope, target path hash, alias name, old/new command hash - hashes not bodies, privacy-first) with size-capped rotation.
2. Export as JSON/CSV from DataPanel; include in clear-all-data.
3. Tests: every mutating command writes exactly one record; rotation.

Acceptance: a full history of alias changes is exportable; no command bodies stored.

Verify: unit tests; manual export inspection.

---

## GAM-094 gam CLI companion [gated D4]

Why: CI pipelines and terminal-native developers need pack apply/lint without a GUI; this is what makes packs enforceable in build pipelines.

Priority P2 (gated). Estimate 16 h. Deps: GAM-073, GAM-090, D4.

Files: crates/gam-cli (clap), release.yml artifacts, docs/enterprise/CLI.md.

Steps:

1. Subcommands: list (json/table), export, apply <pack> (--dry-run diff), lint (validate aliases against policy, exit nonzero on violations - the CI hook), verify <pack> (signature).
2. Reuse gam-core exclusively; no code duplication; same policy/pack semantics as the GUI.
3. Ship as separate binaries per platform in releases (cargo-dist or extend release.yml); checksummed + signed like the app.
4. Tests: golden-output CLI tests per subcommand.

Acceptance: gam lint in CI fails a repo whose aliases violate policy; apply matches GUI behavior byte-for-byte.

Verify: cargo tests + a sample GitHub Actions snippet in docs proven on a scratch repo.

---

## GAM-095 Docs site + product page

Why: adoption and procurement both start at a URL; GitHub READMEs do not carry enterprise sections well.

Priority P1. Estimate 12 h. Deps: GAM-054 (content), GAM-037 (privacy content).

Files: new site (Astro Starlight fits the stack) in site/ or a separate repo (Stephen's call at review), deployed to gam.cyberskill.world.

Steps:

1. Pages: home (value + screenshots), install (all channels), manual (from docs/MANUAL.md), enterprise (deployment guide, policy keys, privacy, security posture, verification), changelog feed.
2. Single-source: pull the existing markdown docs rather than forking their content.
3. SEO basics + OpenGraph; link prominently from cyberskill.world (funnel).

Acceptance: site live with the enterprise page answering deploy/privacy/verify without contacting anyone.

Verify: build in CI; lighthouse pass noted; Stephen wires DNS.

---

## GAM-096 Support + versioning policy

Why: enterprises standardize only on tools with a stated lifecycle.

Priority P2. Estimate 2 h. Deps: none.

Files: docs/SUPPORT.md (extend), docs/VERSIONING.md.

Steps:

1. Declare SemVer discipline, release cadence intent, fix policy (latest release + previous minor receive fixes), security-fix SLA reference to SECURITY.md, and the update-channel meanings (GAM-056).

Acceptance: lifecycle questions answerable from one page.

Verify: consistency check against SECURITY.md and RELEASING.md.

---

## GAM-097 THREAT-MODEL.md

Why: writing the model down converts audit knowledge into a living security baseline reviewers can extend.

Priority P3. Estimate 3 h. Deps: GAM-037.

Files: docs/THREAT-MODEL.md.

Steps:

1. Assets (gitconfigs, shell history, backups), trust boundaries (webview/IPC/core; updater endpoint; pack sources), actors, STRIDE-lite table mapping each threat to the shipped mitigation (scope enum, isolation pattern, signature checks, policy), and accepted risks.

Acceptance: every mitigation cites the code or config that implements it.

Verify: cross-reference pass against the repo.

---

## GAM-098 Flathub + AUR

Why: Linux enterprise and enthusiast reach; AppImage alone has no update/discovery story.

Priority P2. Estimate 6 h. Deps: none (signing not required, repos have their own trust).

Files: flathub manifest repo (org), AUR PKGBUILD (gam-bin), release.yml bump automation, docs install section.

Steps:

1. Flatpak manifest (webkit runtime), submit to Flathub review; automate version bumps on release.
2. AUR gam-bin PKGBUILD from the .deb or AppImage; maintainership documented.

Acceptance: flatpak install flathub world.cyberskill.gam (or current id per D3) and yay -S gam-bin both work.

Verify: install logs in ledger; bump automation dry run.

---

## GAM-099 Opt-in telemetry [gated D5]

Why: only if D5 says yes. Blind releases are hard to support at fleet scale, but the zero-telemetry posture is itself a selling point - decision first.

Priority P2 (gated). Estimate 12 h. Deps: D5.

Files: sentry-rust + JS (or GlitchTip self-hosted per D5 detail), consent UI, PRIVACY.md update, docs/TELEMETRY.md schema.

Steps:

1. Crash reporting + a handful of counted events (app_open, alias_added - names only, never values); OFF by default, consent screen with the schema linked; policy key to force-disable (GAM-055).
2. Publish TELEMETRY.md listing every event and field; PRIVACY.md updated in the same PR.
3. Scrub paths/usernames in crash payloads.

Acceptance: nothing transmits before consent; schema doc matches emitted events exactly; policy can force it off fleet-wide.

Verify: network log with consent off (zero calls beyond updater); event audit test.

---

## GAM-100 Microsoft Store evaluation spike

Why: Store distribution gives Intune-native supply and auto-updates outside the self-updater; worth a timeboxed answer, not an assumption.

Priority P3. Estimate 4 h. Deps: GAM-051.

Files: docs/improvement/runbooks/msix-evaluation.md.

Steps:

1. Timebox: package the current build as MSIX (tauri supports the target), test identity/updater interaction (Store apps must not self-update - policy hook from GAM-055 solves this), list cert/publisher requirements and fees, and recommend go/no-go.

Acceptance: a written go/no-go with effort estimate replaces speculation.

Verify: the runbook exists with a tested MSIX artifact or documented blockers.

---

## GAM-101 Air-gapped install docs

Why: regulated networks install from internal mirrors with the updater off; make that a paragraph, not a support ticket.

Priority P3. Estimate 2 h. Deps: GAM-055, GAM-057.

Files: docs/enterprise/DEPLOYMENT.md section.

Steps:

1. Document: download + SHA256/attestation verification on a connected machine, transfer, silent install per OS, policy file disabling updates, and the statement that GAM makes no other network calls (link PRIVACY.md).

Acceptance: an admin can deploy fully offline from this section alone.

Verify: walkthrough against a VM with networking disabled (evidence in ledger).
