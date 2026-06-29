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

While code contributions through PRs are not the model here, **feedback is highly valued and is the primary way to influence this project.** Please open an issue for any of the following:

| Use case | Example |
|---|---|
| 🐛 **Report a broken or stale binary** | A release crashes on launch, or a published version is far behind upstream. |
| 🏷️ **Request a specific version** | You need a build of a particular older release or commit that isn't published yet. |
| 💡 **Suggest an improvement** | A different feature set, a smaller binary, better release notes, etc. |
| 🔒 **Report a security concern** | Anything about the binary, the pipeline, or the published artifacts. |
| 📝 **Point out a documentation gap** | Something in the README or ARCHITECTURE.md that is unclear or missing. |

### Tips for a useful issue

- **Include the release tag** you are referring to (e.g. `vw-win-v1.36.0-f21a3adae2fb-r6`).
- **Describe what you expected vs. what happened.** Logs, error messages, and your Windows version help a lot.
- **For version requests**, state the exact upstream tag or commit you want built.

### How requests are handled

- Version-build requests are acted on by the maintainer using the pipeline's `custom` mode (no action needed from you beyond the request).
- Bug reports are triaged and, where confirmed, fixed by the maintainer and re-published.
- Security reports can also be sent privately if you prefer — see the contact details below (if none are listed here, open an issue marked "security" and the maintainer will follow up).

---

## Forking

If you want full control — different features, a different platform target, your own release cadence — **fork the repository**. Everything you need is in `.github/workflows/vaultwarden-build.yml` plus `.github/dependabot.yml`. Read [ARCHITECTURE.md](./ARCHITECTURE.md) to understand how to adapt it.

---

## Code of conduct

Be respectful and constructive in issues. This is a one-person-maintained project; clear, kind, and specific feedback is the fastest path to a fix.
