<p align="center">
  <img src="https://raw.githubusercontent.com/apicircle/.github/main/assets/logo.svg" alt="API Circle" width="120" height="120" />
</p>

<h1 align="center">API Circle</h1>

<p align="center">
  <strong>Know when a pull request changes your API.</strong><br />
  Lens reads your repository and maps every endpoint to the code behind it,<br />
  then shows what a pull request changes against your OpenAPI spec.<br />
  The API workspace is free.
</p>

<p align="center">
  <a href="https://apicircle.dev"><img src="https://img.shields.io/badge/website-apicircle.dev-8B5CF6?style=flat-square" alt="apicircle.dev" /></a>
  <a href="https://www.npmjs.com/package/@apicircle-lens/cli"><img src="https://img.shields.io/npm/v/@apicircle-lens/cli?style=flat-square&color=8B5CF6&label=%40apicircle-lens%2Fcli" alt="@apicircle-lens/cli on npm" /></a>
  <a href="https://github.com/apicircle/studio"><img src="https://img.shields.io/github/stars/apicircle/studio?style=flat-square&color=8B5CF6&label=Studio%20stars" alt="Studio stars" /></a>
  <a href="https://github.com/apicircle/studio/blob/main/LICENSE"><img src="https://img.shields.io/badge/Studio%20licence-source--available-8B5CF6?style=flat-square" alt="Studio licence: source-available" /></a>
</p>

---

API Circle is one app with two halves. The **API workspace**, API Circle Studio,
is free and needs no account. **API Circle Lens** is the paid half, and it reads
the code behind your API. The two are licensed and priced differently, so this
page keeps them apart.
[apicircle.dev/pricing](https://apicircle.dev/pricing) has the plans and what
each one includes.

## API Circle Lens

Lens builds a **Code graph** of your API from its source, every endpoint mapped
to the code that implements it, and, using the Code graph and the **Spec** (the
OpenAPI contract), identifies **PR Review drift**: where a pull request moves
the API away from its contract.

Lens is the paid half. What it does:

- **Code graph.** It is built from your source, never from a spec. For each
  endpoint Lens records the route, the handler and the functions the handler
  calls, and writes the result into your repository as a `codegraph/` sidecar,
  so it's versioned and reviewed like any other file. Where it can't read an
  endpoint with confidence, it reports `unknown`.
- **PR Review drift.** Lens makes two comparisons: the pull request against the
  Code graph of its base, and against your Spec. The findings are concrete: a
  route removed, access control dropped, a status the Spec doesn't document. In
  the desktop app, nothing reaches your Git host until you press Submit.
- **Languages.** TypeScript and JavaScript, Python, Go, Java, C# and Rust.
  [The Code graph page](https://apicircle.dev/features/code-graph) lists the
  frameworks for each. PHP is read only by pattern matching on route lines, and
  Ruby, Elixir and Kotlin aren't supported yet.
- **Git hosts.** GitHub works on every plan, including free. Lens speaks GitLab,
  Bitbucket Cloud and Azure DevOps through each host's own review API, and
  those three are a paid capability.
- **Your code stays on your machine.** Analysis runs locally and sends API
  Circle no repository content. The one exception is "Index with AI", an
  optional pass that asks first and sends the files it reads to the AI provider
  you chose, with your own key.

### Where Lens runs

| Surface | Where it runs | What it needs |
| --- | --- | --- |
| **Lens desktop** | Windows 10 or later (x64), macOS (Apple silicon and Intel), Linux (x64) | A paid plan for the Lens panels: you sign in, and the ones your plan includes appear |
| **MCP server** | `apicircle-lens mcp`, over stdio, for an MCP client | A CLI key, on a plan that includes the MCP server |
| **Command line** | Node.js 20 or later, on any CI runner | A CLI key, on a plan that includes the CLI |

Lens desktop holds the whole workspace plus the Code graph, Review, Assistant
and MCP panels, and the workspace inside it stays free.
[Download it from the latest release](https://github.com/apicircle/.github/releases/latest).

> **The Lens installers aren't code-signed yet**, so your system asks before the
> first run. On Windows, SmartScreen warns: choose **More info → Run anyway**.
> On macOS, open **System Settings → Privacy & Security** and choose **Open
> Anyway**. On Linux, install the `.deb`, or make the AppImage executable and
> run it. The release page has the full steps. Signed builds are in progress.

### The Lens command line

[`@apicircle-lens/cli`](https://www.npmjs.com/package/@apicircle-lens/cli)
installs the `apicircle-lens` binary. It runs the same drift check in CI and
exits with a code your pipeline can stop on. The MCP server ships inside it.

```bash
npm install -g @apicircle-lens/cli
export APICIRCLE_LENS_CLI_KEY="<your key>"

apicircle-lens codegraph index                          # build the Code graph, then commit the folder it writes
apicircle-lens review --base base.json --openapi openapi.yaml --fail-on breaking
apicircle-lens mcp --repo /absolute/path/to/your/repo   # the Lens MCP server, over stdio
```

`base.json` is the Code graph committed on your base branch. The documentation
shows the `git show` line that produces it.

Before you start, you need:

- Node.js 20 or later.
- An API Circle account on a plan that includes the CLI. `apicircle-lens mcp`
  needs a plan that includes the MCP server.
  [Compare plans](https://apicircle.dev/pricing).
- A CLI key. Sign in at [account.apicircle.dev](https://account.apicircle.dev),
  open **CLI keys**, create a key and put it in `APICIRCLE_LENS_CLI_KEY`. In CI,
  store it as a secret.

The [Lens CLI documentation](https://github.com/apicircle/.github/blob/main/docs/lens-cli/README.md)
has a walkthrough of one pull request, every finding the review can raise, a
reference for each command, and a complete pipeline file for GitHub Actions,
GitLab CI/CD, Bitbucket Pipelines, Azure Pipelines and CircleCI.

Two more things live in this binary and therefore need a CLI key.
`apicircle-lens run` runs a saved execution plan headlessly, and
`apicircle-lens mock` serves a mock from a spec file. Inside the free apps, both
are free.

The MCP server gives an AI client your Code graph and PR Review. The desktop
app's MCP panel installs it into Claude Desktop, Claude Code, Codex, Cursor,
Continue, Zed and Windsurf, and any other MCP client takes a pasted snippet.
HAR import (`import.har`) and code generation (`generate.code`) are tools of
this MCP server. The free workspace has neither.

Moving from `@apicircle/cli` or `@apicircle/mcp-server`? Both are deprecated. `apicircle <command>` is now `apicircle-lens <command>`, `apicircle-mcp` is now `apicircle-lens mcp`, and both need a CLI key. <!-- claims: history -->

## API Circle Studio, the free API workspace

Your API workspace is just code. Store it like code and review it like code.
Studio is the API client built on that: free, no account, and developed in the
open at [apicircle/studio](https://github.com/apicircle/studio).

- **A workspace is a Git repository.** Collections, environments and mock
  definitions are plain JSON that Studio pushes to a working branch in your own
  GitHub repository. It opens the pull request from inside the app, and walks
  you through a three-way merge when two people changed the same thing. Auth
  credentials are blanked before every push. Run history and sessions stay on
  the device.
- **Requests.** Seven HTTP methods and eight body types, GraphQL among them. It
  speaks HTTP only: no WebSocket, SSE, gRPC or scripts.
- **Authentication.** 15 schemes: Bearer, Basic, API key, custom header, six
  OAuth 2.0 grants (with PKCE, device code and token refresh), AWS Signature
  v4, Digest, NTLM, Hawk and JWT Bearer. A folder can carry auth that its
  requests inherit. Signing is checked against RFC and NIST reference vectors.
- **Environments and secrets.** Environments stack in priority order. Secrets
  live in the Secret Vault, encrypted with AES-256-GCM, and on the desktop the
  operating system's keychain wraps the key.
- **Mock servers.** Give Studio an OpenAPI, Swagger, Postman or Insomnia file
  and it serves a mock on `127.0.0.1`, with response rules, request validation,
  delays and multipliers. Studio desktop and the VS Code extension run mocks.
  The web app edits mock definitions but can't run one, because a browser tab
  can't listen on a port.
- **Execution plans and assertions.** Chain saved requests, pass a value
  extracted from one response into the next request, and grade each response
  with five kinds of assertion. Plans run in the app for free. Running one
  headlessly in CI is the Lens command line.
- **Imports.** OpenAPI / Swagger, Postman v2.1 collections, Postman
  environments, Insomnia v4 exports, cURL commands and API Circle exchange
  files.
- **History and snapshots.** Every request and plan you run is kept on the
  device with its headers, body and assertion results. Studio also snapshots
  the workspace before a push, a merge or an import.

### Where Studio runs

| Surface | Where it runs | Notes |
| --- | --- | --- |
| **Studio desktop** | Windows, macOS and Linux | Runs mock servers. [Latest release](https://github.com/apicircle/studio/releases/latest). The builds are unsigned, so expect a one-time OS prompt: see the [installing guide](https://github.com/apicircle/studio/blob/main/docs/installing.md). |
| **Studio web** | [studio.apicircle.dev](https://studio.apicircle.dev), any modern browser | Nothing to install. It can't run a mock server. |
| **VS Code extension** | VS Code 1.94 or later | The same workspace as YAML documents, and it runs mock servers. [Marketplace](https://marketplace.visualstudio.com/items?itemName=apicircle.apicircle-vscode) · [Open VSX](https://open-vsx.org/extension/apicircle/apicircle-vscode) |

The workspace engine and the mock engine are also on npm, as
`@apicircle/shared`, `@apicircle/core` and `@apicircle/mock-server-core`, under
the Studio licence below.

## What we believe

- **Own your data.** A workspace you can `cat`, `diff`, and `git log` is a
  workspace nobody can hold hostage.
- **Git is the sync layer.** Branches and pull requests already solved
  collaboration. We didn't need to reinvent that.

## Repositories

| Repo | What's in it |
| --- | --- |
| [**studio**](https://github.com/apicircle/studio) | The free API workspace, source-available: the desktop app, the web app, the VS Code extension, the mock engine and the core packages. It ships no CLI and no MCP server. |
| [**.github**](https://github.com/apicircle/.github) | This profile, the [Lens desktop releases](https://github.com/apicircle/.github/releases) and the [Lens CLI documentation](https://github.com/apicircle/.github/blob/main/docs/lens-cli/README.md). Lens's source isn't public. |

## Get involved

- ⭐ Star [**studio**](https://github.com/apicircle/studio) to follow the free workspace
- 🐛 File workspace bugs and ideas in [issues](https://github.com/apicircle/studio/issues)
- ✉️ Ask about Lens, a plan or a licence at [apicircle.dev/contact](https://apicircle.dev/contact)
- 🔒 Report a vulnerability through [apicircle.dev/security](https://apicircle.dev/security)

> **Licences.** API Circle Studio is released under a **custom source-available
> licence**: free for personal, educational, and non-commercial use (plus a
> 30-day commercial evaluation). It is _not_ an OSI-approved open-source
> licence; ongoing commercial use requires a separate licence. See
> [`LICENSE`](https://github.com/apicircle/studio/blob/main/LICENSE).
> API Circle Lens, `@apicircle-lens/cli` included, is proprietary, and the
> [Terms of Service](https://apicircle.dev/terms) govern its use.

---

<p align="center">
  <sub>
    <a href="https://apicircle.dev">apicircle.dev</a> ·
    <a href="https://apicircle.dev/pricing">Pricing</a> ·
    <a href="https://apicircle.dev/download">Download</a> ·
    <a href="https://apicircle.dev/docs">Docs</a>
  </sub>
</p>
