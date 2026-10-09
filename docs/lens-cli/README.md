# API Circle Lens CLI

`apicircle-lens` is the command line for [API Circle Lens](https://apicircle.dev/lens).
It builds a **Code graph** of your API from its source, mapping every endpoint to
the code that implements it. It then uses the Code graph and your **Spec** (the
OpenAPI contract) to find **PR Review drift**: where a pull request moves the API
away from its contract.

These pages document `@apicircle-lens/cli` 2.1.

## How it works

1. **Index.** `apicircle-lens codegraph index` reads your repository on your
   machine and writes the Code graph to
   `.apicircle/workspace-lens-desktop/codegraph/`. You commit that folder.
2. **Compare.** `apicircle-lens review` takes the Code graph committed on the
   base branch and indexes the branch you are on. It reports, per endpoint, what
   the change did: a route removed, access control dropped, a handler edited.
3. **Check the Spec.** With `--openapi`, it also compares the code with your
   Spec and reports where the pull request introduced a disagreement: an
   undocumented route, a missing parameter, a different status code.
4. **Gate and post.** `--fail-on` turns the result into an exit code your
   pipeline can stop on. `--comment` posts the review on the pull request.

Your source code is analysed on the machine that runs the CLI and is never sent
to API Circle. The CLI contacts `https://api.apicircle.dev` to check your key,
and your Git host when you ask it to read or post on a pull request.

## Before you start

- Node.js 20 or later.
- An API Circle account on a plan that includes the CLI.
  [Compare plans](https://apicircle.dev/pricing).
- A CLI key. Sign in at [account.apicircle.dev](https://account.apicircle.dev),
  open **CLI keys**, create a key and put it in `APICIRCLE_LENS_CLI_KEY`. In CI,
  store it as a secret.

## Start in five minutes

```sh
npm install -g @apicircle-lens/cli
export APICIRCLE_LENS_CLI_KEY="<your key>"
```

Build the Code graph on your default branch and commit it:

```sh
apicircle-lens codegraph index
git add .apicircle .gitignore
git commit -m "Add the Code graph"
git push
```

On a branch with a change, review it against the default branch and the Spec:

```sh
git show origin/main:.apicircle/workspace-lens-desktop/codegraph/endpoints.json > ../base.json
apicircle-lens review --base ../base.json --openapi openapi.yaml --fail-on breaking
```

The command prints the review and exits with `1` when it finds a breaking
change. The [walkthrough](walkthrough.md) shows a full run and how to read it.

## The pages

| Page | What it covers |
| --- | --- |
| [Walkthrough](walkthrough.md) | One pull request on a sample API, from the first index to a red pipeline, with the real output of every step. |
| [The review command](review.md) | What `review` compares, each line of its three output formats, the `--fail-on` gate, exit codes and posting to a pull request. |
| [Findings](findings.md) | Every finding the review can raise, grouped by severity: Critical, Warning and Info. |
| [Command reference](commands.md) | Every command: what it runs, what it reads and writes, what it prints and how it exits. |
| [CI recipes](ci/README.md) | The steps every pipeline needs, and a complete file for each service below. |

## CI recipes

| Service | Pipeline file | Recipe |
| --- | --- | --- |
| GitHub Actions | `.github/workflows/api-drift.yml` | [ci/github-actions.md](ci/github-actions.md) |
| GitLab CI/CD | `.gitlab-ci.yml` | [ci/gitlab-ci.md](ci/gitlab-ci.md) |
| Bitbucket Pipelines | `bitbucket-pipelines.yml` | [ci/bitbucket-pipelines.md](ci/bitbucket-pipelines.md) |
| Azure Pipelines | `azure-pipelines.yml` | [ci/azure-pipelines.md](ci/azure-pipelines.md) |
| CircleCI | `.circleci/config.yml` | [ci/circleci.md](ci/circleci.md) |

Lens speaks GitLab, Bitbucket Cloud and Azure DevOps through each host's own
review API, and GitHub through GitHub's.

## Versions

A repository that version 2.0 indexed keeps its Code graph in
`.apicircle/workspace-default/codegraph/`, and 2.1 keeps using that folder when
it finds it. Replace `workspace-lens-desktop` with `workspace-default` in the
`git show` paths on these pages if that is where yours lives.

Run `apicircle-lens --version` to see what you have, and
`apicircle-lens <command> --help` for the options of any command.
