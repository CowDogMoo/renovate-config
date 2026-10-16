# Migrating Go repos onto the shared Go CI workflow

This page records the Go CI inventory taken on 2026-10-15 and the plan for
moving each Go repo's pre-commit and test workflows onto
[go-ci.yaml](../.github/workflows/go-ci.yaml). Migrating a repo is separate
work, done in that repo; nothing here changes another repository.

## Why

The Go 1.27.2 bump failed pre-commit in squad, warpgate, awsutils,
signal-triage and others the same way, because each repo installs its own Go
linters. In warpgate the cause was go-critic v0.15.0 built against the
`golang.org/x/tools` version it pins, which cannot type-check Go 1.27 code and
reported false `sloppyTypeAssert` and `caseOrder` findings. The fix was to
build gocritic against the latest x/tools. With one shared workflow that fix,
and the next one, lands in one place.

## Inventory

The default branches of these repos were read for
`.github/workflows/pre-commit.yaml`, `.github/workflows/tests.yaml`,
`.pre-commit-config.yaml` and `go.mod`, plus a code search across CowDogMoo and
l50 for workflows that install gocritic or configs that use
`dnephin/pre-commit-golang`. Nine Go repos have a pre-commit workflow (awsutils,
bcp, homebrew-brewfile, personal-agents, signal-triage, squad, throwingshade,
warpgate, warpgate-mcp-server) and nine have a tests workflow (the same list
without personal-agents, plus goproject).

### What the pre-commit workflows share

| Step | Repos | In `go-ci.yaml` |
| --- | --- | --- |
| Checkout, set up Python, `pip install pre-commit` | all 9 | yes |
| Set up Go with `check-latest` | 7; 2 read `go.mod`, the rest a `GO_VERSION` env or literal | yes, from `go.mod` unless `go-version` is set |
| `go install` shfmt | all 9 | yes, always |
| `go install` goimports, gocyclo, gocritic, golangci-lint | 6, plus 1 through `mage installDeps` | yes, with `go-linters: true` (default) |
| golangci-lint version | `latest` in warpgate through `golangci-lint-action`, pinned in 5 (v2.11.4 to v2.14.0) | `golangci-lint-version`, default `latest`, always `go install` |
| apt `build-essential btrfs-progs libgpgme-dev libbtrfs-dev pkg-config` | squad, warpgate; personal-agents installs two of them | `apt-packages` |
| GoReleaser install for the `goreleaser-check` hook | squad, warpgate | `go-installs` |
| Run hooks: `task -y pre-commit:run-hooks` | 3 | default `pre-commit-command`; that task runs `pre-commit run --all-files --show-diff-on-failure` |
| Run hooks: `task -y pre-commit:run-pre-commit` | bcp, throwingshade | not kept: it also runs `pre-commit autoupdate` and `pre-commit clean` in CI, and Renovate already bumps hook revisions |
| Run hooks: `pre-commit run --all-files` | 2 | default `pre-commit-command` |
| Run hooks: `mage runPreCommit`, after a `go mod tidy` commit step | awsutils, homebrew-brewfile | `pre-commit-command`; the commit step stays local or goes away |

The five repos that install go-task use it only to call the remote
`pre-commit` Taskfile, which wraps `pre-commit` commands, so the shared
workflow runs pre-commit directly and installs no Task.

### What the test workflows share

| Step | Repos | In `go-ci.yaml` |
| --- | --- | --- |
| Checkout and set up Go | all 9 | yes |
| Set a git identity for tests that commit | 6 | yes |
| `bash .hooks/run-go-tests.sh coverage` | 7 | default `test-command` |
| Upload `coverage-all.out` to Codecov with `CODECOV_TOKEN` | 5 | yes, when the secret is passed |
| Upload to Coveralls (`actions-goveralls`) | awsutils, goproject | no: stays in the repo |
| AWS OIDC credentials before tests | awsutils | no: stays in the repo |
| `go install` the warpgate CLI before tests | warpgate-mcp-server | `test-go-installs` |

## Migration table

Each migrated repo replaces `pre-commit.yaml` and `tests.yaml` with the caller
in the README. "Inputs" lists what to pass beyond the defaults. "Keep" is a
job that stays in the repo next to the `go-ci` job. "Change" flags behaviour
that moves to the shared defaults.

| Repo | Inputs | Keep / Change |
| --- | --- | --- |
| CowDogMoo/squad | `apt-packages: build-essential btrfs-progs libgpgme-dev libbtrfs-dev pkg-config`, `go-installs: github.com/goreleaser/goreleaser/v2@latest`, `golangci-lint-version: v2.14.0` if the pin should stay, `python-version` | Keep: govulncheck, goreleaser, semgrep workflows. |
| CowDogMoo/warpgate | same `apt-packages` and `go-installs` as squad | Keep: the `schema-validation` job. |
| CowDogMoo/warpgate-mcp-server | `golangci-lint-version: v2.12.2` if the pin should stay, `test-go-installs: github.com/cowdogmoo/warpgate/v3/cmd/warpgate@main`, no `CODECOV_TOKEN` (it uploads nothing today) | Change: Go comes from `go.mod` (1.24) instead of the `GO_VERSION` env (1.25); bump `go.mod` or pass `go-version`. |
| CowDogMoo/throwingshade | `golangci-lint-version: v2.14.0` and `gocritic-version: v0.15.0` if the pins should stay | Change: no `pre-commit autoupdate` in CI. |
| CowDogMoo/bcp | `golangci-lint-version: v2.14.0` if the pin should stay | Change: no `pre-commit autoupdate` in CI. |
| l50/signal-triage | `golangci-lint-version: v2.11.4` if the pin should stay | Change: Go comes from `go.mod` (1.23) instead of the `GO_VERSION` env (1.26.9); bump `go.mod` or pass `go-version`. |
| l50/awsutils | `go-installs: github.com/magefile/mage@latest`, `pre-commit-command: mage installDeps && mage runPreCommit`, `tests: false` | Keep: the tests job (AWS OIDC, Coveralls). Change: Go comes from `go.mod` (1.25) instead of a literal 1.22; the `go mod tidy` commit step is dropped, so a stale `go.sum` fails the hook instead of being patched in CI. |
| l50/homebrew-brewfile | `go-version-file: magefiles/go.mod`, `go-linters: false`, `go-installs: github.com/magefile/mage@latest`, `pre-commit-command: mage installDeps && mage runPreCommit`, `tests: false` | Keep: the macOS Brewfile test jobs. |
| l50/personal-agents | `apt-packages: libgpgme-dev libbtrfs-dev`, `go-linters: false`, `tests: false` | Keep: validate-agents. |
| l50/goproject | `pre-commit: false`, `test-command: bash .hooks/go-unit-tests.sh coverage`, `coverage-file: ''` | Keep: Coveralls upload and failure issue as local jobs. Go 1.20 in `go.mod`, so bump it first. |

A pinned `golangci-lint-version` passed as an input is not tracked by
Renovate: the `go` preset tracks `GOLANGCI_VERSION` envs, and env values do
not reach a reusable workflow. Leave the input at `latest` unless a pin is
needed.

### Not migrated

| Repo | Reason |
| --- | --- |
| CowDogMoo/terraform-cowdogmoo, l50/terraform-cowdogmoo | Terraform pre-commit (tflint, tfsec, checkov, OIDC), not the Go lint setup |
| l50/keeper | No push since 2022 |
