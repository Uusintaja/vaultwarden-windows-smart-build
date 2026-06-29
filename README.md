# Vaultwarden Smart Build

Pre-built, **standalone Windows binaries** of [Vaultwarden](https://github.com/dani-garcia/vaultwarden) — the lightweight, self-hostable server API compatible with [Bitwarden](https://bitwarden.com) clients.

This repository does not modify Vaultwarden. It automatically tracks the upstream releases, compiles a **dependency-free Windows executable**, bundles the matching web vault, and publishes ready-to-use packages to **Releases** — so you can self-host your password vault on Windows without installing a Rust toolchain, vcpkg, or any build dependencies.

---

## What you get

Each release is a single `vaultwarden.zip` containing:

| File | Description |
|---|---|
| `vaultwarden.exe` | The server binary. **SQLite-only**, **fully statically linked** — no DLLs, no runtime installer, copy and run. |
| `web-vault/` | The Bitwarden web UI, matched to the version the binary expects. |

The binary is built with `--no-default-features --features sqlite` (no MySQL/PostgreSQL) and `-Ctarget-feature=+crt-static` (OpenSSL + C runtime linked in). Every package is smoke-tested: the exe is actually launched and kept alive before publishing, so a broken build is never released.

---

## How to get a version

### 1. Pick a release

Go to the **[Releases](../../releases)** page.

| Type | Tag pattern | Marked as | Recommendation |
|---|---|---|---|
| **Stable** | `vw-win-v{version}-{sha}-r{run}` | **Latest** | ✅ Use this for production. Always tracks an upstream tagged release. |
| **Pre-release** | `vw-win-...-latest-r{run}` or `...-custom-r{run}` | *Pre-release* | For testing main-branch snapshots or specific commits. |

The release title and body show the exact Vaultwarden version, source commit, web-vault version, package size, and SHA256 — so you always know precisely what you are downloading.

### 2. Download and verify

1. Download `vaultwarden.zip` from the release assets.
2. (Optional) Verify integrity against the **SHA256** listed in the release body:
   ```powershell
   (Get-FileHash vaultwarden.zip -Algorithm SHA256).Hash.ToLower()
   ```
3. Extract the archive.

### 3. Run it

```powershell
.\vaultwarden.exe
```

By default the server listens on `http://localhost:80` and looks for the `web-vault/` folder next to the executable. Common options:

```powershell
# Custom port + explicit web-vault path + data folder
.\vaultwarden.exe --rocket-port=8222 -w .\web-vault --data-folder .\data
```

See the [Vaultwarden documentation](https://github.com/dani-garcia/vaultwarden/wiki) for the full configuration reference (admin panel, HTTPS, backups, etc.). The binary behaves identically to an official Vaultwarden build with the SQLite backend.

> **Security note:** this binary handles your passwords. Treat the host, the data folder, and the `ADMIN_TOKEN` with the same care as any production secret. Always back up the `data/` folder.

---

## Customizing the build

This pipeline supports three build modes and is designed to be forked and adapted. If you want to understand how it works, tune the cache strategy, change the feature set, or run your own builds, read:

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — full design document: why each decision was made, the defect history it avoids, the tiered cache strategy, security model, and key algorithms.
- **[CONTRIBUTION.md](./CONTRIBUTION.md)** — repository contribution policy and how to give feedback.

The build is driven by `.github/workflows/vaultwarden-build.yml`. In a fork, you can trigger a manual build from the **Actions** tab (*Run workflow*) choosing `release`, `latest`, or `custom` (with a specific commit/tag/branch).

---

## How releases stay current

A scheduled job runs daily and checks whether upstream has published a new release. It only starts an expensive Windows build when a genuinely new version appears — and only when that exact `version + commit` has not already been published here. Redundant and stale pre-releases are cleaned up automatically, while stable releases are never deleted.

---

## Acknowledgements

- [Vaultwarden](https://github.com/dani-garcia/vaultwarden) by Daniel García — the actual project. This repo only repackages it for Windows.
- [Bitwarden](https://bitwarden.com) — the upstream password manager.
- Built with Rust, vcpkg (OpenSSL), and the GitHub Actions ecosystem.
