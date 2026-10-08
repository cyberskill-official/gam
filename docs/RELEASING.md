# Release Process

This document describes how to build and release GAM as a desktop application.

## Prerequisites

- [Node.js](https://nodejs.org/) >= 24.0.0
- [pnpm](https://pnpm.io/) >= 10.x
- [Rust](https://rustup.rs/) (stable toolchain)
- [Git](https://git-scm.com/) installed and on `PATH`
- GitHub repository push access (for tagging and CI)

## Signing Setup (One-Time)

Tauri requires signed update artifacts. Generate a keypair once:

```bash
pnpm tauri signer generate -w ~/.tauri/gam.key
```

This outputs:

- **Private key** → `~/.tauri/gam.key` (never share this)
- **Public key** → printed to stdout (already in `tauri.conf.json`)

### GitHub Secrets (for CI)

Add these secrets in **Settings → Secrets → Actions**:

| Secret                               | Value                                |
| ------------------------------------ | ------------------------------------ |
| `TAURI_SIGNING_PRIVATE_KEY`          | Contents of `~/.tauri/gam.key`       |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | Password you chose during generation |

### Local `.env` (for local builds)

Add the same values to `.env` (gitignored):

```env
TAURI_SIGNING_PRIVATE_KEY=<contents of ~/.tauri/gam.key>
TAURI_SIGNING_PRIVATE_KEY_PASSWORD=<your password>
```

The `pnpm build` script uses `dotenv-cli` to load `.env` automatically.

## Build Scripts

| Command          | Description                                                |
| ---------------- | ---------------------------------------------------------- |
| `pnpm build`     | Build for the current platform (auto-detected)             |
| `pnpm build:mac` | Build a universal macOS app (Apple Silicon + Intel, macOS) |
| `pnpm dev`       | Start Tauri + Vite dev server                              |

`pnpm build:mac` needs both Rust targets (`rustup target add aarch64-apple-darwin x86_64-apple-darwin`) and writes its bundles to `src-tauri/target/universal-apple-darwin/release/bundle/`. The release workflow does not use either script: `tauri-action` runs `pnpm tauri build` itself (with a per-architecture `--target` on macOS).

> [!NOTE]
> Cross-platform builds are handled by CI via `tauri-apps/tauri-action`. macOS builds require macOS runners, and Linux builds require `libwebkit2gtk-4.1-dev`.

## Local Build

To build the app locally for testing:

```bash
# Install dependencies
pnpm install

# Build for your current platform (reads signing keys from .env)
pnpm build
```

> [!NOTE]
> `pnpm build` uses `dotenv-cli` to load `TAURI_SIGNING_PRIVATE_KEY` and `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` from `.env`. Without these, the build will fail at the signing step.

The built artifacts are output to `src-tauri/target/release/bundle/`:

| Platform | Artifact               |
| -------- | ---------------------- |
| macOS    | `GAM.app`, `.dmg`      |
| Windows  | `GAM.exe`, `.msi`      |
| Linux    | `gam.AppImage`, `.deb` |

## Automated Release (CI/CD)

GAM uses a two-step release process: a local script triggers the release, and GitHub Actions builds the artifacts.

### Step 1 — Run the release script

```bash
# Patch release (1.0.0 → 1.0.1)
pnpm release

# Minor release (1.0.0 → 1.1.0)
pnpm release minor

# Major release (1.0.0 → 2.0.0)
pnpm release major
```

The `scripts/release.js` script does the following automatically:

1. Bumps version in `package.json` (no git tag yet)
2. Generates a changelog entry from recent commits
3. Commits the version bump and changelog
4. Creates a `v<version>` git tag
5. Pushes the commit and tag to `origin`

### Step 2 — GitHub Actions builds the release

Pushing a `v*` tag triggers the `.github/workflows/release.yml` workflow:

1. **Builds** the Tauri app on all platforms (`macos-latest` arm64+x86_64, `ubuntu-24.04`, `windows-latest`) using `tauri-apps/tauri-action`
2. **Signs** the update artifacts using `TAURI_SIGNING_PRIVATE_KEY` from GitHub Secrets
3. **Publishes** the artifacts as a GitHub Release (`.dmg`, `.AppImage`, `.msi`, `.exe`, `latest.json`)
4. **Updates Homebrew** — Bumps the `gam` cask in the `cyberskill-official/homebrew-tap` repository (only when the repository variable `ENABLE_HOMEBREW` is `true`)

### Release artifacts

After CI completes, the GitHub Release page will contain:

| Platform | Files                                                                 |
| -------- | --------------------------------------------------------------------- |
| macOS    | `.dmg` (x64, aarch64), `.app.tar.gz` and `.app.tar.gz.sig` (updater)  |
| Windows  | `-setup.exe` (NSIS), `.msi`, each with its `.sig`                     |
| Linux    | `.AppImage`, `.deb`, `.rpm`, each with its `.sig`                     |
| Updater  | `latest.json` (auto-update manifest); SBOM `sbom-<version>.spdx.json` |

## Manual / Pre-release Checklist

Before running `pnpm release`, ensure:

- [ ] All tests pass: `pnpm test`
- [ ] Rust compiles cleanly: `cd src-tauri && cargo check`
- [ ] Local build succeeds: `pnpm build`
- [ ] `CHANGELOG.md` (repository root) is up to date with notable changes
- [ ] No uncommitted changes: `git status` is clean

## Tauri Configuration

The build configuration lives in `src-tauri/tauri.conf.json`. Key settings:

- **App identifier**: `com.github.zintaen.gam`
- **Window**: 1100×750, min 800×550
- **Bundle**: all Tauri targets (`tauri.conf.json` `"targets": "all"`; the release workflow ships dmg, nsis, msi, AppImage, deb and rpm)

## Homebrew Installation (macOS)

Once a release is published, macOS users can install via:

```bash
brew install --cask cyberskill-official/tap/gam
```

---

## 🍌 Support

If GAM saves you time and brainpower, consider fueling its development with a banana!

[<img src="https://cdn.buymeacoffee.com/buttons/v2/default-violet.png" alt="Buy Me A Banana" height="50">](https://buymeacoffee.com/zintaen)

Or scan the QR Code:

<img src="screenshots/buy-me-a-coffee.png" alt="Buy Me A Banana QR" width="200" style="border-radius: 12px;">
