# CircleCI

Review every pull request for drift, comment on it, and check the main branch
again after each merge. The example posts to GitHub.

[All CI recipes](README.md) · [What the four steps do](README.md#the-four-steps)

## Context

Create a context named `apicircle-lens` under **Organization Settings →
Contexts** with two variables.

| Name | Value |
| --- | --- |
| `APICIRCLE_LENS_CLI_KEY` | Your CLI key. |
| `GITHUB_TOKEN` | A GitHub token that may comment on pull requests in this repository. CircleCI does not provide one. |

## `.circleci/config.yml`

```yaml
version: 2.1

jobs:
  drift-pull-request:
    docker:
      - image: cimg/node:22.11
    environment:
      CODE_GRAPH: .apicircle/workspace-lens-desktop/codegraph/endpoints.json
    steps:
      - checkout
      - run:
          name: Code graph of the base
          command: |
            git fetch --no-tags origin "+refs/heads/main:refs/remotes/origin/main"
            BASE_SHA="$(git merge-base origin/main HEAD)"
            git show "$BASE_SHA:$CODE_GRAPH" > /tmp/base.json
      - run:
          name: Review the pull request
          command: |
            PR_NUMBER="${CIRCLE_PULL_REQUEST##*/}"
            POST=""
            if [ -n "$PR_NUMBER" ]; then
              POST="--comment --repo $CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME --pr $PR_NUMBER"
            fi
            npx --yes @apicircle-lens/cli review \
              --base /tmp/base.json \
              --openapi openapi.yaml \
              --fail-on breaking \
              $POST

  drift-after-merge:
    docker:
      - image: cimg/node:22.11
    environment:
      CODE_GRAPH: .apicircle/workspace-lens-desktop/codegraph/endpoints.json
    steps:
      - checkout
      - run:
          name: Code graph before the merge
          command: git show "HEAD^1:$CODE_GRAPH" > /tmp/base.json
      - run:
          name: Review the merge
          command: |
            npx --yes @apicircle-lens/cli review \
              --base /tmp/base.json \
              --openapi openapi.yaml \
              --fail-on breaking

workflows:
  api-drift:
    jobs:
      - drift-pull-request:
          context: apicircle-lens
          filters:
            branches:
              ignore: main
      - drift-after-merge:
          context: apicircle-lens
          filters:
            branches:
              only: main
```

## What each part does

| Part | Why |
| --- | --- |
| `git merge-base origin/main HEAD` | CircleCI checks out the branch as it is, not merged with `main`. The base is therefore the commit the branch started from. Against the tip of a `main` that has moved on, a branch that is merely behind would look as if it removed what was merged since. |
| `CIRCLE_PULL_REQUEST` | The pull request's address. The script takes the number from its end. |
| `POST` | Empty when the branch has no open pull request, so the job still reviews and gates without posting. |
| `filters` | The first job runs on every branch but `main`, the second on `main` only. |
| `context` | Gives both jobs the two secrets. |

## Things to know

**Turn on "Only build pull requests"** in **Project Settings → Advanced**.
`CIRCLE_PULL_REQUEST` is empty for a push made before the pull request was
opened, and the review is then not posted.

**The target branch is written into the file.** CircleCI does not expose the
branch a pull request merges into. Replace `main` in both jobs and both filters
if yours differs.

**Pull requests from forks** do not receive the context's secrets. The job
stops with exit code `1` at the key check.

## Variations

**Code on Bitbucket or GitLab.** Add `--host bitbucket` or `--host gitlab` to
`POST`, and put the token in `APICIRCLE_LENS_GIT_TOKEN` instead of
`GITHUB_TOKEN`. The number at the end of `CIRCLE_PULL_REQUEST` is the right one
on all three.

**Routes under a prefix.** When the code mounts routes under `/api` and the
Spec's paths omit it, add `--spec-base-path /api`.
