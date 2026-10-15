# Project Instructions

## Project Overview

Library of reusable GitHub Actions workflows (`on: workflow_call`) that the
RTLDEV middleware repositories call for their tests, releases, Dependabot
auto-merge and the daily Node dependency refresh. There is no package manifest
and no product code.

- **Language / runtime:** GitHub Actions workflow YAML
  (`.github/workflows/*.yml`)
- **Default branch:** `main`

## Architecture

- **Callers pin `@main`, and there are no tags or releases.** A merge to
  `main` is live in every caller's next run — treat each change as a release
  to all of them.
- **Every workflow except `self-test-auto-merge-dependabot.yml` is
  `workflow_call` only.** Keep it that way; a direct trigger would run against
  this repository, not a caller.
- **Naming:** mostly `<repository-short-name>-test.yml` and
  `<repository-short-name>-release.yml`; a release workflow runs its own test
  workflow first via `uses: ./.github/workflows/<name>-test.yml`. Exceptions:
  `middleware-blesta-*`, `php-sdk-lint.yml`, `whmcs-php-check.yml`, and
  `playground-repo-test.yml`, which has no release workflow.
- **Runtime versions come from organisation variables**
  (`vars.RTLDEV_MW_CI_NODE_VERSION`, `vars.RTLDEV_MW_CI_PHP_VERSION`, the
  `*_MATRIX` variables, …) — never hardcode a version a variable carries.
- **`pnpm-version` inputs default to empty** so `pnpm/action-setup` resolves
  pnpm from the caller's `devEngines.packageManager`; a caller passes one only
  when its manifest declares none, since both together are a hard error.
- **Auto-merge uses `RTLDEV_MW_CI_TOKEN`, not `GITHUB_TOKEN`**
  (`auto-merge-dependabot-pr.yml`) — a merge made by `GITHUB_TOKEN` starts no
  workflow run, so the push-triggered release would never fire.
- **The daily refresh takes its release-age threshold from the caller's
  `pnpm-workspace.yaml`** (`daily-node-dependency-refresh.yml`) — never add a
  `minimumReleaseAge` or `--cooldown` override here.

## Testing

- `self-test-auto-merge-dependabot.yml` runs on every pull request: actionlint
  over the workflows, then `markdownlint-cli2` over tracked Markdown (default
  rules, so lines wrap at 80 characters), then the auto-merge workflow.
- There is no package manifest, so that job names pnpm explicitly
  (`version: 11`).

### Running tests

```sh
go run github.com/rhysd/actionlint/cmd/actionlint@v1.7.7
pnpm dlx markdownlint-cli2@0.18.1 $(git ls-files '*.md' '*.markdown')
```

actionlint does not prove a workflow behaves; a change is only fully exercised
from a caller.

## Build, CI & Policies

- **Dependabot** (`.github/dependabot.yml`) bumps the GitHub Actions daily with
  `chore(deps)` commits, which the self-test auto-merges.

## Git Conventions

- **Commit messages:** Conventional Commits with **mandatory scope** —
  `<type>(<scope>): <summary>`, scoped by the workflow changed
  (e.g. `fix(go-sdk-test): …`). Never append a `Co-Authored-By:` trailer.
- **Branch creation:** before creating a branch, and again after any PR merges,
  run `git checkout main && git pull --ff-only` first — it brings in the merged
  commits and their release tags. Never branch from a stale local `main`, from
  `origin/main` (it becomes the new branch's upstream) or from another feature
  branch.
- **Branch naming:** prefix with the Jira issue ID —
  `RSRMID-1234-short-description`.
- **Pull requests:** include the Jira issue link in the description; after
  opening, add the PR URL as a comment on the Jira issue.
- **Merging:** rebase-merge (`gh pr merge --rebase`). Squash merges are
  disabled at the repository level.

## Atlassian / JIRA

Work is tracked in **Jira Cloud**, project `RSRMID` — not GitHub Issues. Load
the `jira-conventions` skill before creating, editing or transitioning an
issue: it carries the required fields, this repository's title prefix and
component, and the worklog rule for Done and Cancelled.

Two of its rules apply whether or not the skill is loaded:

- Write every Jira issue key as a link to the issue, never bare:
  `[RSRMID-1234](https://centralnic.atlassian.net/browse/RSRMID-1234)`.
  Branch names and commit messages keep the bare key.
- Once an issue's last pull request has merged and its worklog is in, close it
  without asking if nothing is left — the skill lists what counts.

## Do NOT

- Add dependencies without explicit request
- Add `Co-Authored-By:` trailers to commit messages
- Rename or remove a workflow, input or secret without updating every caller —
  they reference it by path at `@main`
- Replace `RTLDEV_MW_CI_TOKEN` with `GITHUB_TOKEN` in the auto-merge workflow
