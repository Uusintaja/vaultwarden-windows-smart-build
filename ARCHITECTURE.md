# Vaultwarden Smart Build — Architecture

> Target: `vaultwarden-build.yml` (production)
> Purpose: a GitHub Actions pipeline for Windows "auto-version-tracking + auto-publish + auto-cleanup" of Vaultwarden.
> This document explains **why** it is designed this way, not merely **what** it does.

---

## 1. Design Goals

### 1.1 Positive Requirements

- **Auto-version-tracking**: daily scheduled check of the upstream `dani-garcia/vaultwarden` for new releases; build only when a new version appears.
- **Auto-publish**: package the build artifacts (exe + web-vault) and publish to GitHub Releases automatically.
- **Auto-cleanup**: retain valuable releases and automatically remove redundant / stale ones, keeping Releases tidy.
- **Dual mode**: support scheduled (`schedule`) and manual (`workflow_dispatch`) runs; manual runs can select `release / latest / custom`.

### 1.2 Negative Requirements (the real design driver)

This pipeline was **designed from scratch** after auditing two prior versions (v2, v3). Every defect exposed by those versions was turned into a "must-not" list and avoided at the architecture level — this is the most important part of this document.

| ID | Historical defect | How this design fixes it |
|---|---|---|
| **H1** | Cleanup grouped by "version number" → stable releases of the same version but different commits were deleted | Group key = `version \| sha`; stable releases never enter the cap pool |
| **H2** | `custom` mode crashed on a commit with no reachable tag (`git describe`) | Version parsing falls back to `dev`; never exits 1 |
| **M1** | Claimed env-based injection protection but interpolated `${{ }}` directly into shell | **Zero `${{ }}` interpolation in run blocks**; everything via `env:` |
| **M2** | Unauthenticated web-vault download hit the 60 req/hour rate limit | `Bearer token` + `--retry` everywhere |
| **M3** | Used a `last-tag.txt` cache to record "last built version"; evicted by actions.cache's 7-day LRU, failing during quiet periods | No cache record; **query own releases** directly for an existing `version+sha` |
| **ABI** | vcpkg wide `restore-keys` pulled back an ABI-mismatched old OpenSSL on upgrade | Exact key; wide fallback intentionally omitted |
| **Discovery** | openssl-sys crashed on a missing vcpkg status db when the Rust cache missed | Explicit `OPENSSL_LIB_DIR/INCLUDE_DIR` bypasses the vcpkg-rs discovery mechanism |

> **Core philosophy**: rely on no intermediate state that has an expiry, can fail, or needs extra maintenance (cache records, wide-prefix fallback). Every decision is based on a re-queryable fact (upstream API, own releases, whether a file truly exists).

---

## 2. Architecture Overview

The pipeline is three jobs chained by dependencies and conditions into a decide → execute → fallback flow:

```
┌─────────────────────────────────────────────────────────────┐
│  check  (ubuntu-latest · seconds · nearly free)             │
│  Decision: fetch upstream → validate inputs → resolve sha   │
│            → decide whether to build                        │
│  outputs: should_build / mode / target_tag / target_sha     │
└──────────────────────────┬──────────────────────────────────┘
                           │ if: should_build == 'true'
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  build  (windows-latest · ~30min · expensive)               │
│  Execution: compile → smoke → package → publish → cleanup   │
│  (Windows runner starts only when compilation is needed)    │
└──────────────────────────┬──────────────────────────────────┘
                           │ if: always() && either failed
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  notify (ubuntu-latest · on failure)                        │
│  Fallback: create/append a failure issue (fixed title)      │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 Why check runs on Ubuntu (the most important decision)

Windows runner billing minutes are about 2× Linux, and a build takes ~30 minutes. Front-loading all lightweight decisions (fetch upstream tag, validate custom input, query whether already built) onto **nearly-free Ubuntu** has two benefits:

1. **Cost**: when the version is unchanged (the common case for scheduled checks), the whole pipeline runs only a seconds-long Ubuntu job; no Windows runner starts.
2. **Security**: custom-input injection validation runs on Ubuntu, so malicious input is rejected before it ever touches the Windows environment.

The `build` job is gated by `if: needs.check.outputs.should_build == 'true'`, ensuring the Windows runner starts only when compilation is genuinely needed.

---

## 3. Run Modes

Four actual triggers map to three internal `MODE`s (schedule normalizes to release):

| Trigger | MODE | target ref | cache write-back | release attrs |
|---|---|---|---|---|
| schedule (daily) | `release` | upstream latest release tag | ✅ write | stable (make_latest) |
| dispatch: release | `release` | upstream latest release tag | ✅ write | stable (make_latest) |
| dispatch: latest | `latest` | main branch HEAD | ❌ read-only | prerelease |
| dispatch: custom | `custom` | user-specified ref | ❌ read-only | prerelease |

### 3.1 Smart skip (the core value of schedule)

Schedule mode is not "build blindly every day"; it is "check every day whether a build is needed" (the `Decide whether to build` step):

1. Dereference the target tag → short commit sha (precise identity).
2. Paginate the **own repo's** releases and check for an existing build of this `version+sha` via the regex `^vw-win-v?{ver}.*-{sha}(-|$)`.
3. Found → `should_build=false` (skip); not found → build.

**Why not a cache record (M3)**: actions.cache has a 7-day LRU eviction + a 10 GB total cap, and Vaultwarden release intervals often exceed 7 days. Once the cache record is evicted, a "previously built version" is misjudged as "not built", triggering a redundant build (wasting ~30 minutes of Windows quota). Querying own releases is an O(1) re-verifiable fact with no expiry.

### 3.2 Concurrency control

```yaml
concurrency:
  group: vw-${{ github.event.inputs.tag_choice || 'release' }}
```

- schedule and manual release share the `vw-release` group → **serialized**: they build the same version; running them concurrently only wastes quota and causes cleanup races.
- latest / custom get independent groups → **parallel**: they build different content and do not block each other.

---

## 4. Cache Strategy (Tiered Caching)

This is the subsystem that reflects "mode differences." **Not all caches are treated equally across modes** — a cache's value depends on whether its artifact will be reused by a future build.

### 4.1 Cache layer overview

| Cache layer | key dimension | release write-back | latest/custom write-back | Reason |
|---|---|---|---|---|
| **Rust build artifacts** | Cargo.lock hash | ✅ | ❌ read-only | version-specific: custom's old Cargo.lock is never reused |
| **web-vault frontend** | web-vault version | ✅ | ❌ read-only | version-specific: custom's old frontend is never reused |
| **vcpkg OpenSSL** | vcpkg commit hash | ✅ | ✅ | **shared across versions**: independent of vaultwarden version |
| **OPENSSL_LIB_DIR** | — (env var) | ✅ set | ✅ set | correctness: schedule tracks new versions ⇒ always a Rust miss |

### 4.2 Why Rust / web-vault are read-only for latest/custom

- **Save cost**: the Rust cache is ~500 MB and takes ~2 min to upload. latest/custom one-off artifacts (a main snapshot, a specific old version) will almost certainly never be built again; storing them is pure waste.
- **Prevent key bloat**: one cache entry per version, never reused → fills the 10 GB quota → evicts the genuinely valuable vcpkg cache.
- Implemented via Swatinem/rust-cache's native `save-if: ${{ env.MODE == 'release' }}`; web-vault uses a conditional save step.

### 4.3 Why vcpkg is written back for all modes

**Key insight**: the vcpkg cache key is `vcpkg-openssl-static-{vcpkg_commit}-v4`, keyed on the **runner's vcpkg version**, **independent of the vaultwarden version**. 1.35.8 and 1.36.0 use the same OpenSSL.

So it is a **cross-version shared infrastructure cache**, the opposite nature of Rust/web-vault (version-specific): any mode's write-back benefits all future builds. If a build skips write-back and the runner then upgrades vcpkg, the next build waits 9–11 min to rebuild OpenSSL for nothing.

### 4.4 Why OPENSSL_LIB_DIR is set for all modes (correctness > economy)

See [6.3 OpenSSL discovery fix](#63-openssl-discovery-fix-the-most-subtle-and-severe).

---

## 5. Security Design

### 5.1 Injection protection (M1, fully realized)

**Principle**: zero `${{ }}` interpolation inside `run:` blocks. All external values (inputs, step outputs, github context) are injected via the `env:` block; the script reads only `$env:VAR` / `$VAR`.

So even if a value contains something like `';rm -rf /'`, it is merely the value of a string variable and is never interpreted by the shell as a command. The same applies to the `build` PowerShell steps, which read `$env:VAR` throughout.

> Verifiable: the script can be statically scanned; `${{ }}` occurrences in run blocks = 0.

### 5.2 custom input validation

Three lines of defense, all in the check job (Ubuntu):

1. **Character allowlist**: `^[A-Za-z0-9][A-Za-z0-9._/-]{0,127}$`, rejecting spaces, semicolons, backticks, etc.
2. **Path-traversal guard**: rejects `..` and `//`.
3. **Leading/trailing char check**: rejects refs starting or ending with `.`/`/`/`-`.

### 5.3 Least privilege

```yaml
permissions:        # top level: minimal
  contents: read
# job level, raised as needed:
#   check  -> contents: read
#   build  -> contents: write (publish release)
#   notify -> issues: write (create issue)
```

And checkout uses `persist-credentials: false`, so GITHUB_TOKEN is not written into the checked-out repo's git config.

### 5.4 Supply-chain defense: Action SHA pinning

All third-party Actions are pinned to **immutable commit SHAs** (not mutable major-version tags like `@v6`):

```yaml
# ❌ mutable tag: after compromising a maintainer account, an attacker can force-push the tag to a malicious commit
- uses: softprops/action-gh-release@v3
# ✅ immutable SHA: locked to the exact reviewed commit; trailing "# vX" lets dependabot track it
- uses: softprops/action-gh-release@718ea10b132b3b2eba29c1007bb80653f286566b # v3
```

**Why it's necessary** (real incidents, not theory):
- **tj-actions/changed-files (2025-03)**: 23,000+ repos compromised; CI secrets dumped to logs.
- **aquasecurity/trivy-action (2026-03, 3 months ago)**: TeamPCP force-pushed 76/77 version tags to credential-stealing commits, exfiltrating CI secrets from 10,000+ workflows. Ars Technica and the Microsoft Security Blog confirmed: "workflows referencing mutable tags automatically pulled the malicious code, while SHA-pinned ones were unaffected."

**Why this pipeline especially needs it**: `softprops/action-gh-release` holds GITHUB_TOKEN and creates Releases — once poisoned, an attacker can publish malicious artifacts or push malicious tags. It is the highest-value target in the pipeline.

**The staleness counter-argument and its resolution**: SHA immutability means security fixes are not picked up automatically; pinning without updating is an anti-pattern. The companion `dependabot.yml` opens weekly PRs to update the SHAs (preserving the `# vX` comments) — manual review + merge keeps the pins both locked and current.

**Note on rust-toolchain**: `dtolnay/rust-toolchain@stable` uses a branch (`stable`). Pinning to a SHA freezes the **installer script** only; the runtime still installs the latest stable Rust via rustup — **the Rust version is not frozen**, so behavior is unchanged.

**What SHA pinning cannot prevent**: an attacker editing the SHA in a PR to point at a malicious commit from a fork (the vector revealed by the 2026 Trivy attack). This is mitigated by **branch protection + code review**, outside the workflow layer.

---

## 6. Key Algorithms

### 6.1 Version parsing with fallback (H2 fix)

```powershell
$describe = git describe --tags --always --long
# regex accepts only digit-leading real version numbers; a pure sha (no reachable tag) falls back to "dev"
if ($describe -match '^(\d+\.\d+(?:\.\d+)?(?:-[A-Za-z0-9.]+)?)(?:-\d+-g[0-9a-f]+)?$') {
    $ver = "v" + $matches[1]
}
if (-not $ver) { $ver = "dev" }
```

The historical H2 defect: when a commit has no reachable tag, `git describe` returns a pure short SHA, wrongly treated as a version → fails the version-format check → build crash. This design uses a strict digit-leading regex + a `dev` fallback, guaranteeing any commit yields a valid version.

### 6.2 Cleanup algorithm (H1 fix)

The release tag encodes "identity"; formats differ by mode:

- Stable (release): `vw-win-{version}-{sha12}-r{run_number}`
- Prerelease (latest/custom): `vw-win-{version}-{sha12}-{mode}-r{run_number}`

Cleanup is two steps:

**Step (1): exact dedup**
```
group key = version | sha12   ← sha is a dimension!
```
Multiple builds of the same `version+sha` (e.g. rebuilds) keep only the newest. **Key point**: the group key includes sha, so two builds with the "same version, different commit" **are never in the same group** — this is exactly why the v2/v3 H1 mis-delete scenario (build a stable release, then run latest, stable gets deleted) is fixed.

**Step (2): prerelease capping**
- Only `prerelease=true` releases participate; **stable releases never enter**.
- Sort by `published_at` descending; keep the newest `KEEP=8`; delete the rest.
- `CURRENT_TAG` (just created) is doubly protected: it is first in descending order so it never falls into the deletion tail, and the loop has a `-ne $CUR` backstop.

**Final protection for stable releases**: (1) different group dimension (no mutual deletion with latest); (2) excluded from the cap pool. Even if latest/custom build a release with the same version number, the stable release is untouched.

### 6.3 OpenSSL discovery fix (the most subtle and severe)

<a id="63-openssl-discovery-fix-the-most-subtle-and-severe"></a>

**Symptom**: in custom mode switching to an earlier version (e.g. 1.35.8), `cargo build` reports:
```
openssl-sys: Could not find directory of OpenSSL installation
vcpkg did not find openssl: could not read status file updates dir (os error 3)
```

**Root-cause chain** (confirmed via vcpkg-rs source and rust-openssl issues):

```
Cargo.lock differs → Rust cache key mismatch/miss → openssl-sys re-runs its build-script
  → build-script falls back to the vcpkg-rs discovery mechanism → vcpkg-rs reads installed/vcpkg/updates/
  → but that status dir is not cached (only installed/x64-windows-static, the lib itself, is cached) → crash
```

The vcpkg-rs source (`load_ports`) does `read_dir(status_path/updates)`; when missing it throws `could not read status file updates dir` — exactly the error text.

**Why the first release tests didn't expose it**: same version → Rust cache hit → openssl-sys doesn't re-run its build-script → the missing status db is masked. Switching versions in custom broke that "lucky condition."

**Fix**: explicitly set
```yaml
OPENSSL_LIB_DIR: 'C:\vcpkg\installed\x64-windows-static\lib'
OPENSSL_INCLUDE_DIR: 'C:\vcpkg\installed\x64-windows-static\include'
```

rust-openssl's discovery priority is `OPENSSL_LIB_DIR → INCLUDE_DIR → OPENSSL_DIR → pkg-config → vcpkg-rs`. With the first set, openssl-sys **locates the library in the very first branch and never calls vcpkg-rs**; the status db being missing is irrelevant.

**Why it's set for all modes (not just custom)**: schedule's job is to track new versions; a new version = a new Cargo.lock = an inevitable Rust cache miss. If release/schedule didn't set it, the crash would recur on the **main path** — far worse than a sporadic custom failure. And setting it unconditionally costs nothing (it is only read when openssl-sys recompiles, and the value is correct then). **The crash condition is mode-independent, so the fix must be mode-independent.**

> This fix has a bonus: no cache-path change, no cache-key bump — the existing vcpkg cache is reused directly, with no rebuild.

---

## 7. Artifact Specification

### 7.1 Binary properties

- **SQLite only**: `--no-default-features --features sqlite` drops MySQL/PostgreSQL for the smallest size.
- **Fully static linking**: `OPENSSL_STATIC=1` + `RUSTFLAGS=-Ctarget-feature=+crt-static -Cstrip=debuginfo`. The artifact has no external DLL dependencies — copy and run.

### 7.2 Packaging

`7z a -tzip -mx=9` preserves the `web-vault` top-level directory (the zip contains `vaultwarden.exe` + `web-vault/`); faster and smaller than `Compress-Archive`. A SHA256 checksum is attached; the Release body carries full provenance (commit, mode, web-vault version, build-run link).

### 7.3 Smoke test

Rather than checking file existence only, it:
1. Validates the PE header (MZ magic) + a sane size.
2. **Actually launches the exe and keeps it alive for 4 seconds** before passing.

This catches "compiled successfully but crashes on launch due to ABI/static-link mismatch" — a class of defect a file-existence check cannot detect.

---

## 8. Failure Notification

The `notify` job triggers when check or build fails:

- Looks up an open issue with the fixed title `🚨 Vaultwarden build FAILED`:
  - exists → appends a comment (with the failed run link); no duplicate issue (no spam).
  - doesn't exist → creates one.
- `exit 0`: the notification itself never fails the workflow (otherwise a notification failure would mask the real build-failure status).
- `github.event_name != 'workflow_run'` guard: if the workflow is forked/wrapped and invoked via workflow_run, the upstream is responsible for notification.

---

## 9. Observability

| Mechanism | Location |
|---|---|
| Decision log | `::notice::decision: should_build=... reason=...` (visible in the Actions summary) |
| Cleanup log | each deletion prints the tag + type (dedup/cap) + a total |
| Release provenance | body carries commit/sha/mode/web-vault version/build-run link |
| Failure issue | auto-created/appended, with the failed run link |
| Artifact fallback | if Release creation fails, the artifact is retained for 14 days and can be recovered manually |

---

## 10. Future Work

Low-risk directions for evolution:

1. **Windows ARM64 matrix**: `matrix: [x64, arm64]`, with a matching vcpkg triplet.
2. ~~**Pin third-party Actions to SHAs**~~: ✅ **done** (see 5.4, with dependabot.yml).
3. **Configurable KEEP**: expose the cleanup cap count `8` as a workflow input.
4. ~~**Webhook notification**~~: superseded — GitHub Watch with a "releases" trigger pushes to email, which is sufficient.
5. **Dry-run enhancement**: in dry_run mode, print "the version that would be built / the releases that would be deleted" without executing.
6. **Defense in depth** (optional): step-security/harden-runner (egress network policy, blocks secret exfiltration), zizmor (SAST scan of the workflow). Note harden-runner is itself a third-party Action that would also need SHA pinning.

---

## Appendix: Defect-Avoidance Quick Reference

| Concern | Where | Mechanism |
|---|---|---|
| Stable release not mis-deleted | Cleanup · KeyOf | version+sha grouping + stable excluded from cap pool |
| Version parsing doesn't crash | Get version | digit regex + dev fallback |
| Injection protection | all run blocks | zero `${{ }}`, all via env |
| Rate-limit protection | all API calls | Bearer token + --retry |
| Skip decision doesn't expire | Decide whether to build | query own releases, no cache dependency |
| OpenSSL ABI correct | vcpkg cache | exact key, no wide restore-keys |
| openssl-sys discovery | Build · env | OPENSSL_LIB_DIR bypasses vcpkg-rs |
| custom input safety | Resolve target version | allowlist + traversal + leading/trailing checks |
| Runtime defects | Smoke test | actually launch, alive 4 seconds |
| Failure visibility | notify job | auto create/append issue |
| Supply-chain poisoning | all actions @SHA | commit SHA pin + dependabot auto-update |
| Node 24 compatible | all actions | checkout@v6 / cache@v5 / artifact@v7 / gh-release@v3 |
