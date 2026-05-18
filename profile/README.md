<p align="center">
  <img src="https://raw.githubusercontent.com/apicircle/.github/main/assets/logo.svg" alt="APICircle" width="120" height="120" />
</p>

<h1 align="center">APICircle</h1>

<p align="center">
  <strong>A Git-native, AI-native API workspace.</strong><br />
  Build, test, mock, and ship APIs — collaborate through pull requests,<br />
  and let any AI client drive your workspace.
</p>

<p align="center">
  <a href="https://github.com/apicircle/studio"><img src="https://img.shields.io/github/stars/apicircle/studio?style=flat-square&color=8B5CF6&label=stars" alt="Stars" /></a>
  <a href="https://github.com/apicircle/studio/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-Source--Available-8B5CF6?style=flat-square" alt="License" /></a>
  <img src="https://img.shields.io/badge/platforms-Windows%20%7C%20macOS%20%7C%20Linux%20%7C%20Web-3B82F6?style=flat-square" alt="Platforms" />
  <img src="https://img.shields.io/badge/node-%3E%3D20-22C55E?style=flat-square" alt="Node >=20" />
  <a href="https://discord.gg/apicircle"><img src="https://img.shields.io/badge/discord-join-5865F2?style=flat-square&logo=discord&logoColor=white" alt="Discord" /></a>
</p>

---

### What is APICircle?

**APICircle Studio** is an API client in the spirit of Postman and Insomnia —
rebuilt around two ideas the others miss:

- **Your workspace is a Git repo.** Collections, environments, and mock
  definitions are plain JSON, pushed to your own GitHub repo on a working
  branch. Teams collaborate the way they collaborate on code: branches, diffs,
  pull requests, review.
- **Your workspace is an AI tool catalog.** A built-in Model Context Protocol
  (MCP) server exposes **71 tools**, so Claude, ChatGPT, Cursor, Copilot, and
  any other MCP client can read, author, and run requests for you.

No cloud account. No vendor lock-in. Your data stays on your machine and in
your repo.

### What we provide

**🔀 Git-backed workspaces** — A workspace is plain JSON: the shared collection
tree, environments, and mock definitions push to a GitHub repo on a working
branch, and teammates pull, branch, and merge it like any other file.
Per-device runtime state (history, sessions, UI) stays local. Secrets are
encrypted on-device (AES-GCM via WebCrypto, wrapped by the OS keychain on
desktop) — nothing is uploaded to a third-party server.

**🤖 AI integration via MCP** — The bundled `@apicircle/mcp-server` speaks the
open [Model Context Protocol](https://modelcontextprotocol.io) over stdio, so it
works with Claude Desktop, Claude Code, ChatGPT, GitHub Copilot, Cursor,
Continue, Cline, Zed, and Windsurf. Its **71-tool catalog** covers
request/folder CRUD, environment authoring, assertions, execution plans,
history, mock-server lifecycle, codebase scanning, imports, code generation,
and natural-language authoring.

**🧪 Local mock servers** — Point APICircle at an OpenAPI / Swagger / Postman /
Insomnia file and get a running HTTP mock on `localhost` in seconds. The
Hono-based engine handles `$ref` dereferencing, per-endpoint overrides (flip a
`200` to a `503` to test error paths), conditional response rules, request
validation, and response multipliers.

**🧰 A complete request toolkit**

- **17 authentication schemes**, all end-to-end functional — Bearer, Basic,
  API key, custom header, the full OAuth2 grant set (client credentials, auth
  code, PKCE, password, implicit, device flow, with auto-refresh), AWS SigV4,
  Digest, NTLM, Hawk, and JWT — signing primitives verified against the
  relevant RFC / NIST reference vectors.
- **Import what you already have** — cURL, OpenAPI/Swagger, Postman
  collections, Insomnia exports, and HAR files.
- **Generate client code** from any saved request — cURL, fetch, Node (axios),
  Python (requests), Go, and Rust.
- **Environments** with priority ordering and cross-workspace variable sources.
- **Assertions** and multi-step **execution plans** to chain requests.
- **Request history** with full headers, body previews, and assertion results.

### Use it your way

| Surface          | Best for                                         |
| ---------------- | ------------------------------------------------ |
| **Desktop app**  | Day-to-day development (Windows / macOS / Linux) |
| **Web app**      | Quick access, zero install                       |
| **CLI**          | CI pipelines, terminals, headless agents         |
| **npm packages** | Embedding the engine in your own tooling         |

```bash
npx @apicircle/cli mock   ./openapi.yaml              # local mock server from a spec
npx @apicircle/cli mcp    --workspace ./workspace     # MCP server for any AI client
npx @apicircle/cli import ./postman_collection.json   # import an existing collection
```

### Core principles

- **Own your data** — the workspace is JSON you can read, diff, and back up
- **Git over cloud sync** — branch, diff, and PR your API collections
- **AI-native, not AI-bolted-on** — the MCP server is a first-class surface, not a plugin
- **Runs everywhere you do** — one engine behind the desktop app, web app, CLI, and npm packages
- **Built on open standards** — MCP for AI, Git for sync, OpenAPI / Postman / Insomnia / HAR for import

### Repositories

| Repo                                              | Description                                                                       |
| ------------------------------------------------- | --------------------------------------------------------------------------------- |
| [**studio**](https://github.com/apicircle/studio) | Main monorepo — desktop app, web app, CLI, MCP server, mock engine, core packages |

### Get involved

APICircle Studio is **pre-launch and self-funded** — expect rough edges and
occasional breaking changes before v1.0.

- ⭐ Star [**studio**](https://github.com/apicircle/studio) to follow progress
- 🐛 Report bugs and ideas in [issues](https://github.com/apicircle/studio/issues)
- 📦 Grab a build from the [latest release](https://github.com/apicircle/studio/releases/latest)
- 💬 Join the [Discord](https://discord.gg/apicircle)

> **License** — APICircle Studio is released under a **custom source-available
> license**: free for personal, educational, and non-commercial use (plus a
> 30-day commercial evaluation). It is _not_ an OSI-approved open-source
> license; ongoing commercial use requires a separate license. See
> [`LICENSE`](https://github.com/apicircle/studio/blob/main/LICENSE).

---

<p align="center">
  <sub>Own your APIs. Version them like code. Drive them with AI.</sub>
</p>
