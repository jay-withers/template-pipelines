# template-pipelines

Reusable GitHub Actions workflows, called from other repositories via `workflow_call`.

| Workflow | Purpose |
| --- | --- |
| [`terraform.yml`](.github/workflows/terraform.yml) | Terraform CI: credential-free init, validate and optional `terraform test` across one or more root modules |
| [`pre-commit.yml`](.github/workflows/pre-commit.yml) | Runs all pre-commit hooks against the full repository |
| [`release.yml`](.github/workflows/release.yml) | Semver tagging on CD: bumps from conventional commits, pushes the tag, creates a GitHub release |

## Terraform CI

Credential-free validation of every root module in the calling repository: `terraform init -backend=false` then `terraform validate`, one matrix leg per directory, plus an optional `terraform test` job for repos whose tests mock their providers.

There is no plan or apply job, deliberately. A plan needs Azure credentials and only succeeds once the configuration's dependencies already exist — root modules resolve each other with data sources, so a plan against a not-yet-applied dependency fails at plan time and reports a CI failure for something that is not a defect. Applies are run by hand against remote state.

`-backend=false` is what keeps this credential-free: a repo whose `backend.tf` points at a real remote state account still validates on a fresh clone with no Azure auth.

```yaml
name: ci-terraform

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  terraform:
    permissions:
      contents: read
      pull-requests: read
    uses: jay-withers/workflows/.github/workflows/terraform.yml@main
    with:
      directories: '["terraform/management", "terraform/connectivity"]'
```

### Terraform inputs

| Input | Default | Description |
| --- | --- | --- |
| `directories` | `["terraform"]` | JSON array of root-module directories to init and validate, relative to the repo root. One matrix leg each |
| `test-directories` | `[]` | JSON array of directories to run `terraform test` in. Empty skips the test job |
| `runs-on` | `ubuntu-latest` | Runner label for every job |

Notes:

- **The required status check is `<caller job id> / Terraform`** — for the example above, `terraform / Terraform`. A reusable workflow's checks are namespaced by the calling job's id, the same way `pre-commit.yml` reports as `pre-commit / Pre-commit`. Update branch protection when adopting this, or the required check hangs pending forever.
- The caller must grant the calling job `pull-requests: read`; the path filter that decides whether the PR touches Terraform runs inside this workflow.
- Path filtering lives inside this workflow rather than on the caller's trigger, so the workflow always runs and the gate job always reports. A workflow skipped by a top-level paths filter leaves its required check pending and blocks the merge. The filter matches `terraform/**`, `.terraform-version` and `.github/workflows/ci-terraform.yml`.
- The gate job treats *skipped* as success, so a PR touching no Terraform is not blocked.
- The Terraform version comes from a `.terraform-version` file (the [tfenv](https://github.com/tfutils/tfenv) convention) — looked up in each directory first, then the repo root. Renovate's built-in `terraform-version` manager (part of `config:recommended`) keeps it bumped.
- `fmt`, TFLint and Checkov are not run here — they belong to `pre-commit.yml`.

## Pre-commit CI

Runs `pre-commit run --all-files` with the hook environments cached. The calling repository must contain a `.pre-commit-config.yaml`.

```yaml
name: Pre-commit

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  pre-commit:
    uses: jay-withers/template-pipelines/.github/workflows/pre-commit.yml@main
```

### Terraform hooks

Set `terraform: true` to install the toolchain that Terraform pre-commit hooks (`terraform_fmt`, `terraform_tflint`, `terraform_docs`, `checkov`) expect on the runner — Terraform, TFLint (with plugins initialised), terraform-docs, Checkov, and Node.js. The toolchain is opt-in so non-Terraform repos keep a lean Python-only run.

```yaml
jobs:
  pre-commit:
    uses: jay-withers/template-pipelines/.github/workflows/pre-commit.yml@main
    with:
      terraform: true
```

The Terraform version resolves from the `terraform-version` input, falling back to a `.terraform-version` file, then `latest`. `actionlint` and `gitleaks` need no extra install here — they run as pre-commit-managed hook repos from [`.pre-commit-config.yaml`](.pre-commit-config.yaml).

### Pre-commit inputs

| Input | Default | Description |
| --- | --- | --- |
| `python-version` | `3.12` | Python version used to run pre-commit |
| `terraform` | `false` | Install the Terraform toolchain (Terraform, TFLint, terraform-docs, Checkov, Node) for Terraform hooks |
| `terraform-version` | `""` | Terraform version to install; if empty, read from `.terraform-version`, falling back to `latest` |
| `node-version` | `24` | Node.js version to install when `terraform` is enabled |
| `terraform-docs-version` | `0.24.0` | terraform-docs version to install (without leading `v`) when `terraform` is enabled |
| `checkov-version` | `3.3.6` | Checkov version to install when `terraform` is enabled |
| `tflint-config` | `terraform/.tflint.hcl` | Path to the TFLint config used to initialise plugins when `terraform` is enabled |

## Semver release tagging

On each push to the default branch, computes the next semver from [conventional commits](https://www.conventionalcommits.org/) since the last tag (`fix:` → patch, `feat:` → minor, `BREAKING CHANGE`/`!` → major), pushes the tag, and creates a GitHub release with a generated changelog. Commits with no conventional prefix fall back to `default-bump`.

```yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    uses: jay-withers/template-pipelines/.github/workflows/release.yml@main
    with:
      update-major-tag: true
```

Typically chained after the Terraform apply job with `needs:`, so a release is only cut when the deploy succeeds.

### Release inputs

| Input | Default | Description |
| --- | --- | --- |
| `default-bump` | `patch` | Bump when no conventional commit is found (`major`, `minor`, `patch`, or `false` to skip tagging) |
| `release` | `true` | Create a GitHub release for the new tag |
| `update-major-tag` | `false` | Force-move the major tag (e.g. `v1`) to the new release — useful for action/workflow repos |
| `dry-run` | `false` | Compute the next version without tagging |

Outputs `tag` (e.g. `v1.4.2`) and `version` (`1.4.2`) for downstream jobs.

## Repo CI

This repo dogfoods its own reusable workflows by calling them with local paths (no `@ref` needed). These callers are prefixed `_self_` to distinguish them from the public reusable workflows — they are internal and not meant to be consumed by other repos:

- [`_self_ci.yml`](.github/workflows/_self_ci.yml) calls [`pre-commit.yml`](.github/workflows/pre-commit.yml) on every push and pull request — the hooks in [`.pre-commit-config.yaml`](.pre-commit-config.yaml) cover actionlint, gitleaks secret scanning and file hygiene.
- [`_self_cd.yml`](.github/workflows/_self_cd.yml) calls [`release.yml`](.github/workflows/release.yml) on pushes to `main` with `update-major-tag: true`, so callers can pin to `@v1`.
- Every external action is pinned to a full commit SHA with the version in a trailing comment. A pre-commit hook (`pin-github-actions`) enforces this in CI, and Renovate keeps the pins up to date.
- Dependency updates come from the shared presets in [`template-renovate`](https://github.com/jay-withers/template-renovate); [`renovate.json`](renovate.json) just extends them, so policy changes land centrally.

## Development

Open the repo in the [dev container](.devcontainer/devcontainer.json) (or locally with `make install`) to get the git hooks set up. Useful targets:

```sh
make install     # install pre-commit and the git hooks
make lint        # actionlint + shellcheck (falls back to docker if not installed)
make pre-commit  # run all pre-commit hooks against the full repo
```

The same hooks run in CI via the repo's own [pre-commit workflow](.github/workflows/pre-commit.yml).

## Versioning

Callers should pin to a tag (`@v1`) or commit SHA rather than `@main` once this repo is tagged, so pipeline changes roll out deliberately.
