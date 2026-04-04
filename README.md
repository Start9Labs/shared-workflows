# shared-workflows

Reusable GitHub Actions workflows for building, tagging, and releasing StartOS service packages (.s9pk).

## How It Works

The CI pipeline has three stages, each triggered automatically by the previous:

```
PR opened/updated ──> Build
PR merged to master ──> Version check ──> Tag ──> Build ──> Release ──> Publish
```

### 1. Build on PR

When a pull request is opened or updated against `master`, **build.yml** builds the `.s9pk` to verify it compiles. Draft PRs are skipped. If a new commit arrives on the PR, the previous build is cancelled and a fresh one starts.

### 2. Tag and release on merge

When a PR is merged to `master`, **tagAndRelease.yml** runs:

1. Extracts the package ID and current version from source (via the **extract-version** action)
2. Queries the production registry to check if that version already exists
3. If it exists, the workflow **fails** — you forgot to bump the version
4. If it doesn't exist, deletes any prior GitHub release and tag for that version, then creates a fresh tag
5. Calls **release.yml** to build, create a GitHub release, and publish to the test registry

If the production registry is unreachable, the workflow fails rather than silently proceeding.

### 3. Manual release

You can also cut a tag manually and push it to trigger **release.yml** directly, bypassing the version check. This is useful for re-releasing or testing.

## Version Encoding

Versions are extracted from `startos/versions/index.ts` by the **extract-version** composite action. Tags strip the flavor prefix and replace `:` with `_`:

| Version | Tag | Release Name |
|---|---|---|
| `2.0.0:2` | `v2.0.0_2` | `2.0.0:2` |
| `29.3:5` | `v29.3_5` | `29.3:5` |
| `#knots:29.3:2-beta.1` | `v29.3_2-beta.1` | `#knots:29.3:2-beta.1` |

The flavor is omitted from tags because each repo only produces one flavor. The full version (including flavor) is used as the GitHub release name.

## Workflows

### build.yml

Builds the service package(s). Does not publish or create releases. If `DEV_KEY` is not provided, generates a temporary signing key.

### tagAndRelease.yml

Checks the current version against a production registry, creates a release tag, then calls **release.yml** to build and publish.

### release.yml

Builds the service package(s), creates a GitHub release, and publishes to a registry. Can be called by tagAndRelease or triggered directly by a manual tag push.

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
    branches: ['master']
    paths-ignore: ['*.md']

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

```yaml
name: Tag and Release

on:
  push:
    branches: ['master']
    paths-ignore: ['*.md']

jobs:
  tag-and-release:
    uses: start9labs/shared-workflows/.github/workflows/tagAndRelease.yml@master
    with:
      PROD_REGISTRY: ${{ vars.PROD_REGISTRY }}
      # FREE_DISK_SPACE: true
      REGISTRY: ${{ vars.TEST_REGISTRY }}
      S3_S9PKS_BASE_URL: ${{ vars.S3_S9PKS_BASE_URL }}
    secrets:
      DEV_KEY: ${{ secrets.DEV_KEY }}
      S3_ACCESS_KEY: ${{ secrets.S3_ACCESS_KEY }}
      S3_SECRET_KEY: ${{ secrets.S3_SECRET_KEY }}
    permissions:
      contents: write
```

### 3. Release on manual tag (`.github/workflows/release.yml`, optional)

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*'

jobs:
  release:
    uses: start9labs/shared-workflows/.github/workflows/release.yml@master
    with:
      # FREE_DISK_SPACE: true
      REGISTRY: ${{ vars.TEST_REGISTRY }}
      S3_S9PKS_BASE_URL: ${{ vars.S3_S9PKS_BASE_URL }}
    secrets:
      DEV_KEY: ${{ secrets.DEV_KEY }}
      S3_ACCESS_KEY: ${{ secrets.S3_ACCESS_KEY }}
      S3_SECRET_KEY: ${{ secrets.S3_SECRET_KEY }}
    permissions:
      contents: write
```

## Required Secrets and Variables

### Repository Secrets

| Secret | Used by | Description |
|---|---|---|
| `DEV_KEY` | build, release | Developer signing key |
| `S3_ACCESS_KEY` | release | S3 access key for package uploads |
| `S3_SECRET_KEY` | release | S3 secret key for package uploads |

### Repository Variables

| Variable | Used by | Description |
|---|---|---|
| `PROD_REGISTRY` | tagAndRelease | Production registry URL (checked for existing versions) |
| `TEST_REGISTRY` | release | Test registry URL (where releases are published) |
| `S3_S9PKS_BASE_URL` | release | S3 base URL for package uploads |
