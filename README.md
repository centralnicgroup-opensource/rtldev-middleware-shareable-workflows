# rtldev-middleware-shareable-workflows

## Scheduled Node Dependency Refresh

GitHub reusable workflows can contain the dependency refresh implementation, but the `schedule` trigger must live in each repository that should run on a cron. The consuming repository keeps a small wrapper workflow and calls this shared workflow from the scheduled job.

Example replacement for `centralnicgroup-opensource/rtldev-middleware-php-sdk`:

```yaml
name: Daily Node dependency refresh

"on":
  schedule:
    - cron: "17 3 * * *"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: daily-node-dependency-refresh
  cancel-in-progress: false

jobs:
  refresh-dependencies:
    uses: centralnicgroup-opensource/rtldev-middleware-shareable-workflows/.github/workflows/daily-node-dependency-refresh.yml@main
    secrets: inherit
    with:
      base-ref: master
```

`base-ref` can be omitted for repositories where the default branch is the branch that should receive the dependency refresh pull request. Other optional inputs are available for the Node.js version, pnpm version, pull request branch, commit message, pull request title, pull request body, auto-merge behavior, and merge method.
