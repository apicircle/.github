<p align="center">
  <img src="https://raw.githubusercontent.com/apicircle/.github/main/assets/logo.svg" alt="API Circle" width="120" height="120" />
</p>

<h1 align="center">API Circle</h1>

<p align="center">
  <strong>The API workspace that lives in your repo — and talks to your AI.</strong><br />
  Build, test, mock, and ship APIs the way you ship code:<br />
  branches, diffs, pull requests — and any MCP-capable AI client on the keyboard.
</p>

<p align="center">
  <a href="https://github.com/apicircle/studio"><img src="https://img.shields.io/github/stars/apicircle/studio?style=flat-square&color=8B5CF6&label=stars" alt="Stars" /></a>
  <a href="https://github.com/apicircle/studio/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-Source--Available-8B5CF6?style=flat-square" alt="License" /></a>
  <img src="https://img.shields.io/badge/platforms-Windows%20%7C%20macOS%20%7C%20Linux%20%7C%20Web-3B82F6?style=flat-square" alt="Platforms" />
  <img src="https://img.shields.io/badge/node-%3E%3D20-22C55E?style=flat-square" alt="Node >=20" />
  <a href="https://discord.gg/apicircle"><img src="https://img.shields.io/badge/discord-join-5865F2?style=flat-square&logo=discord&logoColor=white" alt="Discord" /></a>
</p>

---

### Why we built this

API clients haven't kept up. Postman pushed your collections into a SaaS silo.
Insomnia bolted AI on as an afterthought. Both leave you syncing data through
someone else's cloud, with no real story for the new shape of engineering work:
**branch-and-PR collaboration** and **AI agents that need to drive your tools.**

API Circle starts from a simpler premise — **your API workspace is just code.**
Store it like code. Review it like code. Let agents read and write it like
code. Everything else follows.

### The product

**[API Circle Studio](https://github.com/apicircle/studio)** is a desktop, web,
and CLI API workspace built on three load-bearing ideas:

#### 🔀 Your workspace is a Git repo

Collections, environments, and mock definitions are plain JSON pushed to your
own GitHub repo on a working branch. Teammates pull, branch, and merge them
like any other file. Per-device runtime state — history, sessions, UI — stays
local. Secrets are encrypted on-device with AES-GCM, wrapped by the OS keychain
on desktop. Nothing is uploaded to a third-party server, ever.

#### 🤖 Your workspace is an AI tool catalog

A built-in Model Context Protocol server exposes **71 tools** that any MCP
client can drive — Claude Desktop, Claude Code, ChatGPT, GitHub Copilot,
Cursor, Continue, Cline, Zed, Windsurf. The catalog covers request and folder
CRUD, environment authoring, assertions, execution plans, history, mock-server
lifecycle, codebase scanning, imports, code generation, and natural-language
authoring. Not a plugin. Not a wrapper. A first-class surface, equal to the UI.

#### 🧪 Your specs are runnable

Point API Circle at an OpenAPI, Swagger, Postman, or Insomnia file and get a
running HTTP mock on `localhost` in seconds. The Hono-based engine handles
`$ref` dereferencing, per-endpoint overrides (flip a `200` to a `503` to
exercise error paths), conditional response rules, request validation, and
response multipliers. Mock _definitions_ ship with the repo. Mock _runtime_
stays on your machine.

### What's in the box

- **17 authentication schemes**, all end-to-end functional — Bearer, Basic, API
  key, custom header, the full OAuth2 grant set (client credentials, auth code,
  PKCE, password, implicit, device flow, with auto-refresh), AWS SigV4, Digest,
  NTLM, Hawk, and JWT. Signing primitives verified against RFC and NIST
  reference vectors.
- **Imports that work** — cURL, OpenAPI/Swagger, Postman collections, Insomnia
  exports, and HAR files.
- **Code generation** for any saved request — cURL, fetch, Node (axios),
  Python (requests), Go, and Rust.
- **Environments** with priority ordering and cross-workspace variable sources.
- **Assertions and execution plans** to chain requests, runnable headlessly in
  CI with `apicircle run` (text, JSON, or JUnit reports).
- **Request history** with full headers, body previews, and assertion results.

### Pick your surface

| Surface          | Best for                                         |
| ---------------- | ------------------------------------------------ |
| **Desktop app**  | Day-to-day development (Windows / macOS / Linux) |
| **Web app**      | Quick access, zero install                       |
| **CLI**          | CI pipelines, terminals, headless agents         |
| **npm packages** | Embedding the engine in your own tooling         |

```bash
npx @apicircle/cli mock   ./openapi.yaml              # local mock server from a spec
npx @apicircle/cli mcp    --workspace ./workspace     # MCP server for any AI client
npx @apicircle/cli import ./postman_collection.json   # bring your existing collections
npx @apicircle/cli run    "Smoke Tests"               # run a saved plan in CI (text/json/junit)
```

### What we believe

- **Own your data.** A workspace you can `cat`, `diff`, and `git log` is a
  workspace nobody can hold hostage.
- **Git is the sync layer.** Branches and pull requests already solved
  collaboration. We didn't need to reinvent that.
- **AI is a first-class user.** Agents shouldn't have to scrape a UI. They get
  the same typed API the UI gets.
- **One engine, every surface.** Desktop, web, CLI, and embeddable packages
  share one workspace format and one mutation API.
- **Open standards only.** MCP for AI, Git for sync, OpenAPI / Postman /
  Insomnia / HAR for import. No proprietary lock-in.

### Repositories

| Repo                                              | What's in it                                                                      |
| ------------------------------------------------- | --------------------------------------------------------------------------------- |
| [**studio**](https://github.com/apicircle/studio) | The monorepo — desktop app, web app, CLI, MCP server, mock engine, core packages |

### Get involved

API Circle Studio is **pre-launch and self-funded.** Expect rough edges and the
occasional breaking change before v1.0 — and a team that genuinely reads every
issue.

- ⭐ Star [**studio**](https://github.com/apicircle/studio) to follow progress
- 📦 Grab a build from the [latest release](https://github.com/apicircle/studio/releases/latest)
- 🐛 File bugs and ideas in [issues](https://github.com/apicircle/studio/issues)
- 💬 Trade notes in the [Discord](https://discord.gg/apicircle)

> **License** — API Circle Studio is released under a **custom source-available
> license**: free for personal, educational, and non-commercial use (plus a
> 30-day commercial evaluation). It is _not_ an OSI-approved open-source
> license; ongoing commercial use requires a separate license. See
> [`LICENSE`](https://github.com/apicircle/studio/blob/main/LICENSE).

---

<p align="center">
  <sub>Own your APIs. Version them like code. Drive them with AI.</sub>
</p>
