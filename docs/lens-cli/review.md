# The review command

`apicircle-lens review` finds PR Review drift: what a pull request changes in
your API, and where that moves it away from the Spec. This page covers what it
compares, every line it prints, how it gates a pipeline and how it posts to a
pull request.

[Back to the overview](README.md) · [Every finding it can raise](findings.md)

## What it compares

A review has two sides, **base** and **head**, and each is a Code graph.

| Side | Where it comes from |
| --- | --- |
| Base | `--base <file>`: the `endpoints.json` committed on the branch you merge into. |
| Head | The checkout, indexed on the spot. Or `--head <file>` for a Code graph you already have. |

```sh
git show origin/main:.apicircle/workspace-lens-desktop/codegraph/endpoints.json > ../base.json
apicircle-lens review --base ../base.json --openapi openapi.yaml
```

Endpoints are matched by method and route. For each one on both sides, the
review compares the code it runs, block by block: guards, validators, the
handler, what the handler calls, and code it shares with other endpoints. A
block is compared by its content, and a change of line endings alone is not a
finding.

### Reviewing a pull request by number

```sh
apicircle-lens review --pull-request 42 --repo acme/api --openapi openapi.yaml
```

This reads the Code graph committed at the pull request's base and at its head
from your Git host, so nothing is indexed and no checkout is needed. Both
branches must have a committed Code graph, and the head's must be current: when
the pull request changes source files but did not re-index, the review warns
that drift from those changes is missing. Two options change that:

| Option | Effect |
| --- | --- |
| `--require-fresh-index` | Exit with `3`, reporting and posting nothing, when the head's Code graph is not current. |
| `--local-fallback` | Index this checkout at the pull request's base and head commits when its head has no current Code graph. The checkout needs both commits. |

In CI, `--base` with an indexed checkout is the simpler path, because the head
is always current. The [CI recipes](ci/README.md) use it.

## The Spec

`--openapi <file>` takes your OpenAPI contract as JSON or YAML. With it, the
review adds two checks:

- **Presence.** A route the code answers and the Spec does not describe, or the
  reverse.
- **Contract.** For a route on both sides: parameters, cookies, the request
  body, responses by status, response headers and auth.

Only drift the pull request **introduces** is reported. A disagreement that is
already on the base is not this pull request's doing and is left out.

**Without `--openapi`, the review still runs and reports code changes only.** It
prints nothing to say the Spec was skipped, so check that your pipeline passes
the option.

### When your routes have a prefix the Spec omits

A Spec often lists `/widgets` under a server URL of `/api/v1`, while the code
registers `/api/v1/widgets`. The review does not guess. If the Spec declares a
base path and you did not align it, it says so on stderr:

```text
Note: the spec declares a base path of "/api/v1" that its `paths` omit. If your code mounts routes under it, re-run with --spec-base-path /api/v1 — otherwise every route will read as missing.
```

| Option | Use it when |
| --- | --- |
| `--spec-base-path /api/v1` | The code includes a prefix the Spec's paths omit. |
| `--spec-strip-base /v2` | The Spec's paths carry a prefix the code does not declare. |

Without the right prefix no route matches, and the contract checks find nothing
to compare.

> **Git Bash on Windows** rewrites an argument such as `/api/v1` into a Windows
> path before the CLI sees it. Run the command from PowerShell, or set
> `MSYS_NO_PATHCONV=1` first.

## Reading the text report

This is the default output, from the [walkthrough](walkthrough.md):

```text
PR Review
⛔ [BREAKING] POST /api/v1/widgets — auth-removed
    • Auth/access-control `requireBearer` removed in the pre-request chain
⚠️ [WARN] DELETE /api/v1/widgets/:widgetId — new-side-effect, contract-drift
    • Data-write `deleteWidget` changed in the handler
    • Spec documents response 204, not returned in code.
    • Code returns response 200, not documented in the spec.
⚠️ [WARN] GET /api/v1/widgets/export — endpoint-added, openapi-drift
    • New route GET /api/v1/widgets/export
    • Implemented but missing from the OpenAPI spec

3 endpoint(s): 1 breaking, 1 added, 0 removed, 2 changed.

Focus shift
  Verdict: diffracting
  Focus: api +1
  Away from spec (diffraction): GET /api/v1/widgets/export
```

| Part | Meaning |
| --- | --- |
| `⛔ [BREAKING]`, `⚠️ [WARN]`, `ℹ️ [INFO]` | The endpoint's severity: the worst of its findings. The Lens app shows `BREAKING` as **Critical**. |
| `POST /api/v1/widgets` | The affected endpoint. Endpoints the change does not touch are not listed. |
| `— auth-removed` | The finding types raised for it. Each is described in [Findings](findings.md). |
| `• …` | One reason per line, naming the code it is about. |
| `✓ human-verified` after a reason | A drift your team's verified schema confirms. |
| `✓ resolved (matches spec)` after a reason | A drift your team corrected: the verified schema agrees with the Spec. |
| `3 endpoint(s): 1 breaking, 1 added, 0 removed, 2 changed.` | Affected endpoints, how many are breaking, and how they split into new, removed and changed. |
| `No endpoint impact detected.` | Printed instead of the list when nothing changed. |

Endpoints are listed worst first, then by method and path.

### Focus shift

The last block says which way the API moved.

| Line | Meaning |
| --- | --- |
| `Verdict: in-focus` | No route moved away from the Spec. Always this when no Spec was given. |
| `Verdict: diffracting` | The change added a route the Spec does not document, or removed one it does. |
| `Focus: api +1` | Areas of the API that grew or shrank, by the Spec's tag when it has one and by the first path segment otherwise. |
| `Toward spec:` | Routes that moved toward the contract: a documented route implemented, or an undocumented one removed. |
| `Away from spec (diffraction):` | The routes behind a `diffracting` verdict. |

## The Markdown report

`--format markdown` prints what `--comment` posts:

```markdown
## PR Review

**3** endpoint(s) · **1** breaking · 1 added · 0 removed · 2 changed

| Severity | Endpoint | Flags |
| --- | --- | --- |
| ⛔ BREAKING | `POST /api/v1/widgets` | auth-removed |
| ⚠️ WARN | `DELETE /api/v1/widgets/:widgetId` | new-side-effect, contract-drift |
| ⚠️ WARN | `GET /api/v1/widgets/export` | endpoint-added, openapi-drift |

## Focus shift

**Verdict:** `diffracting`

**Focus:** `api` +1

**Away from spec (diffraction):** `GET /api/v1/widgets/export`
```

It has one row per endpoint and leaves the reasons out. Use `--submit` to put
each reason on the line of code it is about.

## The JSON report

`--format json` prints the whole review as one object, for a pipeline that wants
to decide for itself.

```json
{
  "schemaVersion": "1.1.0",
  "base": {},
  "head": { "commit": "af116e4c46efe1860d6e6b9eb8b153f57edff923" },
  "findings": [
    {
      "endpointId": "ep_2d113154660c500d",
      "method": "DELETE",
      "path": "/api/v1/widgets/:widgetId",
      "flags": ["new-side-effect", "contract-drift"],
      "severity": "warning",
      "details": [
        {
          "flag": "new-side-effect",
          "message": "Data-write `deleteWidget` changed in the handler",
          "filePath": "src/handlers/widgets.ts",
          "symbolName": "deleteWidget",
          "startLine": 79,
          "endLine": 88,
          "change": "changed",
          "side": "head"
        },
        {
          "flag": "contract-drift",
          "driftKind": "response-status-missing-in-code",
          "severity": "info",
          "message": "Spec documents response 204, not returned in code.",
          "filePath": "src/handlers/widgets.ts",
          "symbolName": "deleteWidget",
          "startLine": 79,
          "endLine": 88
        }
      ]
    }
  ],
  "summary": {
    "endpointsReviewed": 3,
    "added": 1,
    "removed": 0,
    "changed": 2,
    "breaking": 1,
    "flags": { "auth-removed": 1, "new-side-effect": 1, "contract-drift": 1, "endpoint-added": 1, "openapi-drift": 1 }
  },
  "shift": {
    "foci": [{ "focus": "api", "baseOps": 6, "headOps": 7, "delta": 1 }],
    "added": [{ "method": "GET", "route": "/api/v1/widgets/export" }],
    "removed": [],
    "towardSpec": [],
    "awayFromSpec": [{ "method": "GET", "route": "/api/v1/widgets/export" }],
    "verdict": "diffracting"
  }
}
```

The example is shortened to one endpoint and the fields below.

| Field | Meaning |
| --- | --- |
| `findings[]` | One entry per affected endpoint, worst first. |
| `findings[].severity` | `breaking`, `warning` or `info`: the worst of the endpoint's details. |
| `findings[].flags` | The finding types raised for the endpoint. |
| `details[].flag` | The finding type of one reason. |
| `details[].driftKind` | Which disagreement with the Spec it is. Set on `contract-drift` and `openapi-drift` only. |
| `details[].severity` | Set when a reason carries its own severity: every `contract-drift`, and middleware a caller could notice. When absent, the flag's severity in [Findings](findings.md) applies. |
| `details[].filePath`, `startLine`, `endLine`, `symbolName` | The code the reason is about. |
| `details[].change` | `added`, `removed`, `changed` or `moved`. |
| `details[].side` | `base` when the lines belong to the base: the code was removed. |
| `details[].before`, `after` | Where the code sits on each side. |
| `details[].purpose`, `basis` | What a guard or middleware is for, and what that reading rests on: its `name`, the `module` it comes from, the `path` it lives in, or its `structure`. |
| `details[].verified`, `resolved` | Set when your team's verified schema confirms a drift, or corrects it. |
| `summary` | The totals the text report prints, and a count per finding type. |
| `shift` | The focus shift. `verdict` is `in-focus` or `diffracting`. |

### Gating on a finding type

`--fail-on` gates on severity. To fail on particular finding types, read the
JSON with a tool such as `jq`. This step fails when the pull request changes
access control or removes validation, whatever their severity:

```sh
apicircle-lens review --base base.json --openapi openapi.yaml --format json > review.json
jq -e '[.findings[].details[] | select(.flag == "auth-changed" or .flag == "validation-removed")] | length == 0' review.json
```

And this one fails on a required response field the code no longer returns:

```sh
jq -e '[.findings[].details[] | select(.driftKind == "response-field-missing" and .severity == "warning")] | length == 0' review.json
```

Run the JSON report as its own step. With `--comment` or `--submit`, a line
naming the posted URL follows the report on stdout.

## Gating a pipeline

| Option | The command exits with `1` when |
| --- | --- |
| `--fail-on breaking` | Any endpoint is breaking. |
| `--fail-on warning` | Any endpoint is breaking or a warning. |
| `--fail-on info` | Anything at all was found. |
| `--fail-on diffracting` | The focus shift verdict is `diffracting`. It needs `--openapi`: without a Spec the verdict is always `in-focus`. |
| `--fail-on-breaking` | The same as `--fail-on breaking`. |

Without `--fail-on`, the review reports and exits with `0` whatever it finds.

`--fail-on` takes the engine's words. `breaking` is the level the Lens app calls
Critical, and `--fail-on critical` is rejected:

```text
Invalid --fail-on value "critical" — expected: breaking, warning, info, or diffracting.
```

### Gating on the schema your team verified

In the Lens app you can verify an endpoint's schema where the extracted one is
wrong or incomplete. `--use-verified` gates on that verified contract instead of
the extracted one: a drift your team resolved stops failing the build, and a
drift only the verified schema reveals starts failing it. The printed report
does not change. It needs `--openapi`.

## Exit codes

| Code | Meaning |
| --- | --- |
| `0` | The review ran, and no `--fail-on` level was met. |
| `1` | The `--fail-on` level was met, posting failed, or the CLI key was not accepted. |
| `2` | The review could not run or post as asked: a file that cannot be read, a missing token or `--repo`, the Git host's API failing, an invalid option value, or a post refused on a public repository. |
| `3` | `--require-fresh-index` was given, and the Code graph at the pull request's head is not current. |

A pipeline can therefore tell drift (`1`) from a review that never ran (`2`).

## Posting to a pull request

```sh
apicircle-lens review --base base.json --openapi openapi.yaml \
  --comment --repo acme/api --pr 42
```

| Option | What it posts |
| --- | --- |
| `--comment` | The Markdown report as one comment. A later run updates that comment. |
| `--submit` | A review with a comment on each drifted line. `--event` sets the verdict: `COMMENT` (the default), `APPROVE` or `REQUEST_CHANGES`. |

After posting it prints the address:

```text
Posted review to https://github.com/acme/api/pull/42#issuecomment-2400000000
```

```text
Submitted COMMENT review to https://github.com/acme/api/pull/42#pullrequestreview-2400000000 (3 inline comments, 0 off-diff).
```

`off-diff` counts reasons about lines the pull request's diff does not show,
which a host will not accept a comment on.

### Hosts

| `--host` | `--repo` | Token variable | `--base-url` |
| --- | --- | --- | --- |
| `github` (the default) | `owner/name` | `GITHUB_TOKEN` or `APICIRCLE_LENS_GIT_TOKEN` | Only for GitHub Enterprise Server: `https://ghe.example.com/api/v3` |
| `gitlab` | `group/project`, subgroups included | `APICIRCLE_LENS_GIT_TOKEN` | Only when self-managed: `https://gitlab.example.com/api/v4` |
| `bitbucket` | `workspace/repo` | `APICIRCLE_LENS_GIT_TOKEN` | Not needed |
| `azure-devops` | `project/repo` | `APICIRCLE_LENS_GIT_TOKEN` | Always: `https://dev.azure.com/<organisation>` |

The token is taken from `--token`, else from a host you connected with
`apicircle-lens git connect`, else from the variable. For Bitbucket Cloud, an
access token is passed as it is, and an Atlassian API token as
`<account email>:<token>`.

GitLab has no request-changes verdict. `--submit --event REQUEST_CHANGES` there
posts the review as a comment and says so.

### Public repositories

A review can name access control that moved or a route the Spec never
documented. On a public repository anyone could read that, so the CLI checks
before it posts:

```text
acme/api is a PUBLIC repository — this review will be readable by anyone, not only people with access to the code.
Refusing to post: acme/api is public and 2 findings in this review describe a security weakness (access control that moved, auth the code does not enforce, or a route implemented but absent from the spec). Re-run with --omit-security-findings to post everything else, or --allow-public-security-findings to publish them deliberately.
```

| Option | Effect |
| --- | --- |
| `--omit-security-findings` | Posts everything else. The printed report still lists them. |
| `--allow-public-security-findings` | Posts them on a public repository. |

A private repository is never affected.

## Messages you may see

| Message | Exit | What to do |
| --- | --- | --- |
| `Provide --base <file> (a base-ref Code graph), or --pull-request <n> --repo <owner/name> …` | `2` | Pass one of the two. |
| `Could not read the base Code graph: base.json` | `2` | The `git show` step wrote nothing: the base branch has no committed Code graph at that path. |
| `Could not read the OpenAPI spec: openapi.yaml` | `2` | Check the path passed to `--openapi`. |
| `Could not parse the OpenAPI spec: …` | `2` | The file is not valid JSON or YAML. Try `apicircle-lens verify --spec`. |
| `--comment needs a token. Pass --token, run `apicircle-lens git connect github`, or set GITHUB_TOKEN or APICIRCLE_LENS_GIT_TOKEN.` | `2` | Set the token variable for your host. The review itself was still printed. |
| `--comment needs --repo <owner/name>.` | `2` | Add `--repo` in your host's shape. |
| `--comment needs --pr <number>.` | `2` | Add the pull request number. |
| `Could not post the review comment: …` | `1` | The host refused: check the token's permissions. |
| `--use-verified needs --openapi …; ignoring it and gating on the extracted drift.` | unchanged | Add `--openapi`. |
| `Pull request #42 has no Code graph committed at its head …` | `2` | Run `codegraph index` on that branch and commit the folder, or review a checkout with `--base`. |
