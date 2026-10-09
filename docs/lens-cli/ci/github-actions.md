# GitHub Actions

Review every pull request for drift, comment on it, and check the default
branch again after each merge.

[All CI recipes](README.md) · [What the four steps do](README.md#the-four-steps)

## Secrets

| Name | Where | Value |
| --- | --- | --- |
| `APICIRCLE_LENS_CLI_KEY` | Repository **Settings → Secrets and variables → Actions** | Your CLI key. |
| `GITHUB_TOKEN` | Provided by GitHub Actions | Nothing to create. The workflow below grants it `pull-requests: write`. |

## `.github/workflows/api-drift.yml`

```yaml
name: PR Review drift

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  pull-request:
    if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Code graph of the base branch
        run: git show "origin/$GITHUB_BASE_REF:.apicircle/workspace-lens-desktop/codegraph/endpoints.json" > "$RUNNER_TEMP/base.json"
      - name: Review
        run: >-
          npx --yes @apicircle-lens/cli review
          --base "$RUNNER_TEMP/base.json"
          --openapi openapi.yaml
          --fail-on breaking
          --comment --repo "$GITHUB_REPOSITORY" --pr "${{ github.event.pull_request.number }}"
        env:
          APICIRCLE_LENS_CLI_KEY: ${{ secrets.APICIRCLE_LENS_CLI_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  after-merge:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Code graph before the merge
        run: git show "HEAD^1:.apicircle/workspace-lens-desktop/codegraph/endpoints.json" > "$RUNNER_TEMP/base.json"
      - name: Review
        run: >-
          npx --yes @apicircle-lens/cli review
          --base "$RUNNER_TEMP/base.json"
          --openapi openapi.yaml
          --fail-on breaking
        env:
          APICIRCLE_LENS_CLI_KEY: ${{ secrets.APICIRCLE_LENS_CLI_KEY }}
```

## What each part does

| Part | Why |
| --- | --- |
| `fetch-depth: 0` | The default checkout has one commit. The job needs the base branch too. |
| `origin/$GITHUB_BASE_REF` | The branch the pull request merges into. On `pull_request`, the checkout is the pull request already merged into it, so its tip is the right base. |
| `permissions` | `pull-requests: write` lets the provided token post the comment. |
| The `if` on `pull-request` | Skips pull requests from forks, which do not receive secrets. |
| `after-merge` | Compares the commit before the merge with the one after it. `fetch-depth: 2` is enough for `HEAD^1`. |

## Variations

**Comment on the changed lines.** Replace `--comment` with `--submit` to post a
review with a comment on each drifted line. Add `--event REQUEST_CHANGES` to
block the pull request with it.

**GitHub Enterprise Server.** Add `--base-url https://ghe.example.com/api/v3`.

**Routes under a prefix.** When the code mounts routes under `/api` and the
Spec's paths omit it, add `--spec-base-path /api`.

**Fail on Warnings too.** Use `--fail-on warning`.
