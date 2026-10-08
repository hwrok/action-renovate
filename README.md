# action-renovate

Composite action that runs self-hosted [Renovate](https://docs.renovatebot.com/) against the calling repo, authenticated as a GitHub App, plus a shared preset (`default.json`).

Each repo owns its own workflow (triggers, cadence, runner), so what runs and when is visible from inside the repo.

## Quickstart

### 1. Workflow

`.github/workflows/renovate.yml`:

```yaml
name: renovate

on:
  schedule:
    - cron: '17 6 1 * *' # 1st of the month, 06:17
      timezone: 'America/Chicago'
  # when renovate updates, merge it first and it will apply new config immediately
  push:
    branches: [main]
    paths:
      - .github/renovate.json
      - .github/workflows/renovate.yml
  workflow_dispatch:
    inputs:
      dry-run:
        description: 'dry run mode'
        type: choice
        options: [none, extract, lookup, full]
        default: none
      log-level:
        type: choice
        options: [info, debug]
        default: info

permissions:
  contents: read

jobs:
  renovate:
    concurrency:
      group: renovate
      cancel-in-progress: false
    runs-on: ubuntu-latest
    steps:
      - uses: hwrok/action-renovate@<sha> # v1.x.y
        with:
          app-client-id: ${{ vars.APP_CLIENT_ID }}
          app-private-key: ${{ secrets.APP_PRIVATE_KEY }}
          dry-run: ${{ inputs.dry-run != 'none' && inputs.dry-run || '' }}
          log-level: ${{ inputs.log-level || 'info' }}
```

### 2. Config

`.github/renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>hwrok/action-renovate#v1.x.y"]
}
```

Pin the preset to the same version as the action. Renovate bumps both together in one `action-renovate` PR, with no cooldown.

A repo without a renovate config is skipped (`requireConfig: required`, no onboarding PRs).

## Cadence

- The caller's cron is the whole schedule. Manual dispatch runs any time.
- Pick an off-peak minute - GitHub delays scheduled runs most at the top of the hour.
- Cooldowns (`minimumReleaseAge`) still apply on every run, so a manual run can legitimately open nothing.

## Inputs

| Input              | Description                                                                          | Default |
| ------------------ | ------------------------------------------------------------------------------------ | ------- |
| `app-client-id`    | GitHub App client id                                                                 |         |
| `app-private-key`  | GitHub App private key                                                               |         |
| `dry-run`          | `extract`, `lookup`, `full`; empty = real run                                        | `''`    |
| `log-level`        | Renovate log level                                                                   | `info`  |
| `repository-cache` | Persist Renovate's repository cache via `actions/cache`                              | `true`  |
| `fail-on`          | Lowest report problem level that fails the job: `warn`, `error`, or `none`           | `warn`  |
| `ignore-problems`  | Newline-separated regexes; matching report problems don't fail the job, at any level | `''`    |

Renovate exits 0 on plenty of per-repo failures (bad token, external host errors), so `fail-on` reads Renovate's JSON report instead of trusting the exit code.

Known-harmless warnings can be skipped per repo:

```yaml
ignore-problems: |
  ^Could not determine resolved version after updating package file
```

## Preset

`default.json`, extending `config:recommended`:

- Actions pinned to SHA with a `# vX.Y.Z` comment
- Exact version pins (`rangeStrategy: pin`), except `engines`, which is left alone
- Groups: `github-actions`, `npm`, `terraform`, `docker`, `go`, and `toolchain`. Majors split into `major-*` PRs
- Go: `go mod tidy` runs after updates, and majors rewrite `/vN` import paths in code (`gomodUpdateImportPaths`). The `go` directive is a floor and is never bumped; the `toolchain` directive is, in `toolchain`
- Cooldowns: major 60d, minor 14d, patch 7d; vulnerability fixes 0d. Versioned runner labels (`ubuntu-24.04`) skip cooldowns; `-latest` is never touched
- Updates without a release date (e.g. ghcr.io images) are held by the cooldown indefinitely (the job lists them as notices). Opt specific deps in per repo with `minimumReleaseAgeBehaviour: "timestamp-optional"`
- Dependency Dashboard off. To rebase/retry a PR, tick its checkbox, then dispatch the workflow

Override anything in the consuming repo's config.

## Requirements

- `ubuntu-*` runner (Renovate runs in Docker, cache step uses `sudo`)
- GitHub App installed on the repo. Permissions:
  - read/write: Checks, Commit statuses, Contents, Issues, Pull requests, Workflows
  - read: Administration, Dependabot alerts, Metadata
- The App's client id and private key stored as a variable and secret (`APP_CLIENT_ID` / `APP_PRIVATE_KEY` above are placeholders)
