# GitHub Actions – Reusable CI/CD Workflows

This repository contains **reusable GitHub Actions workflows** used by application repositories to implement a fully automated CI/CD pipeline.

It is part of the **DevOps CI/CD Challenge** and demonstrates how common CI/CD logic can be centralized and reused across multiple projects using **GitHub reusable workflows (`workflow_call`)**.

---

## Purpose

The goal of this repository is to:
- Avoid duplication of CI/CD logic across repositories
- Encapsulate best practices for build, test, validation, and release
- Keep application repositories minimal and focused on business logic

All workflows here are **generic**, **parameterized**, and **application-agnostic**.

---

## Used by

- **Application repo:** https://github.com/ym-innowise/ts-sum-package

The application repo references these workflows via:

```yaml
uses: ym-innowise/workflows-library/.github/workflows/<workflow>.yml@main
```

Workflows are pinned to a tag (e.g. `@v1`) to ensure stability.

---

## Reusable workflow mechanism (`workflow_call`)

All workflows in this repository are implemented as **GitHub reusable workflows** using the
`workflow_call` trigger.

This allows application repositories to:
- Reuse CI/CD logic without duplicating YAML
- Pin workflows to a specific version tag
- Consume shared conventions and defaults centrally

Example usage from an application repository:

```yaml
jobs:
  pr-verify:
    uses: ym-innowise/workflows-library/.github/workflows/pr-verify.yml@main
    with:
      working-directory: '.'
```

---

## Shared configuration (Node.js version)

The Node.js version is defined **only at the GitHub Environment level**.

The workflows read the Node.js version exclusively from:
- GitHub Environment variable (e.g. `PROD → NODE_VERSION`)

A fallback value is defined in the workflow to ensure safe defaults if the environment
variable is missing.

---

## Reusable workflows overview

This repository provides the following reusable workflows.

### `pr-verify.yml` — Pull Request verification

Used on **every pull request**.

Responsibilities:
- Install dependencies using `npm ci`
- Enforce presence of `package-lock.json`
- Run:
  - Linting
  - Build
  - Unit tests
- Enforce that the package version is explicitly bumped in the PR

This workflow is intended to be configured as a **required status check** via branch protection.

---

### `e2e.yml` — Integration / E2E verification

Triggered via the **`verify` label** on a pull request.

Responsibilities:
- Install dependencies
- Build the project
- Run integration / E2E tests (`npm run e2e`)

Tests can be minimal or trivial, but must execute against the built artifacts.

This workflow is **label-driven** and only runs when the label is present.

---

### `pr-label-publish.yml` — Pre-merge release candidate (RC)

Triggered via the **`publish` label** on a pull request.

Responsibilities:
- Validate semantic versioning (`X.Y.Z`) in `package.json`
- Ensure the version is bumped compared to `main`
- Block the pipeline if a Git tag or GitHub Release for `vX.Y.Z` already exists
- Generate a **pre-merge release candidate version**:
  ```
  X.Y.Z-<RC_SUFFIX>-<short-sha>
  ```
  where `<RC_SUFFIX>` is configured via the GitHub Environment variable `RC_SUFFIX`
- Build and package the application using `npm pack`
- Upload the resulting tarball as a **GitHub Actions artifact**, named exactly as the RC version

The release candidate version is applied **only in CI** and is not committed back to the repository.

---

### `gh-release-on-merge.yml` — Release on merge

Triggered when a pull request with the **`publish` label** is merged into the main branch.

Responsibilities:
- Rebuild the project from `main`
- Re-run linting, build, and tests
- Ensure the release version does not already exist
- Create a Git tag:
  ```
  vX.Y.Z
  ```
- Create a **GitHub Release**
- Upload the packaged tarball as a release asset

This workflow performs the **actual “publish” step**, where “publish” means creating a GitHub Release (not publishing to npm).

---

## Label-driven behavior summary

| Label | Effect |
|------|-------|
| `verify` | Triggers E2E / integration tests |
| `publish` | Generates a pre-merge release candidate and enables release on merge |

Labels act as **opt-in behavior modifiers** and do not affect pull requests unless explicitly applied.

---

## Versioning strategy

- Semantic versioning is required (`X.Y.Z`)
- The version must be explicitly bumped in the pull request
- Pre-merge builds use a dynamically generated suffix:
  ```
  X.Y.Z-<RC_SUFFIX>-<short-sha>
  ```
- The final release uses the clean version:
  ```
  X.Y.Z
  ```

The suffix (`RC_SUFFIX`) is configured via GitHub Environment variables and is not hard-coded in workflows.

---

## Pull Request verification guarantees

Workflows in this repository are designed to be used together with **branch protection rules**
configured in the application repository.

In combination, they enforce the following guarantees on every pull request:

- **Up-to-date with base branch**  
  Pull requests must include all commits from the base branch (e.g. `main`) before they can be merged.
  This is enforced via strict required status checks, ensuring CI runs on the exact commit that will be merged.

- **Linear history**  
  Merge commits are disallowed. Pull requests must be rebased or squashed, resulting in a clean,
  linear Git history.

- **Mandatory validation**  
  Linting, build, and unit tests must all pass. If any check fails, GitHub blocks the merge.

These guarantees are enforced at the platform level (GitHub branch protection),
not via custom workflow logic.

---

## Design principles

- Reusable workflows are implemented using `workflow_call`
- No application-specific assumptions
- Centralized configuration via GitHub Environments
- Clear separation between:
  - **Validation** (PR verification)
  - **Preview artifacts** (release candidates)
  - **Publishing** (release on merge)
- No manual steps required after PR approval