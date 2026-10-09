# Bitbucket Pipelines

Review every pull request for drift, comment on it, and check the main branch
again after each merge. This recipe is for Bitbucket Cloud.

[All CI recipes](README.md) · [What the four steps do](README.md#the-four-steps)

## Variables

Add both under **Repository settings → Repository variables**, secured.

| Name | Value |
| --- | --- |
| `APICIRCLE_LENS_CLI_KEY` | Your CLI key. |
| `APICIRCLE_LENS_GIT_TOKEN` | A repository access token with **Pull requests: Write**. An Atlassian API token works too, written as `<account email>:<token>`. |

## `bitbucket-pipelines.yml`

```yaml
image: node:22

clone:
  depth: full

definitions:
  steps:
    - step: &pull-request-drift
        name: PR Review drift
        script:
          - export CODE_GRAPH=.apicircle/workspace-lens-desktop/codegraph/endpoints.json
          - git fetch origin "$BITBUCKET_PR_DESTINATION_BRANCH"
          - git show "FETCH_HEAD:$CODE_GRAPH" > /tmp/base.json
          - >-
            npx --yes @apicircle-lens/cli review
            --base /tmp/base.json
            --openapi openapi.yaml
            --fail-on breaking
            --comment --host bitbucket
            --repo "$BITBUCKET_REPO_FULL_NAME" --pr "$BITBUCKET_PR_ID"
    - step: &after-merge-drift
        name: Drift after merge
        script:
          - export CODE_GRAPH=.apicircle/workspace-lens-desktop/codegraph/endpoints.json
          - git show "HEAD^1:$CODE_GRAPH" > /tmp/base.json
          - >-
            npx --yes @apicircle-lens/cli review
            --base /tmp/base.json
            --openapi openapi.yaml
            --fail-on breaking

pipelines:
  pull-requests:
    '**':
      - step: *pull-request-drift
  branches:
    main:
      - step: *after-merge-drift
```

## What each part does

| Part | Why |
| --- | --- |
| `clone: depth: full` | Bitbucket clones 50 commits by default. A full clone always has the commit the job compares with. |
| `pull-requests: '**'` | Runs for a pull request from any branch. Bitbucket merges the destination into the checkout before the script starts. |
| `git fetch origin "$BITBUCKET_PR_DESTINATION_BRANCH"` | Brings in the destination branch, which the clone does not track. Because the checkout is already merged with it, its tip is the right base. |
| `--host bitbucket` | Posts through Bitbucket Cloud's pull request API. |
| `--repo "$BITBUCKET_REPO_FULL_NAME"` | `workspace/repository`. |
| `--pr "$BITBUCKET_PR_ID"` | The pull request's number. |
| `branches: main` | Runs after a merge, comparing the commit before it with the one after. |

## Variations

**Submit a review with line comments.** Replace `--comment` with `--submit`,
and add `--event APPROVE` or `--event REQUEST_CHANGES` to give a verdict.

**Routes under a prefix.** When the code mounts routes under `/api` and the
Spec's paths omit it, add `--spec-base-path /api`.

**Another default branch.** Replace `main` under `branches`.
