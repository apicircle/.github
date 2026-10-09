# CI recipes

Run `apicircle-lens review` on every pull request, fail the build on drift, and
post the result where the reviewer is looking.

[Back to the overview](../README.md)

| Service | Recipe |
| --- | --- |
| GitHub Actions | [github-actions.md](github-actions.md) |
| GitLab CI/CD | [gitlab-ci.md](gitlab-ci.md) |
| Bitbucket Pipelines | [bitbucket-pipelines.md](bitbucket-pipelines.md) |
| Azure Pipelines | [azure-pipelines.md](azure-pipelines.md) |
| CircleCI | [circleci.md](circleci.md) |

Every recipe is the same four steps. This page explains them once.

## Before the first run

**Commit the Code graph on your default branch.** The review compares the
branch under review with the Code graph of the branch it merges into, so that
one has to exist:

```sh
apicircle-lens codegraph index
git add .apicircle .gitignore
git commit -m "Add the Code graph"
git push
```

Pull request branches do not need to re-index. The pipeline indexes the
checkout itself.

**Create a CLI key** at [account.apicircle.dev](https://account.apicircle.dev),
under **CLI keys**, and store it in your CI service as the secret
`APICIRCLE_LENS_CLI_KEY`.

**Create a token for your Git host** that may comment on pull requests, and
store it as a secret too. Each recipe names the permission it needs.

## The four steps

### 1. Check out with history

The job needs the commit it compares with, so a shallow clone of one commit is
not enough. Each recipe sets the depth its service needs.

### 2. Take the base Code graph

```sh
git show "<base commit>:.apicircle/workspace-lens-desktop/codegraph/endpoints.json" > base.json
```

Which commit is the base depends on what your CI service checks out:

| The job checks out | Services | Base to use |
| --- | --- | --- |
| The pull request merged into its target | GitHub Actions, Bitbucket Pipelines, Azure Pipelines, GitLab merged results pipelines | The tip of the target branch. |
| The pull request's own branch | GitLab merge request pipelines, CircleCI | The merge base: the commit the branch started from. |

The second row matters. Compared with the tip of a target branch that has moved
on, a branch that is merely behind looks as if it removed whatever was merged
since.

Write `base.json` to a temporary folder outside the checkout, so it is never
committed by mistake.

### 3. Review

```sh
npx --yes @apicircle-lens/cli review \
  --base base.json \
  --openapi openapi.yaml \
  --fail-on breaking \
  --comment --repo "<repository>" --pr "<number>"
```

| Option | Why it is there |
| --- | --- |
| `--base base.json` | The Code graph from step 2. The head is the checkout, which the command indexes. |
| `--openapi openapi.yaml` | Your Spec. Without it, only code changes are reported and nothing says the Spec was skipped. |
| `--fail-on breaking` | Exit with `1` on a Critical finding. Use `warning`, `info` or `diffracting` for another gate: see [Findings](../findings.md#choosing-a-level-for-your-pipeline). |
| `--comment --repo --pr` | Post the review as one comment on the pull request, updated in place on later runs. |
| `--host`, `--base-url` | Needed for any host but github.com. Each recipe sets them. |
| `--spec-base-path /api` | Add it when your code mounts routes under a prefix the Spec's paths omit. |

To run a fixed version, name it: `npx --yes @apicircle-lens/cli@2.1.0 review …`.

### 4. Read the exit code

| Code | Meaning | The build should |
| --- | --- | --- |
| `0` | No finding at the `--fail-on` level. | Pass. |
| `1` | Drift at the `--fail-on` level, or the comment could not be posted, or the key was not accepted. | Fail. The review is in the job log and on the pull request. |
| `2` | The review could not run: no base Code graph, an unreadable Spec, a missing token. | Fail. Fix the pipeline, not the pull request. |

## After the merge

A second job on the default branch catches what two pull requests did together.
It compares the commit before the merge with the commit after:

```sh
git show "HEAD^1:.apicircle/workspace-lens-desktop/codegraph/endpoints.json" > base.json
npx --yes @apicircle-lens/cli review --base base.json --openapi openapi.yaml --fail-on breaking
```

`HEAD^1` is the right base for a merge commit and for a squash merge. A rebase
merge lands several commits at once, and `HEAD^1` then covers only the last:
use the commit your CI service reports as the previous one on the branch.

There is no pull request to comment on here, so the review goes to the job log.

## Things that apply everywhere

**Forks.** CI services keep secrets from pull requests that come from a fork.
Without `APICIRCLE_LENS_CLI_KEY` the command stops with exit code `1` before it
reviews anything. Skip the job for forks, or run it after a maintainer approves.

**Public repositories.** The CLI refuses to post a review that names a security
weakness on a public repository, and exits with `2`. Add
`--omit-security-findings` to post the rest, or
`--allow-public-security-findings` to post all of it. See
[Public repositories](../review.md#public-repositories).

**A damaged Code graph.** A merge can leave the committed Code graph
inconsistent. Add this step to stop on it early:

```sh
npx --yes @apicircle-lens/cli codegraph status --check
```

**A failing network.** When the CLI cannot reach API Circle, it runs on the
licence cached at the last successful check. A runner that starts clean has
none: set `APICIRCLE_LENS_CACHE_DIR` to a folder your CI service caches between
jobs to keep working through an outage.

**Version 2.0.** A repository that 2.0 indexed keeps its Code graph in
`.apicircle/workspace-default/codegraph/`. Use that path in step 2.
