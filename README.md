# renovate-config

Shared Renovate presets, a reusable Renovate workflow and a reusable Go CI
workflow for CowDogMoo and l50 repos. A repo extends the presets it needs and
swaps its copy-pasted Renovate, pre-commit and test workflows for short caller
workflows. [docs/migration.md](docs/migration.md) has the Renovate inventory
and migration plan; [docs/go-ci-migration.md](docs/go-ci-migration.md) has the
same for Go CI.

## Use it

`.github/renovate.json5`:

```json5
{
  $schema: 'https://docs.renovatebot.com/renovate-schema.json',
  extends: [
    'github>CowDogMoo/renovate-config',
    'github>CowDogMoo/renovate-config:go',
  ],
}
```

`.github/workflows/renovate.yaml`:

```yaml
---
name: Renovate

on:
  schedule:
    - cron: "0 */6 * * *"
  push:
    branches: ["main"]
    paths:
      - .github/renovate.json5
  workflow_dispatch:
    inputs:
      dry-run:
        description: Dry run
        type: boolean
        default: false
      log-level:
        description: Log level
        type: choice
        default: info
        options: [debug, info]

concurrency:
  group: ${{ github.workflow }}
  cancel-in-progress: false

permissions: {}

jobs:
  renovate:
    uses: CowDogMoo/renovate-config/.github/workflows/renovate.yaml@main
    with:
      dry-run: ${{ inputs.dry-run || false }}
      log-level: ${{ inputs.log-level || 'info' }}
    secrets:
      BOT_APP_ID: ${{ secrets.BOT_APP_ID }}
      BOT_APP_PRIVATE_KEY: ${{ secrets.BOT_APP_PRIVATE_KEY }}
```

Pass the two secrets explicitly. `secrets: inherit` only works inside one
organization, and l50 repos call this workflow from CowDogMoo. The GitHub App
behind `BOT_APP_ID` must be installed on the calling repo. The workflow asks
GitHub for the App's bot user, so `BOT_USERNAME` and `BOT_USER_ID` are no
longer needed.

## Presets

Preset files are JSON with comments (`.json`), because Renovate looks up
`default.json` for `github>owner/repo` and `<name>.json` for
`github>owner/repo:<name>`.

| Preset | Extend with | What it does |
| --- | --- | --- |
| [default](default.json) | `github>CowDogMoo/renovate-config` | Base for every repo, described below |
| [go](go.json) | `github>CowDogMoo/renovate-config:go` | `go mod tidy` and import path updates after bumps, bumps the `go` directive, skips majors of indirect modules, tracks `GOLANGCI_VERSION` envs and pinned `go install` lines in `*.sh` |
| [ansible-collection](ansible-collection.json) | `github>CowDogMoo/renovate-config:ansible-collection` | Groups Galaxy collections and roles into one PR and tracks `TASK_VERSION` envs |
| [rust](rust.json) | `github>CowDogMoo/renovate-config:rust` | Groups Rust toolchain bumps, tracks `RUST_VERSION` envs, automerges `Cargo.lock` maintenance |

The default preset:

- extends `config:recommended`, `:semanticCommits`, `:enablePreCommit`,
  `:disableRateLimiting` and `helpers:pinGitHubActionDigests`
- automerges minor, patch, digest and pin updates for every manager through PRs
  with GitHub's native automerge, so required checks gate every merge
- automerges GitHub Actions majors the same way; majors for every other
  manager, including pre-commit and Galaxy, wait for review
- holds each release for 3 days (`minimumReleaseAge`) and opens no branch until
  that has passed (`internalChecksFilter: strict`). Releases without a
  timestamp are let through and reported on the dashboard
- rebases a branch whenever it falls behind its base branch. The value is set
  explicitly because merge-queue repos turn `auto` into `conflicted`, which
  never rebases a branch that is only behind
- blocks Python 3.15 until ansible-core supports it as a controller Python
- puts every Go toolchain bump in one `golang` PR: the go.mod `go` and
  `toolchain` lines, `GO_VERSION` envs in workflows and `setup-go`
  `go-version` values
- puts `.hooks/requirements.txt` updates in one `hook requirements` PR
- tracks `GO_VERSION` and `PYTHON_VERSION` envs in workflows
- uses the `renovate` label, the `renovate-dashboard` dashboard label and the
  "Renovate Dashboard 🤖" dashboard title

## Override locally

Rules in the repo's own `.github/renovate.json5` apply after the presets, so a
later `packageRules` entry wins. Keep only what is specific to the repo.

```json5
{
  extends: ['github>CowDogMoo/renovate-config'],
  packageRules: [
    {
      description: 'Pinned host tools wait a week and merge by hand',
      matchFileNames: ['playbooks/workstation/host_vars/**'],
      minimumReleaseAge: '7 days',
      automerge: false,
    },
    {
      description: 'Automerge pre-commit majors in this repo',
      matchManagers: ['pre-commit'],
      matchUpdateTypes: ['major'],
      automerge: true,
    },
  ],
}
```

To drop a single preset rule, override what it sets: set `rebaseWhen`, set
`minimumReleaseAge: null`, or add a later rule for `python` with
`allowedVersions: '/.*/'` to allow Python 3.15. Do not track `GO_VERSION` or
`PYTHON_VERSION` again with a local `# renovate:` annotation manager, or
Renovate extracts the same value twice.

## Renovate workflow inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `dry-run` | `false` | Run with `dryRun=full`: no branches or PRs |
| `log-level` | `info` | Renovate `LOG_LEVEL` |
| `renovate-version` | empty | Renovate image tag; empty uses the major pinned by `renovatebot/github-action` |
| `allowed-post-upgrade-commands` | empty | JSON array of regexes for `allowedPostUpgradeCommands`, for repos with `postUpgradeTasks` |
| `go-version` | empty | Install Go on the runner |
| `go-installs` | empty | Newline-separated `module@version` list to `go install` |
| `python-version` | empty | Install Python on the runner |
| `pip-installs` | empty | Space-separated packages to `pip install` |

The tool install inputs exist because the old per-repo workflows installed Go,
Python, pre-commit and Go linters on the runner. Renovate runs inside the
action's container and cannot see those tools, so leave these inputs empty
unless a step you add around the workflow needs them. Renovate installs the
tools it needs inside the container on its own.

Secrets: `BOT_APP_ID` and `BOT_APP_PRIVATE_KEY`, both required.

## Go CI workflow

[.github/workflows/go-ci.yaml](.github/workflows/go-ci.yaml) runs the
pre-commit and test jobs every Go repo used to carry its own copy of. It
installs Go from the repo's `go.mod`, builds shfmt and the Go linters the
`dnephin/pre-commit-golang` hooks call (goimports, gocyclo, gocritic,
golangci-lint) from source, runs `pre-commit run --all-files`, then runs
`.hooks/run-go-tests.sh coverage` and uploads `coverage-all.out` to Codecov.

`.github/workflows/go-ci.yaml` in the calling repo:

```yaml
---
name: Go CI

on:
  merge_group:
  pull_request:
    branches: ["main"]
  push:
    branches: ["main"]
  schedule:
    - cron: "0 0 * * 0"
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  go-ci:
    uses: CowDogMoo/renovate-config/.github/workflows/go-ci.yaml@main
    permissions:
      contents: read
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

Pass `CODECOV_TOKEN` explicitly, for the same cross-organization reason as the
Renovate secrets. Without it the tests still run and the upload is skipped.
The checks report as `go-ci / Pre-commit` and `go-ci / Tests`, so update any
ruleset or branch protection that requires the old check names.

Why the tools are built from source: a linter binary built with an older Go,
or against a `golang.org/x/tools` that predates the toolchain, misreads code
written for the new toolchain. go-critic v0.15.0 under Go 1.27 reported false
`sloppyTypeAssert` and `caseOrder` findings because its pinned x/tools could
not type-check Go 1.27 code. The workflow builds every tool with the Go it
just installed and builds gocritic against the latest x/tools, so a Go bump in
`go.mod` rebuilds the linters to match. When the next toolchain bump breaks a
linter, fix it here once.

### Go CI inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `pre-commit` | `true` | Run the pre-commit job |
| `tests` | `true` | Run the tests job |
| `go-version-file` | `go.mod` | File `setup-go` reads the Go version from |
| `go-version` | empty | Go version to install; overrides `go-version-file` |
| `apt-packages` | empty | Space-separated apt packages to install for pre-commit, such as `libgpgme-dev libbtrfs-dev` |
| `go-installs` | empty | Newline-separated `module@version` list to `go install` for pre-commit, such as GoReleaser; `#` starts a comment |
| `test-apt-packages` | empty | Same as `apt-packages`, for the tests job |
| `test-go-installs` | empty | Same as `go-installs`, for the tests job |
| `python-version` | `3.14` | Python for pre-commit |
| `go-linters` | `true` | Install goimports, gocyclo, gocritic and golangci-lint; shfmt is always installed |
| `golangci-lint-version` | `latest` | golangci-lint v2 version, `vX.Y.Z` or `latest` |
| `gocritic-version` | `latest` | go-critic version, `vX.Y.Z` or `latest`; always built against the latest x/tools |
| `pre-commit-command` | `pre-commit run --all-files --show-diff-on-failure --color=always` | Command that runs the hooks |
| `test-command` | `bash .hooks/run-go-tests.sh coverage` | Command that runs the tests |
| `coverage-file` | `coverage-all.out` | Coverage profile to upload; empty skips the upload |

Secrets: `CODECOV_TOKEN`, optional. Only the upload step sees it, not the test
command.

Environment variables set in the caller do not reach a reusable workflow. A
command that needs one sets it inline, such as
`pre-commit-command: TASK_X_REMOTE_TASKFILES=1 task -y pre-commit:run-hooks`.
Jobs the shared workflow does not cover, such as AWS OIDC tests or schema
checks, stay in the calling repo as their own jobs next to `go-ci`.

## This repo

[.github/renovate.json5](.github/renovate.json5) extends the default preset so
this repo's action pins stay current, and
[.github/workflows/renovate-self.yaml](.github/workflows/renovate-self.yaml)
runs it through the reusable workflow every 6 hours. It needs `BOT_APP_ID` and
`BOT_APP_PRIVATE_KEY` set on this repo and the App installed here.

## Validate changes

```bash
for f in default.json go.json ansible-collection.json rust.json .github/renovate.json5; do
  npx --yes --package renovate -- renovate-config-validator --strict --no-global "$f"
done
actionlint
```
