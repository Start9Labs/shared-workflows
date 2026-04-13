# shared-workflows

Reusable GitHub Actions workflows for building, tagging, and releasing StartOS service packages (.s9pk).

## How It Works

The CI pipeline has two automatic stages, plus an optional manual path:

```
PR opened/updated ──> Build
PR merged to master ──> Version check ──> Tag ──> Build ──> GitHub Release ──> [Publish to registry]
Manual tag push ──> Build ──> GitHub Release ──> [Publish to registry] (bypasses version check)
```

Tags created by GitHub Actions (via `GITHUB_TOKEN`) do not trigger other workflows. The tag pushed by **tagAndRelease** will *not* trigger the **release.yml** caller workflow — instead, tagAndRelease calls release directly as a reusable workflow. The standalone **release.yml** caller only runs when a tag is pushed manually.

### 1. Build on PR

When a pull request is opened or updated against `master`, **build.yml** builds the `.s9pk` to verify it compiles. Draft PRs are skipped. If a new commit arrives on the PR, the previous build is cancelled and a fresh one starts.

### 2. Tag and release on merge

When a PR is merged to `master`, **tagAndRelease.yml** runs:

1. Extracts the package ID and current version from source (via the **extract-version** action)
2. Queries the production registry to check if that version already exists
3. If it exists, the workflow **skips** the release gracefully — no tag is created, no build runs
4. If it doesn't exist, deletes any prior GitHub release and tag for that version, then creates a fresh tag
5. Calls **release.yml** to build, create a GitHub release, and (if S3 is configured) publish to the test registry

If the production registry is unreachable, the workflow fails rather than silently proceeding.

### 3. Manual release

You can push a tag manually to trigger **release.yml** directly, bypassing the version check. This is useful for re-releasing or testing. Note that only manually pushed tags trigger this workflow — tags created by **tagAndRelease** (via `GITHUB_TOKEN`) do not, since GitHub Actions does not trigger workflows from actions performed by `GITHUB_TOKEN`.

## Version Encoding

Versions are extracted from `startos/versions/index.ts` by the **extract-version** composite action. Tags strip the flavor prefix and replace `:` with `_`:

| Version                | Tag              | Release Name           |
| ---------------------- | ---------------- | ---------------------- |
| `2.0.0:2`              | `v2.0.0_2`       | `2.0.0:2`              |
| `29.3:5`               | `v29.3_5`        | `29.3:5`               |
| `#knots:29.3:2-beta.1` | `v29.3_2-beta.1` | `#knots:29.3:2-beta.1` |

The flavor is omitted from tags because each repo only produces one flavor. The full version (including flavor) is used as the GitHub release name.

## Workflows

### build.yml

Builds the service package(s). Does not publish or create releases. If `DEV_KEY` is not provided, generates a temporary signing key.

### tagAndRelease.yml

Checks the current version against a production registry. If the version already exists, the workflow exits gracefully without building. Otherwise, creates a release tag and calls **release.yml** to build and publish.

### release.yml

Builds the service package(s), creates a GitHub release, and (optionally) publishes to a StartOS registry. Called by tagAndRelease as a reusable workflow, or triggered independently by a manual tag push.

Registry publishing is gated on `S3_S9PKS_BASE_URL`: if it is not provided, the workflow builds and creates the GitHub release but skips S3 upload, registry publish, and GitHub mirror registration. This lets forks and downstream packagers use these workflows with only a GitHub release as the distribution channel.

### uploadArtifacts.yml

Internal workflow called by build and release. Splits the aggregated build output into individual artifacts per `.s9pk` file.

## Composite Actions

### extract-version

Extracts `package_id`, `version`, and `tag` from the service source code. Used by tagAndRelease and release.

### setup-build-env

Sets up the full build environment: Node.js, Rust, Docker, QEMU, and `start-cli`.

### free-disk-space

Frees disk space on the GitHub Actions runner by removing unused SDKs and tools.

## Usage

Service repositories should define three workflow files:

### 1. Build on PR (`.github/workflows/build.yml`)

```yaml
name: Build

on:
  workflow_dispatch:
  pull_request:
    branches: ["master"]
    paths-ignore: ["*.md"]

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true

jobs:
  build:
    if: github.event.pull_request.draft == false
    uses: start9labs/shared-workflows/.github/workflows/build.yml@master
    # with:
    #   FREE_DISK_SPACE: true
    secrets:
      DEV_KEY: ${{ secrets.DEV_KEY }}
```

### 2. Tag and release on merge (`.github/workflows/tagAndRelease.yml`)

`REFERENCE_REGISTRY` is required — it's the registry queried to decide whether the current version has already been released (and should therefore be skipped). The `RELEASE_REGISTRY` / `S3_S9PKS_BASE_URL` / `S3_ACCESS_KEY` / `S3_SECRET_KEY` inputs are only needed if you want to publish the built package to a StartOS registry. If you omit them, the workflow still runs — it tags, builds, and creates a GitHub release, and stops there.

```yaml
name: Tag and Release

on:
  push:
    branches: ["master"]
    paths-ignore: ["*.md"]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  tag-and-release:
    uses: start9labs/shared-workflows/.github/workflows/tagAndRelease.yml@master
    with:
      REFERENCE_REGISTRY: ${{ vars.REFERENCE_REGISTRY }}
      # FREE_DISK_SPACE: true
      # Optional — omit these to publish only to GitHub Releases:
      RELEASE_REGISTRY: ${{ vars.RELEASE_REGISTRY }}
      S3_S9PKS_BASE_URL: ${{ vars.S3_S9PKS_BASE_URL }}
    secrets:
      DEV_KEY: ${{ secrets.DEV_KEY }}
      # Optional — omit these to publish only to GitHub Releases:
      S3_ACCESS_KEY: ${{ secrets.S3_ACCESS_KEY }}
      S3_SECRET_KEY: ${{ secrets.S3_SECRET_KEY }}
    permissions:
      contents: write
```

### 3. Release on manual tag push (`.github/workflows/release.yml`, optional)

This workflow only triggers on manually pushed tags — tags created by the tagAndRelease workflow (via `GITHUB_TOKEN`) do not trigger it. As with tagAndRelease, the registry / S3 inputs are optional; omit them to produce only a GitHub release.

```yaml
name: Release

on:
  push:
    tags:
      - "v*.*"

jobs:
  release:
    uses: start9labs/shared-workflows/.github/workflows/release.yml@master
    with:
      # FREE_DISK_SPACE: true
      # Optional — omit these to publish only to GitHub Releases:
      RELEASE_REGISTRY: ${{ vars.RELEASE_REGISTRY }}
      S3_S9PKS_BASE_URL: ${{ vars.S3_S9PKS_BASE_URL }}
    secrets:
      DEV_KEY: ${{ secrets.DEV_KEY }}
      # Optional — omit these to publish only to GitHub Releases:
      S3_ACCESS_KEY: ${{ secrets.S3_ACCESS_KEY }}
      S3_SECRET_KEY: ${{ secrets.S3_SECRET_KEY }}
    permissions:
      contents: write
```

## Required Secrets and Variables

Only `DEV_KEY` (for signed builds) and `REFERENCE_REGISTRY` (for tagAndRelease) are truly required. Everything else is optional — if the S3 / registry inputs are omitted, `release.yml` skips registry publishing and distributes packages exclusively through GitHub Releases.

### Repository Secrets

| Secret          | Used by        | Required?                  | Description                                                        |
| --------------- | -------------- | -------------------------- | ------------------------------------------------------------------ |
| `DEV_KEY`       | build, release | release: yes; build: no    | Developer signing key. If absent in build, a temporary key is used |
| `S3_ACCESS_KEY` | release        | only if publishing to S3   | S3 access key for package uploads                                  |
| `S3_SECRET_KEY` | release        | only if publishing to S3   | S3 secret key for package uploads                                  |

### Repository Variables

| Variable             | Used by       | Required?                | Description                                                                     |
| -------------------- | ------------- | ------------------------ | ------------------------------------------------------------------------------- |
| `REFERENCE_REGISTRY` | tagAndRelease | yes                      | Registry where published versions are permanent (checked for existing versions) |
| `RELEASE_REGISTRY`   | release       | only if publishing       | Registry URL (where releases are published)                                     |
| `S3_S9PKS_BASE_URL`  | release       | only if publishing to S3 | S3 base URL for package uploads. Acts as the on/off switch for registry publish |
