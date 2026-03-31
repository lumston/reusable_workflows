# reusable_workflows

A collection of reusable GitHub Actions workflows for common CI/CD tasks.

## Workflows

### `check_credentials` — Scan for AWS Credentials

**File:** `.github/workflows/scan_credentials.yml`

Scans repository files for accidentally committed AWS credentials or environment files.

**Triggers:** `workflow_call`, `workflow_dispatch`

**What it does:**
- Scans all normal files for AWS access key patterns (regex `AK[A-Z0-9]{18}`)
- Scans hidden files (dotfiles) for the same patterns
- Searches for environment files (`.env`, etc.) outside of allowed paths
- Fails with an error message if any credentials or env files are found

**Exclusions:** `*.d.ts`, `*.tsbuildinfo`, `node_modules/`, `.git/`, `dist/`, `.gitignore`, `src/assets/`, `src/environments/`

**Usage:**
```yaml
jobs:
  scan:
    uses: lumston/reusable_workflows/.github/workflows/scan_credentials.yml@main
```

---

### `release-branch-created` — Create Initial RC Tag

**File:** `.github/workflows/release-branch-created.yml`

Creates the first release candidate tag (`vX.Y.Z-rc.1`) and a GitHub prerelease when a `release/*` branch is created.

**Triggers:** `workflow_call`

**Inputs:**

| Name | Required | Description |
|------|----------|-------------|
| `branch_name` | Yes | The release branch name (e.g. `release/1.2.0`) |
| `repository` | Yes | The target repository (e.g. `org/repo`) |

**Secrets:**

| Name | Required | Description |
|------|----------|-------------|
| `token` | Yes | GitHub token with `contents: write` permission |

**What it does:**
1. Extracts the version from the branch name (strips `release/` and optional `v` prefix)
2. Checks that `vX.Y.Z-rc.1` tag and release do not already exist
3. Creates and pushes the `vX.Y.Z-rc.1` tag
4. Creates a GitHub prerelease with auto-generated notes

**Usage:**
```yaml
on:
  create:

jobs:
  rc:
    if: startsWith(github.ref_name, 'release/')
    uses: lumston/reusable_workflows/.github/workflows/release-branch-created.yml@main
    with:
      branch_name: ${{ github.ref_name }}
      repository: ${{ github.repository }}
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

---

### `release-branch-push` — Increment RC Tag on Push

**File:** `.github/workflows/release-branch-push.yml`

Creates the next release candidate tag (`vX.Y.Z-rc.N`) and a GitHub prerelease on every push to a `release/*` branch.

**Triggers:** `workflow_call`

**Inputs:**

| Name | Required | Description |
|------|----------|-------------|
| `repository` | Yes | The target repository (e.g. `org/repo`) |
| `branch_name` | Yes | The release branch name (e.g. `release/1.2.0`) |

**Secrets:**

| Name | Required | Description |
|------|----------|-------------|
| `token` | Yes | GitHub token with `contents: write` permission |

**What it does:**
1. Extracts the version from the branch name
2. Finds the highest existing `vX.Y.Z-rc.*` tag and increments it
3. Creates and pushes the next RC tag (skips if already exists)
4. Creates a GitHub prerelease with auto-generated notes (skips if already exists)

Uses a concurrency group (`release-rc-<repo>-<branch>`) to prevent race conditions on parallel pushes.

**Usage:**
```yaml
on:
  push:
    branches:
      - 'release/**'

jobs:
  rc:
    uses: lumston/reusable_workflows/.github/workflows/release-branch-push.yml@main
    with:
      branch_name: ${{ github.ref_name }}
      repository: ${{ github.repository }}
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

---

### `release-merged-to-main` — Promote RC to Stable Release

**File:** `.github/workflows/release-merged-to-main.yml`

Promotes the latest RC tag to a stable release (`vX.Y.Z`) when a `release/*` branch is merged into `main`.

**Triggers:** `workflow_call`

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `repository` | Yes | — | The target repository (e.g. `org/repo`) |
| `base_ref` | No | `main` | The branch that was merged into |
| `head_ref` | Yes | — | The release branch that was merged (e.g. `release/1.2.0`) |
| `merged` | Yes | — | Whether the PR was actually merged (`true`/`false`) |

**Secrets:**

| Name | Required | Description |
|------|----------|-------------|
| `token` | Yes | GitHub token with `contents: write` permission |

**What it does:**
1. Skips if `merged` is `false` or `head_ref` does not start with `release/`
2. Finds the latest `vX.Y.Z-rc.*` tag for the version
3. Ensures the final stable tag does not already exist
4. Creates and pushes the `vX.Y.Z` stable tag on the base branch
5. Creates a stable GitHub release with auto-generated notes

**Usage:**
```yaml
on:
  pull_request:
    types: [closed]
    branches:
      - main

jobs:
  promote:
    uses: lumston/reusable_workflows/.github/workflows/release-merged-to-main.yml@main
    with:
      repository: ${{ github.repository }}
      head_ref: ${{ github.head_ref }}
      merged: ${{ github.event.pull_request.merged }}
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Changelog Configuration

**File:** `.github/release.yml`

Configures how GitHub auto-generates release notes by mapping PR labels to changelog categories.

| Category | Labels |
|----------|--------|
| 🚀 Features | `feat`, `feature` |
| 🐛 Bug Fixes | `fix`, `bugfix`, `hotfix` |
| ♻️ Refactors | `refactor` |
| 🧪 Tests | `test`, `tests` |
| 📝 Documentation | `docs`, `documentation` |
| 🧹 Maintenance | `chore`, `ci`, `build` |
| ⚠️ Breaking Changes | `breaking-change` |

PRs labeled with `ignore-for-release`, `skip-changelog`, or `dependencies` (including Dependabot) are excluded from release notes.
