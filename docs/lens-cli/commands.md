# Command reference

Every command of `apicircle-lens` 2.1: what it runs, what it reads and writes,
what it prints, and how it exits. For the options of a command, run it with
`--help`.

[Back to the overview](README.md)

## At a glance

| Command | What it does | Changes files | Talks to |
| --- | --- | --- | --- |
| [`codegraph index`](#codegraph-index) | Builds the Code graph from your source. | The Code graph folder, `.gitignore` | Nothing |
| [`codegraph list-endpoints`](#codegraph-list-endpoints) | Lists the indexed endpoints. | No | Nothing |
| [`codegraph endpoint`](#codegraph-endpoint) | Shows the code behind one endpoint. | No | Nothing |
| [`codegraph explain`](#codegraph-explain) | Summarises the Code graph. | No | Nothing |
| [`codegraph status`](#codegraph-status) | Checks the committed Code graph is intact. | No | Nothing |
| [`codegraph support`](#codegraph-support) | Lists the languages and frameworks it parses. | No | Nothing |
| [`review`](#review) | Finds PR Review drift, gates CI and posts to a pull request. | The Code graph folder, when it indexes the checkout | Your Git host, when posting or reading a pull request |
| [`verify`](#verify) | Checks an OpenAPI or Swagger file is usable. | No | Nothing |
| [`scaffold`](#scaffold) | Generates code for endpoints the Spec declares and the code lacks. | Only with `--apply` | Nothing |
| [`git`](#git) | Connects a Git host and lists repositories and pull requests. | A credentials file in your home folder | Your Git host |
| [`mcp`](#mcp) | Runs the Lens MCP server for an AI client. | When a tool writes | Your Git host, when allowed |
| [`mock`, `mocks`](#mock-and-mocks) | Runs a mock server from a spec file. | Workspace files, for `mocks set-port` | Nothing |
| [`run`](#run) | Runs a saved execution plan and reports the result. | Run history, unless `--no-save` | The API under test |
| [`workspaces`, `folder`, `import`, `export`](#workspace-commands) | Manage API Circle workspaces. | Workspace files | Nothing |
| [`account`](#account) | Signs in and shows your entitlement. | A cached entitlement | API Circle |

Every command except `account` and `help` first checks `APICIRCLE_LENS_CLI_KEY`
with `https://api.apicircle.dev`. That check is the only thing sent to API
Circle: your source code is not.

## What every command prints when the key is wrong

No key set:

```text
The API Circle CLI is part of the Team plan.
Set APICIRCLE_LENS_CLI_KEY to a key from your account page (Account -> CLI keys).
```

A key that was revoked or mistyped:

```text
That APICIRCLE_LENS_CLI_KEY is not valid. Create a new one on your account page.
```

Both exit with code `1`, and nothing else runs. When the CLI cannot reach API
Circle at all, it runs on the licence it cached at the last successful check and
says so on stderr:

```text
Offline: using your cached licence (valid until 2026-10-29).
```

## codegraph index

```sh
apicircle-lens codegraph index
```

**Runs:** reads the source files your `.gitignore` keeps, finds every route and
traces the code each one runs. To leave more out, list patterns in
`.apicircle/.ignore` using `.gitignore` syntax.

**Writes:** `endpoints.json` and `index.json` in
`.apicircle/workspace-lens-desktop/codegraph/`, and one rule in `.gitignore` for
its local parser cache. Commit the folder: `review` compares it between
branches.

**Prints:**

```text
Indexed 6 endpoint(s) — detected: express.
  → .apicircle/workspace-lens-desktop/codegraph/endpoints.json
  → .apicircle/workspace-lens-desktop/codegraph/index.json
  → added the parser-cache rule to .gitignore
```

| Line | Meaning |
| --- | --- |
| `Indexed 6 endpoint(s)` | How many routes it found. |
| `detected: express` | The frameworks it recognised. `no known frameworks` means it recognised none and found routes by pattern matching. |
| `→ …/endpoints.json` | The Code graph: every endpoint and its code. This is the file `review --base` reads. |
| `→ …/index.json` | When, by which version and at which commit the Code graph was written. |
| `→ added the parser-cache rule` | Printed the first time only. |

**Options:** `--root <dir>` for a repository other than the current folder,
`--workspace-id <id>` for another folder name
(`.apicircle/workspace-<id>/codegraph/`), `--commit <sha>` to record a commit
other than `HEAD`.

**Exits:** `0` on success, `1` when it cannot index.

## codegraph list-endpoints

```sh
apicircle-lens codegraph list-endpoints
```

**Runs:** reads the committed Code graph. It does not index.

**Prints** one endpoint per line: method, path, framework and confidence.

```text
DELETE /api/v1/widgets/:widgetId  (express, high)
GET    /api/v1/health  (express, high)
GET    /api/v1/widgets  (express, high)
GET    /api/v1/widgets/:widgetId  (express, high)
PATCH  /api/v1/widgets/:widgetId  (express, high)
POST   /api/v1/widgets  (express, high)
```

Confidence says how sure the mapping is: `high`, `medium`, `low` or `unknown`.
`--json` prints the same list as JSON.

**Exits:** `0`, or `1` with `This repository has not been indexed yet. Run
`apicircle-lens codegraph index` first.`

## codegraph endpoint

```sh
apicircle-lens codegraph endpoint "DELETE /api/v1/widgets/:widgetId"
```

**Runs:** reads the committed Code graph and shows one endpoint in four parts.

```text
DELETE /api/v1/widgets/:widgetId  (express, confidence: high)
  pre-requisites/  — route registration, auth/tenant middleware, request validation + transformation
    src/middleware/auth.ts:17  requireApiKey  [high]
    src/routes/widgets.ts:20  DELETE /api/v1/widgets/:widgetId route  [high]
  handlers/  — controller, services, repository/DB access, domain logic, DTO/schema/model
    src/handlers/widgets.ts:78  deleteWidget  [high]
    src/shared/db.ts:39  removeWidget  [high]
  post-update/  — response mapping, audit logging, events, cache writes, notifications
    (none)
  shared/  — code reused by multiple endpoints (changing it impacts all of them)
    src/services/audit.ts:3  recordAudit  [high] · impacts 2 endpoints
```

| Part | What is in it |
| --- | --- |
| `pre-requisites/` | What runs before the handler: the route registration, guards and validators. |
| `handlers/` | The handler and the functions it calls. |
| `post-update/` | What runs after the response is shaped: response mapping, audit, events. |
| `shared/` | Code more than one endpoint reaches. `impacts 2 endpoints` is how many. |

Each line is a file, the line the code starts on, its name and the confidence of
that mapping. These four parts are what `review` compares between two branches.
`--json` prints the endpoint's full record.

**Exits:** `0`; `1` when no endpoint matches or the repository is not indexed;
`2` when the query is not `"<METHOD> <path>"`.

## codegraph explain

```sh
apicircle-lens codegraph explain
```

```text
Code graph — local, source-validated endpoint → code map.
  commit:      dcc90600871824336881f95c92f22e339b735acd
  generated:   2026-10-09T08:48:37.251Z
  languages:   typescript
  frameworks:  express
  endpoints:   6
  shared fns:  6 (reached by >1 endpoint)
  confidence:  high=6 medium=0 low=0 unknown=0

Mappings come only from AST/source analysis — they are authoritative, never model-guessed.
```

A quick read of what the Code graph holds: the commit it was built at, the
stacks it found, and how many endpoints sit at each confidence.

## codegraph status

```sh
apicircle-lens codegraph status --check
```

**Runs:** checks the committed Code graph: that both files can be read, that
they agree on how many endpoints there are, and that a newer version of the CLI
did not write them.

```text
Code graph — .apicircle/workspace-lens-desktop/codegraph
  endpoints:   6
  written by:  @apicircle-lens/cli 2.1.0 (generation 1)
  generated:   2026-10-09T08:48:37.251Z
  commit:      dcc90600871824336881f95c92f22e339b735acd
  integrity:   ok
```

Without `--check` it only reports. With `--check` it exits with `1` and lists
each problem when the Code graph is damaged, missing or from a newer version, so
a pipeline can stop on a Code graph a merge left broken. `codegraph index`
repairs what it reports. `--json` prints the report as JSON.

## codegraph support

```sh
apicircle-lens codegraph support
```

Lists every language and framework this version parses, the depth it reads each
at, and whether its Code graph can be edited. Stacks it does not know fall back
to pattern matching on route lines, at low confidence. `--all` adds the
languages it does not support yet, and `--json` prints the table as JSON.

## review

```sh
apicircle-lens review --base base.json --openapi openapi.yaml --fail-on breaking
```

**Runs:** compares the base Code graph with the branch under review and, with
`--openapi`, with your Spec. Without `--head`, it indexes the checkout first,
which rewrites the Code graph folder in the working tree.

**Prints** the review as text, Markdown or JSON, then the focus shift.

**Exits:** `0` passed, `1` the `--fail-on` level was met, `2` the review could
not run or post, `3` the Code graph at the pull request's head is not current
and `--require-fresh-index` was given.

This command has its own page: [The review command](review.md). The findings it
can raise are listed in [Findings](findings.md).

## verify

```sh
apicircle-lens verify --spec openapi.yaml
```

**Runs:** parses an OpenAPI 3.x or Swagger 2.0 file and checks it is complete
enough to generate from.

```text
✓ Contract is valid (3.0.3) — 8 operation(s), 0 warning(s).
```

Problems go to stderr, one per line: `✗ [code] location: message` for an error
and `! [code] location: message` for a warning, followed by
`Contract verification FAILED — 2 error(s), 1 warning(s).`

**Exits:** `0` when the file is valid, warnings included; `1` when it has an
error, or cannot be read or parsed.

## scaffold

```sh
apicircle-lens scaffold --spec openapi.yaml
```

**Runs:** finds the endpoints your Spec declares and your code does not
implement, and plans a handler and a route for each, following the conventions
it learns from the Code graph. It only previews until you pass `--apply`.

```text
2 endpoint(s) would be scaffolded (preview — re-run with --apply to write):
  POST /api/v1/widgets/{widgetId}/archive  [express · db: none]
    + src/handlers/archive.ts
    ~ src/routes/widgets.ts
  GET /api/v1/widgets/{widgetId}/history  [express · db: none]
    + src/handlers/history.ts
    ~ src/routes/widgets.ts
```

`+` is a file it would create and `~` one it would edit. The bracket names the
framework and database adapter it would generate for.

**Needs:** an account that allows code changes (Security → Code changes in your
account). The preview is refused too when that is off. In a Git clone, `--apply`
writes only on a branch of your own: it refuses the default branch and a
detached HEAD.

**Options worth knowing:** `--endpoint "GET /users/{id}"` for one operation,
`--tests` for a test stub beside each endpoint, `--spec-base-path` and
`--spec-strip-base` to line the Spec's paths up with your routes.

**Exits:** `0`, or `1` when it is refused or cannot plan.

## git

```sh
apicircle-lens git connect gitlab --token "<token>"
apicircle-lens git list
apicircle-lens git repos gitlab
apicircle-lens git prs gitlab acme api --state open
apicircle-lens git disconnect gitlab
```

**Runs:** `connect` verifies a token with the host and saves it, so `review`
can post without `--token`. It takes `github`, `gitlab`, `bitbucket` or
`azure-devops`. `list` shows the connected hosts, `repos` the repositories the
token reaches, `prs` a repository's open pull or merge requests, and
`disconnect` forgets the token.

**Writes:** `git-credentials.json` in `~/.apicircle` (or `APICIRCLE_LENS_HOME`).
In CI, pass the token in an environment variable instead: see
[CI recipes](ci/README.md).

For a self-managed host, add `--base-url`: `https://gitlab.example.com/api/v4`
for GitLab, `https://dev.azure.com/<organisation>` for Azure DevOps.

With nothing connected, `git list` prints:

```text
No Git hosts connected. Run `apicircle-lens git connect <provider> --token <t>`.
```

## mcp

```sh
apicircle-lens mcp --repo /absolute/path/to/your/repo
```

**Runs:** the Lens MCP server over stdio, so an AI client can read the Code
graph and run reviews. It needs a plan that includes the MCP server.

The tools that reach outside your machine (submitting a review, opening a pull
request, connecting a host, running a plan) are off until you start it with
`--allow-egress`. The tools that change code are refused unless your account
allows code changes.

## mock and mocks

```sh
apicircle-lens mock openapi.yaml --port 4010
```

**Runs:** a mock server from an OpenAPI, Postman or Insomnia file, on
`127.0.0.1` and a free port unless you name one. It runs until you press Ctrl-C.

```text
Mock server listening on http://127.0.0.1:4010 with 8 endpoints (type=openapi). Press Ctrl-C to stop.
```

`mocks list` shows the mock servers saved in a workspace with their default
ports, and `mocks set-port <name> <port>` sets or clears one.

## run

```sh
apicircle-lens run "Smoke tests" --reporter junit > results.xml
```

**Runs:** a saved execution plan from a workspace: its requests in order, with
their assertions.

**Prints** a line per step and a summary. `--reporter json` and
`--reporter junit` print a machine-readable report instead.

**Options worth knowing:** `--env <name>` to layer an environment, `--secrets
<file>` for secret values, `--bail` to stop at the first failed step,
`--no-save` to leave run history alone.

**Exits:** `0` when every step passed, `1` when a step failed or the run was
interrupted, `3` when the run was not permitted.

## Workspace commands

`workspaces`, `folder`, `import` and `export` manage API Circle workspaces from
the terminal.

| Command | What it does |
| --- | --- |
| `workspaces list` | Lists the workspaces registered on this machine. `●` marks the active one. |
| `workspaces create <name>` | Creates a workspace. `--sample` adds one sample request. |
| `workspaces use <name>` | Sets the active workspace. |
| `workspaces path [name]` | Prints where a workspace is on disk. |
| `folder list` | Prints the folder tree, with a marker on folders that carry auth. |
| `folder create`, `rename`, `move`, `delete` | Edit the tree. `create` prints the new folder's id. |
| `folder set-auth`, `clear-auth` | Set or clear folder-level auth that requests inherit. |
| `import <type> <file>` | Imports `openapi`, `postman`, `insomnia`, `curl` or an `apicircle` folder export into a workspace. |
| `export folder <name>` | Writes a folder and everything under it as JSON. Credentials are left out unless you name them with `--include-credential`. |

Each takes `--workspace-name <name>` or `--workspace-path <dir>` to pick a
workspace other than the active one. On an error, such as a workspace that
cannot be found, they print `error: …` and exit with a non-zero code.

## account

```sh
apicircle-lens account whoami
```

`login` signs in with a device code, `refresh` renews the cached entitlement,
`logout` clears it, `activate` takes a licence token or file, and `whoami`
shows what is cached without contacting anything:

```text
Not signed in to API Circle Lens.
```

These work without a CLI key. A pipeline does not need them: it authenticates
with `APICIRCLE_LENS_CLI_KEY`.
