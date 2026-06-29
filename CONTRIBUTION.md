# Contribution Policy

Thank you for your interest in this project. This document explains how the repository is governed and how you can contribute.

---

## Open source, single-maintainer ("benevolent dictator")

This repository is **public and open source** — anyone is free to read the code, inspect the build pipeline, audit it for security, fork it, and download the published binaries at no cost.

However, it follows a **single-maintainer governance model**:

- **Direct pull requests that modify the workflow, build logic, or release process are not accepted.** All decisions about what gets built, how it is built, and what is published rest solely with the maintainer.
- This is deliberate. The build pipeline handles a security-sensitive product (a password manager) and pins every third-party dependency to an immutable commit SHA. Accepting external changes to that chain would weaken the supply-chain guarantees the pipeline is built to provide.

This is sometimes called a "benevolent dictator" model: fully transparent, but with one accountable owner who controls what ships. If you need a different build configuration, **forking** is explicitly encouraged — the pipeline is designed to be self-contained and portable.

---

## We welcome feedback via Issues

While code contributions through PRs are not accepted, **bug reports and security concerns are welcome via GitHub Issues.** 

Please open an issue if you encounter:
- 🐛 **Bugs or broken builds**: The release crashes, fails to run, or has obvious functional regressions.
- 🔒 **Security concerns**: Any potential vulnerabilities in the build pipeline or published artifacts.
- 📝 **Documentation gaps**: Unclear setup instructions or missing details in the README.

### Please Note
- **No ETA / No Guarantees**: This is a personal, hobbyist-maintained repository. Issues will be reviewed and addressed solely at the maintainer's convenience.
- **For custom needs, please fork**: If you need a specific version, a different feature set, or a custom build configuration, please follow the **Forking** section below and build it yourself.

---

## Forking

If you want full control — different features, a different platform target, your own release cadence — **fork the repository**. Everything you need is in `.github/workflows/vaultwarden-build.yml` and `.github/dependabot.yml`. Read [ARCHITECTURE.md](./ARCHITECTURE.md) to understand how to adapt it.

If you only want to use a specific version, click `GitHub Actions` -> `Vaultwarden Smart Build` -> `Run Workflow`, change the `version selection` to `custom`, and then type the corresponding version number below.

---

## Code of conduct

Be respectful and constructive in issues. This is a one-person-maintained project; clear, kind, and specific feedback is the fastest path to a fix.
