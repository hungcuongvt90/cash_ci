# Cashsmartify CI

This repository contains the public GitHub Actions workflow triggered by Jenkins.

The workflow follows the current Jenkins usage and supports:

- production and staging image names
- Build+Deploy, Build Only, and Deploy Only action values
- BE, API, Worker, Solver, UI, dependency-cache, and unit-test targets
- API test options and Cargo parallelism
- source branch and commit verification

The workflow performs the Docker build/test work only. Kubernetes deployment remains in Jenkins.

## ARM64 build contract

Both staging and production run on ARM64. Every application image build in this repository uses exactly:

```text
linux/arm64
```

The GitHub-hosted runner uses QEMU and Docker Buildx. No `linux/amd64` image is built or published.

## Workflow

```text
.github/workflows/build-images.yml
```

## Jenkins dispatch contract

Jenkins triggers the workflow with `workflow_dispatch` inputs matching the current Jenkins parameters:

| Input | Required | Example | Meaning |
|---|---:|---|---|
| `DEPLOY_ENV` | yes | `staging` | `staging` or `production` |
| `DEPLOY_TARGET` | yes | `BE` | Current Jenkins build/test target |
| `PIPELINE_ACTION` | yes | `Build+Deploy` | `Build+Deploy`, `Build Only`, or `Deploy Only` |
| `SOURCE_REPO` | yes | `hungcuongvt90/mcc` | Source repository |
| `BRANCH` | yes | `develop` | Branch, tag, or commit to build |
| `SOURCE_SHA` | no | `40-character SHA` | Commit resolved by Jenkins; the workflow verifies it |
| `IMAGE_OWNER` | yes | `hungcuongvt90` | GHCR owner |
| `IMAGE_PREFIX` | no | `staging-cashsmartify` | Optional environment-specific image prefix |
| `IMAGE_TAG` | no | `latest` | Image tag; use `latest` |
| `API_RUN_UNITTEST` | yes | `false` | Run API tests before BE/API build |
| `API_BYPASS_UNITTEST_FAILURE` | yes | `false` | Continue optional BE/API build after API test failure |
| `CARGO_BUILD_JOBS` | yes | `4` | Cargo parallelism inside the ARM64 build container |
| `BACKUP_FILE` | no | `backup.sql` | Compatibility input; DB actions remain in Jenkins |
| `REQUEST_ID` | yes | `jenkins-1234` | Jenkins correlation identifier |

Jenkins should send this payload to the `cash_ci` workflow:

```json
{
  "ref": "master",
  "inputs": {
    "DEPLOY_ENV": "staging",
    "DEPLOY_TARGET": "BE",
    "PIPELINE_ACTION": "Build+Deploy",
    "SOURCE_REPO": "hungcuongvt90/mcc",
    "BRANCH": "develop",
    "SOURCE_SHA": "SOURCE_COMMIT_SHA",
    "IMAGE_OWNER": "hungcuongvt90",
    "IMAGE_PREFIX": "staging-cashsmartify",
    "IMAGE_TAG": "latest",
    "API_RUN_UNITTEST": "false",
    "API_BYPASS_UNITTEST_FAILURE": "false",
    "CARGO_BUILD_JOBS": "4",
    "BACKUP_FILE": "backup.sql",
    "REQUEST_ID": "jenkins-BUILD_NUMBER"
  }
}
```

These inputs are non-secret build metadata. Jenkins must never send private keys, PATs, or registry passwords as workflow inputs.

The GitHub API token used by Jenkins needs permission to dispatch workflows in this repository. Keep that token in Jenkins Credentials:

```text
mcc-ci-dispatch-token
```

## Target behavior

| Jenkins target | GitHub CI behavior |
|---|---|
| `BE` | Build Rust dependency cache, API, and Worker images; optionally run API tests |
| `BE Build` | Build Rust dependency cache, API, and Worker images |
| `API` | Build Rust dependency cache and API image; optionally run API tests |
| `Worker` | Build Rust dependency cache and Worker image |
| `Solver` | Build Solver image |
| `UI` | Build UI image |
| `Deps` | Build Rust dependency cache image |
| `BE Unit Test` | Run API unit tests |
| `Solver Unit Test` | Run Solver Pytest suite |
| `FE Unit Test` | Run UI Vitest suite |
| `All Unit Tests` | Run API, Solver, and UI unit tests |

`Deploy Only` skips GitHub image work. Jenkins remains responsible for deploying the currently published `latest` images.

Database backup, restore, and sync actions remain entirely in Jenkins.

## Source repository access

The source repository is under the same GitHub account, but GitHub Actions credentials are still scoped to the `cash_ci` repository.

- If `hungcuongvt90/mcc` is public, no source secret is required.
- If `hungcuongvt90/mcc` is private, configure a read-only `SOURCE_REPO_READ_TOKEN` secret in this repository. The token needs Contents: Read access to `mcc`.
- A Jenkins SSH key is not automatically available inside GitHub Actions. Do not pass its private half through workflow inputs or environment variables.

The workflow uses:

```yaml
token: ${{ secrets.SOURCE_REPO_READ_TOKEN || github.token }}
```

## GHCR access

The default login uses the workflow `GITHUB_TOKEN`:

```yaml
permissions:
  contents: read
  packages: write
```

This works when `IMAGE_OWNER` is the owner of this repository (`hungcuongvt90`) and package permissions allow the workflow to publish.

For another GHCR owner, configure these secrets in `cash_ci`:

```text
REGISTRY_USERNAME
REGISTRY_PASSWORD
```

Do not pass registry credentials from Jenkins as dispatch inputs.

## Published image names

Production uses the `cashsmartify` prefix:

```text
ghcr.io/hungcuongvt90/cashsmartify-api:latest
ghcr.io/hungcuongvt90/cashsmartify-worker:latest
ghcr.io/hungcuongvt90/cashsmartify-solver:latest
ghcr.io/hungcuongvt90/cashsmartify-ui:latest
```

Staging uses the `staging-cashsmartify` prefix:

```text
ghcr.io/hungcuongvt90/staging-cashsmartify-api:latest
ghcr.io/hungcuongvt90/staging-cashsmartify-worker:latest
ghcr.io/hungcuongvt90/staging-cashsmartify-solver:latest
ghcr.io/hungcuongvt90/staging-cashsmartify-ui:latest
```

The environment prefix prevents staging and production from overwriting each other's images while retaining the `latest` tag used by the current Kubernetes deployments.

Rust dependency images are fingerprinted from the Cargo manifests, toolchain, and Dockerfiles. If the fingerprinted dependency image already exists for the same environment, the workflow reuses it and skips the dependency-image build. A dependency change creates a new cache image automatically.

A `build-result-<REQUEST_ID>` artifact contains the source revision, image names, platform, and image digests for Jenkins diagnostics.
