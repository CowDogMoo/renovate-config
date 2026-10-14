# Migrating repos onto the shared presets

This page records the Renovate inventory taken on 2026-10-14 and the plan for
moving each repo onto `github>CowDogMoo/renovate-config`. Migrating a repo is
separate work, done in that repo; nothing here changes another repository.

## Inventory

Every non-archived repo in CowDogMoo, l50 and dreadnode that the captain's
account can read was probed for `renovate.json`, `renovate.json5`,
`.github/renovate.json`, `.github/renovate.json5`, `.gitlab/renovate.json`,
`.renovaterc*` and `.github/workflows/renovate.y*ml` on its default branch.
That covered 230 repos (32 CowDogMoo, 165 l50, 33 dreadnode) and found 41 with
a Renovate config.

| Group | Repos | Plan |
| --- | --- | --- |
| Owned, not forks | 25 (13 CowDogMoo, 12 l50) | Migrate (table below) |
| Forks | 10 | Not migrated: a fork keeps its upstream's config so syncs stay clean |
| dreadnode | 7 | Not migrated: no write access |

### What the 25 owned configs share

Counts are out of 25 `.github/renovate.json5` files.

| Setting | Repos | In `default.json` |
| --- | --- | --- |
| `:enablePreCommit`, `pre-commit.enabled` | 25 | yes |
| `:automergeDigest` | 24 | yes, as part of the minor/patch/digest/pin automerge rule |
| `:semanticCommits`, `commitBodyTable: true` | 23 | yes |
| `:dependencyDashboard` | 23 | yes, through `config:recommended` |
| `:disableRateLimiting`, `helpers:pinGitHubActionDigests` | 22 | yes |
| `suppressNotifications: ["prIgnoreNotification"]` | 22 | yes |
| `config:recommended` (4 more still on the old `config:base`) | 20 | yes |
| `labels: ["renovate"]`, `dependencyDashboardLabels: ["renovate-dashboard"]` | 18 | yes |
| `dependencyDashboardTitle: "Renovate Dashboard 🤖"` (6 drop the emoji) | 17 | yes |
| `platformAutomerge: true` | 16 | yes, with `automergeType: "pr"` |
| `:automergeBranch` | 15 | no: branch automerge pushes straight to `main`, which rulesets and merge queues reject |
| `rebaseWhen` | 16 `conflicted`, 4 `auto`, 3 `behind-base-branch` | `behind-base-branch` (accepted default) |
| `docker:enableMajor` | 10 | no: Docker majors are already enabled by default |
| `minimumReleaseAge: "3 days"` | bcp top level, homeassistant for third-party deps | yes, everywhere |

Automerge was the most copy-pasted and least consistent part: every repo
listed its own per-manager allowlist, and 17 of them automerge GitHub Actions
majors (5 also automerge pre-commit majors, 3 also Galaxy majors). The default preset
replaces all of that with two rules: minor, patch, digest and pin updates
automerge for every manager, and GitHub Actions majors automerge too. Other
majors, including pre-commit and Galaxy, wait for review.

Version envs in workflows were the other repeated block. GitHub code search
finds `GO_VERSION` in at least 10 of these repos' workflows and
`PYTHON_VERSION` in at least 15, across every stack,
so both regex managers live in `default.json`; `TASK_VERSION` appears only in
the three Ansible collections.

### The 25 Renovate workflows

All 25 run `renovatebot/github-action` with autodiscover filtered to the repo
itself. 21 authenticate with a GitHub App token from
`BOT_APP_ID`/`BOT_APP_PRIVATE_KEY` and 20 build the git author from
`BOT_USERNAME`/`BOT_USER_ID`. The other 4 (homeassistant, throwingshade,
goproject, homebrew-brewfile) still use a token secret (`BOT_TOKEN` or
`COWDOGMOO_BOT_TOKEN`) and need the App secrets added before they migrate.
They also differ in:

- Schedule: 20 run Sunday and Wednesday at 00:00 UTC, 2 only on Sunday, bcp
  every 6 hours, squad-agents every 4 hours and squad hourly.
- Tool installs: 15 install Python and/or Go on the runner and `pip`/`go install`
  tools (pre-commit, shfmt, golangci-lint, goimports, gocyclo, gocritic). The
  action runs Renovate inside a container that cannot see runner tools, so none
  of these installs reach Renovate.
- 11 pass the repo's own `.github/renovate.json5` as `configurationFile`, which
  makes Renovate load the repo config a second time as global self-hosted
  config.
- Image: 4 use the `full` image and the rest `latest` or the action default; 19 set
  `RENOVATE_INTERNAL_CHECKS_FILTER=strict`.

The reusable workflow keeps the shared parts, derives the git author from the
App itself (so `BOT_USERNAME` and `BOT_USER_ID` are no longer needed), and
drops the `configurationFile` double load.

## Migration table

"Drop" means the local rule is now covered by a preset and can be deleted.
"Keep" items stay in the repo's `.github/renovate.json5` after `extends`.
"Change" flags behaviour that moves to the accepted defaults.

| Repo | Presets | Remaining local overrides |
| --- | --- | --- |
| CowDogMoo/ansible-collection-workstation | default, ansible-collection | Keep: mise `host_vars` and asdf regex managers; `host_vars` no-automerge + 7-day rule. Drop: gitAuthor, pip patterns, PYTHON/GO/TASK_VERSION managers, automerge rules. Change: Galaxy majors no longer automerge. |
| CowDogMoo/bcp | default, go | Keep: test dependencies group. Drop: go.mod and `.tool-versions` Go regexes (gomod and asdf managers read them natively), GO_VERSION/GOLANGCI_VERSION/go install managers, golang group, concurrency limits. Change: `internalChecksFilter` goes from `flexible` to `strict`. |
| CowDogMoo/homeassistant | default | Keep: its four `.github/renovate/*.json5` includes, schedule, dashboard approval, Home Assistant major approval, vulnerabilityAlerts. Its own 3-day rule becomes redundant. Workflow: add `BOT_APP_ID`/`BOT_APP_PRIVATE_KEY` first. |
| CowDogMoo/squad | default, go | Keep: indirect module handling, golang minor/major no-automerge, moby group, OSV alerts, `constraintsFiltering`. Drop: its `# renovate:` annotation manager for PYTHON_VERSION (the default preset tracks it; keeping both extracts it twice). |
| CowDogMoo/squad-agents | default | Change: pre-commit majors no longer automerge. |
| CowDogMoo/squad-skills | default | Keep: no automerge under `.github/`, GitHub-owned action rule, runner major approval. |
| CowDogMoo/taskfile-templates | default | Keep: `# renovate:` annotation manager for envs other than GO/PYTHON_VERSION. Change: pre-commit majors no longer automerge. |
| CowDogMoo/terraform-cowdogmoo | default, go | Keep: terraform manager, tfstate `ignorePaths`, terraform-docs `postUpgradeTasks` (pass `allowed-post-upgrade-commands` to the workflow), annotation manager. Change: pre-commit majors no longer automerge. |
| CowDogMoo/throwingshade | default, go | Keep: merge-queue `automergeStrategy`, indirect group, test deps group. Drop: GO_VERSION and go install managers. Workflow: add `BOT_APP_ID`/`BOT_APP_PRIVATE_KEY` first. |
| CowDogMoo/Walls-of-Excellence | default | Keep: its `.github/renovate/*.json5` includes, flux/helm/kubernetes patterns, k8s regex manager, `ignorePaths`, k8s-sidecar block. |
| CowDogMoo/warpgate | default, go | Keep: single "go modules" group, jinzhu/copier pin. Drop: indirect-major disable (now in `go`). |
| CowDogMoo/warpgate-mcp-server | default, go | Keep: test deps group. Drop: go.mod regex, go install manager. |
| CowDogMoo/warpgate-templates | default | Keep: WARPGATE_VERSION manager. Drop: warpgate minor/patch automerge. |
| l50/ansible-collection-arsenal | default, ansible-collection | Keep: roles `defaults/main.yml` asdf regex. Change: Galaxy majors no longer automerge. |
| l50/ansible-collection-bulwark | default, ansible-collection | None. Change: Galaxy majors no longer automerge. |
| l50/awsutils | default, go | Drop: own `autoMerge.json5` include, `config:base`, deprecated `matchPackagePatterns`, go install manager. |
| l50/bbh-toolkit | default | None. Change: pre-commit majors no longer automerge. |
| l50/dotfiles | default | Keep: npm and `ignorePaths` patterns, shell/Taskfile/test groups, `files/default-*` regex managers, Galaxy git-refs manager. Change: pre-commit majors no longer automerge. |
| l50/goproject | default, go | Drop: `config:base`, `platform`/`onboarding`/`requireConfig` (global-only settings), legacy `regexManagers`. Workflow: add `BOT_APP_ID`/`BOT_APP_PRIVATE_KEY` first. |
| l50/homebrew-brewfile | default, go | Keep: `requirements.yaml` git-tags manager (rewritten as `customManagers`). Drop: same legacy keys as goproject. Workflow: add `BOT_APP_ID`/`BOT_APP_PRIVATE_KEY` first. |
| l50/iem-decibel-monitor | default, rust | Keep: monthly Actions digest batching. Drop: RUST_VERSION manager, rust-toolchain group, lock file maintenance, the automerge catch-alls. |
| l50/personal-agents | default, go | Keep: annotation manager for `run:` pins and envs other than GO/PYTHON_VERSION. |
| l50/personal-skills | default | Keep: skill-evals workflow no-automerge. |
| l50/signal-triage | default, go | None. |
| l50/terraform-cowdogmoo | default, go | Stale copy of CowDogMoo/terraform-cowdogmoo (last push 2025-05); consider archiving instead. |

Every migrated repo also swaps its workflow for the caller in the README,
which moves it to the 6-hourly schedule.

### Not migrated

| Repo | Reason |
| --- | --- |
| dreadnode/AIRTBench-Code, ares, DreadGOAD, dyana, rigging, robopages-cli, sdk | No write access |
| l50/ares, l50/DreadGOAD | Forks of dreadnode repos; they track upstream |
| CowDogMoo/hass-dyson, CowDogMoo/mealie, l50/asdf-golang, l50/ansible-atomic-red-team, l50/caldera-security-tests, l50/ForgeArmory, l50/go-expect, l50/home-ops | Forks; they track upstream |
