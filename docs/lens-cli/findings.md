# Findings

Every finding `apicircle-lens review` can raise, grouped by the severity it
lands in. The name in the first column is what the report prints and what
`--format json` carries.

[Back to the overview](README.md) · [How to read a report](review.md)

## The three severities

| In the Lens app | In the CLI | `--fail-on` | Meaning |
| --- | --- | --- | --- |
| **Critical** | `BREAKING` | `breaking` | A caller, or the API's access control, is affected now. |
| **Warning** | `WARN` | `warning` | A caller could notice. Read the change before you merge. |
| **Info** | `INFO` | `info` | Worth knowing. Routine in most pull requests. |

An endpoint takes the worst severity among its findings, and `--fail-on` looks
at endpoints. `--fail-on warning` therefore also fails on Critical.

Findings come from two places. **Code changes** compare the Code graph of the
base with the head's and need no Spec. **Drift from the Spec** needs
`--openapi`, and reports only what the pull request introduces.

## Critical

| Finding | Raised when | What a caller sees |
| --- | --- | --- |
| `endpoint-removed` | A route on the base is gone. | `404` where there was an answer. |
| `method-changed` | A route lost one HTTP method and gained another. | `404` or `405` on the old method. |
| `auth-removed` | Access control no longer runs before the endpoint. | The endpoint may answer callers it used to turn away. |
| `contract-drift` · `auth-missing-in-code` | The Spec requires auth the code does not enforce. | An endpoint the contract calls protected is open. |

## Warning

### Code changes

| Finding | Raised when | What a caller could see |
| --- | --- | --- |
| `auth-changed` | Access control was added, changed, or moved to another point in the chain. | `401` or `403` where there was none, or the reverse. |
| `validation-removed` | A request validator no longer runs. | Input that was rejected now reaches the handler. |
| `validation-changed` | A request validator was added or changed. | A request that passed is rejected with `400` or `422`, or the reverse. |
| `middleware-removed`, `middleware-changed` | Middleware was added, removed or changed. The reason says what it is for: request parsing, CORS, rate limiting, response headers or error handling. | Depends on the purpose: a rejected body, a failed preflight, a `429`, different headers or error bodies. |
| `middleware-removed`, `middleware-changed` (purpose not identified) | The same, for middleware the review cannot identify. | Unknown. It could be a guard: read the change. |
| `middleware-changed` (route registration) | The statement that registers the route changed, and nothing else explains it: a guard or option written inline may differ. | Unknown: read the change. |
| `response-shape-changed` | Response mapping, a DTO or a serialiser was added, removed or changed. | A different response body. |
| `new-side-effect` | A data write, or a step that runs after the response, was added, removed or changed. | Different writes, events or audit records. |
| `shared-impact` | Code that several endpoints run changed. The reason says how many it reaches. | The change applies to every endpoint that reaches it. |

### Routes against the Spec

Both are reported as `openapi-drift`.

| Finding | Raised when |
| --- | --- |
| `openapi-drift` · `implemented-not-documented` | The pull request adds a route the Spec does not describe. |
| `openapi-drift` · `documented-not-implemented` | The Spec describes a route that no code answers any more. |

### Fields against the Spec

All are reported as `contract-drift`, with the kind beside it in JSON.

| Kind | Raised when |
| --- | --- |
| `param-missing-in-code` | The Spec declares a path, query or header parameter the code does not handle. |
| `param-required-mismatch` | A parameter is required on one side and optional on the other. |
| `param-type-mismatch` | A parameter's type differs. |
| `cookie-missing-in-code` | The Spec declares a cookie the code does not read. |
| `cookie-required-mismatch` | A cookie is required on one side and optional on the other. |
| `cookie-type-mismatch` | A cookie's type differs. |
| `request-body-missing-in-code` | The Spec declares a request body the code does not accept. |
| `request-body-type-mismatch` | The request body, or a field in it, has a different type. |
| `request-body-field-missing` (required field) | A field the Spec requires is missing in code. |
| `response-type-mismatch` | A response body, or a field in it, has a different type. |
| `response-field-missing` (required field) | A response field the Spec requires is missing in code. |
| `nullable-mismatch` | The Spec allows `null` in a parameter or the request body, and the code refuses it. |

## Info

### Code changes

| Finding | Raised when |
| --- | --- |
| `endpoint-added` | A new route. |
| `handler-changed` | The handler, or code it calls, changed; or the route registration was added, removed, moved, or pointed at another handler. |
| `middleware-removed`, `middleware-changed` (logging) | A logger was added, removed or changed. No caller can see it. |

### Fields against the Spec

All are reported as `contract-drift`.

| Kind | Raised when |
| --- | --- |
| `param-missing-in-spec` | The code handles a parameter the Spec does not document. |
| `cookie-missing-in-spec` | The code reads a cookie the Spec does not document. |
| `request-body-missing-in-spec` | The code accepts a request body the Spec does not document. |
| `request-body-field-missing` (optional field) | An optional Spec field is missing in code. |
| `request-body-field-extra` | The code has a body field the Spec does not document. |
| `response-field-missing` (optional field) | An optional Spec response field is missing in code. |
| `response-field-extra` | The code returns a field the Spec does not document. |
| `response-status-missing-in-code` | The Spec documents a status the code does not return. |
| `response-status-missing-in-spec` | The code returns a status the Spec does not document. |
| `response-header-missing-in-spec` | The code sets a response header the Spec does not document. |
| `format-mismatch` | Both sides declare a format, and they differ. |
| `enum-mismatch` | Both sides declare an enum, and the value sets differ. |
| `auth-missing-in-spec` | The code enforces auth the Spec does not document. |

## How the review decides

**What a guard is for.** A block that runs before the handler is called access
control only when something identifies it: its name (`requireAuth`,
`jwtGuard`), the package it comes from (`passport`, `express-jwt`) or the folder
it lives in (`auth/`, `guards/`). Validation, CORS, rate limiting, logging and
the rest are read the same way. The JSON report says which in `purpose` and
`basis`. A block nothing identifies is reported as middleware of unidentified
purpose, at Warning, and never as access control.

**What it does not assert.** A type, a required flag or a format is compared
only when both the code and the Spec are known at that position. Where the code
could not be read with confidence, the review says nothing instead of guessing.
For the same reason, `auth-missing-in-code` and the header and cookie checks are
raised only for an endpoint whose auth, headers or cookies the review could
read.

**Two checks run one way.** A response header is reported only when the code
sets one the Spec does not document. Nullability is reported only on a request,
and only when the Spec allows `null` and the code refuses it.

**An `integer` in the Spec against a `number` in code is not drift.** The code
accepts everything the Spec promises. The reverse is reported.

## Choosing a level for your pipeline

| Level | A pull request fails on |
| --- | --- |
| `--fail-on breaking` | A removed route, a changed method, removed access control, or auth the Spec requires and the code lacks. |
| `--fail-on warning` | The above, and anything a caller could notice: changed access control or validation, new undocumented routes, type and required mismatches. |
| `--fail-on info` | Every finding, a new route and an edited handler included. Most pull requests fail at this level. |
| `--fail-on diffracting` | A new route the Spec does not document, or a documented route removed, and nothing else. |

`--fail-on breaking` is a common starting point. To fail on particular findings
instead of a level, see
[Gating on a finding type](review.md#gating-on-a-finding-type).
