# GitLab CI/CD

Review every merge request for drift, comment on it, and check the default
branch again after each merge.

[All CI recipes](README.md) · [What the four steps do](README.md#the-four-steps)

## Variables

Add both under **Settings → CI/CD → Variables**, masked.

| Name | Value |
| --- | --- |
| `APICIRCLE_LENS_CLI_KEY` | Your CLI key. |
| `APICIRCLE_LENS_GIT_TOKEN` | A project access token with the `api` scope and a role that may comment on merge requests. The job's own `CI_JOB_TOKEN` cannot post the comment. |

A protected variable is only given to pipelines on protected branches. Leave
these two unprotected, or merge request pipelines will not receive them.

## `.gitlab-ci.yml`

```yaml
stages:
  - review

.api-drift:
  stage: review
  image: node:22
  variables:
    GIT_DEPTH: "0"
    CODE_GRAPH: .apicircle/workspace-lens-desktop/codegraph/endpoints.json

api-drift:merge-request:
  extends: .api-drift
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script:
    - BASE_SHA="${CI_MERGE_REQUEST_TARGET_BRANCH_SHA:-$CI_MERGE_REQUEST_DIFF_BASE_SHA}"
    - git show "$BASE_SHA:$CODE_GRAPH" > /tmp/base.json
    - >-
      npx --yes @apicircle-lens/cli review
      --base /tmp/base.json
      --openapi openapi.yaml
      --fail-on breaking
      --comment --host gitlab --base-url "$CI_API_V4_URL"
      --repo "$CI_PROJECT_PATH" --pr "$CI_MERGE_REQUEST_IID"

api-drift:after-merge:
  extends: .api-drift
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - git show "HEAD^1:$CODE_GRAPH" > /tmp/base.json
    - >-
      npx --yes @apicircle-lens/cli review
      --base /tmp/base.json
      --openapi openapi.yaml
      --fail-on breaking
```

## What each part does

| Part | Why |
| --- | --- |
| `GIT_DEPTH: "0"` | GitLab clones shallowly by default. The job needs the commit it compares with. |
| `rules` on `merge_request_event` | Runs the job in merge request pipelines, where the merge request variables exist. |
| `BASE_SHA` | A merge request pipeline checks out the source branch, so the base is the commit it started from: `CI_MERGE_REQUEST_DIFF_BASE_SHA`. A merged results pipeline checks out the merge, so the base is the target's tip: `CI_MERGE_REQUEST_TARGET_BRANCH_SHA`, which is set only there. |
| `--host gitlab` | Posts through GitLab's merge request API. |
| `--base-url "$CI_API_V4_URL"` | Points at this GitLab instance, so the same file works on gitlab.com and self-managed. |
| `--repo "$CI_PROJECT_PATH"` | The project path, subgroups included: `group/subgroup/project`. |
| `--pr "$CI_MERGE_REQUEST_IID"` | The merge request's number within the project. |

## Variations

**Submit a review with line comments.** Replace `--comment` with `--submit`.
GitLab has no request-changes verdict: with `--event REQUEST_CHANGES` the review
is posted as a comment, and the command says so.

**Routes under a prefix.** When the code mounts routes under `/api` and the
Spec's paths omit it, add `--spec-base-path /api`.

**Merge requests from forks.** Their pipelines run in the fork and do not
receive the parent project's variables. The job stops with exit code `1` at the
key check.
