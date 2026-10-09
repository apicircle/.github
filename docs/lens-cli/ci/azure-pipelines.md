# Azure Pipelines

Review every pull request in Azure Repos for drift, comment on it, and check
the main branch again after each merge.

[All CI recipes](README.md) · [What the four steps do](README.md#the-four-steps)

## Variables

Add both to the pipeline under **Edit → Variables**, with **Keep this value
secret** ticked.

| Name | Value |
| --- | --- |
| `APICIRCLE_LENS_CLI_KEY` | Your CLI key. |
| `APICIRCLE_LENS_GIT_TOKEN` | A personal access token with **Code: Read** and **Pull Request Threads: Read & write**. |

## `azure-pipelines.yml`

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  codeGraph: .apicircle/workspace-lens-desktop/codegraph/endpoints.json

steps:
  - checkout: self
    fetchDepth: 0

  - task: NodeTool@0
    inputs:
      versionSpec: '22.x'

  - bash: |
      set -euo pipefail
      TARGET="${SYSTEM_PULLREQUEST_TARGETBRANCH#refs/heads/}"
      git show "origin/$TARGET:$(codeGraph)" > "$(Agent.TempDirectory)/base.json"
      npx --yes @apicircle-lens/cli review \
        --base "$(Agent.TempDirectory)/base.json" \
        --openapi openapi.yaml \
        --fail-on breaking \
        --comment --host azure-devops \
        --base-url "$(System.CollectionUri)" \
        --repo "$(System.TeamProject)/$(Build.Repository.Name)" \
        --pr "$(System.PullRequest.PullRequestId)"
    displayName: PR Review drift
    condition: and(succeeded(), eq(variables['Build.Reason'], 'PullRequest'))
    env:
      APICIRCLE_LENS_CLI_KEY: $(APICIRCLE_LENS_CLI_KEY)
      APICIRCLE_LENS_GIT_TOKEN: $(APICIRCLE_LENS_GIT_TOKEN)

  - bash: |
      set -euo pipefail
      git show "HEAD^1:$(codeGraph)" > "$(Agent.TempDirectory)/base.json"
      npx --yes @apicircle-lens/cli review \
        --base "$(Agent.TempDirectory)/base.json" \
        --openapi openapi.yaml \
        --fail-on breaking
    displayName: Drift after merge
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    env:
      APICIRCLE_LENS_CLI_KEY: $(APICIRCLE_LENS_CLI_KEY)
```

## Run it on pull requests

Azure Repos does not start a pipeline for a pull request from the YAML file. Add
it as a branch policy: **Project settings → Repositories → your repository →
Policies → Branch policies → main → Build validation**, and choose this
pipeline. Make the policy **Required** to block a merge while the review fails.

## What each part does

| Part | Why |
| --- | --- |
| `trigger: main` | Runs the second step after each merge. |
| `fetchDepth: 0` | New pipelines clone one commit by default. The job needs the target branch and the previous commit. |
| `TARGET` | `System.PullRequest.TargetBranch` is `refs/heads/main`. The script strips the prefix. |
| `origin/$TARGET` | A pull request build checks out the pull request merged into its target, so the target's tip is the right base. |
| `--host azure-devops` | Posts through Azure DevOps' pull request threads. |
| `--base-url "$(System.CollectionUri)"` | Your organisation's address, `https://dev.azure.com/<organisation>/`. Azure DevOps always needs it. |
| `--repo "$(System.TeamProject)/$(Build.Repository.Name)"` | `project/repository`. |
| `--pr "$(System.PullRequest.PullRequestId)"` | The pull request's number. |
| `env:` | Secret variables are not passed to a script unless it maps them. |
| `condition:` | The first step runs for pull request builds only, the second for everything else. |

## Variations

**A project or repository name with a space in it.** `--repo` does not accept
one. Pass the ids instead, which Azure DevOps accepts wherever it accepts a
name: `--repo "$(System.TeamProjectId)/$(Build.Repository.ID)"`.

**Submit a review with line comments.** Replace `--comment` with `--submit`,
and add `--event APPROVE` or `--event REQUEST_CHANGES` to give a verdict.

**Routes under a prefix.** When the code mounts routes under `/api` and the
Spec's paths omit it, add `--spec-base-path /api`.

**Code on GitHub, pipeline on Azure.** Use `--host github`, drop `--base-url`,
pass `--repo "$(Build.Repository.Name)"`, which is `owner/name` there, and put
a GitHub token in `GITHUB_TOKEN`.
