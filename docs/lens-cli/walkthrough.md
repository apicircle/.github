# Walkthrough: one pull request, start to finish

A small API, one pull request with three changes in it, and what
`apicircle-lens` says about each. Every block of output on this page was
printed by version 2.1 on the project described here.

[Back to the overview](README.md)

## The project

A widgets API written with Express and TypeScript. It has six endpoints, all
mounted under `/api/v1`:

```text
src/app.ts                  mounts the routers under /api/v1
src/routes/widgets.ts       the widget routes and their guards
src/handlers/widgets.ts     the handlers
src/middleware/auth.ts      requireBearer and requireApiKey
openapi.json                the Spec: paths such as /widgets, server URL /api/v1
```

## 1. Build the Code graph

On the default branch:

```sh
apicircle-lens codegraph index
```

```text
Indexed 6 endpoint(s) — detected: express.
  → .apicircle/workspace-lens-desktop/codegraph/endpoints.json
  → .apicircle/workspace-lens-desktop/codegraph/index.json
  → added the parser-cache rule to .gitignore
```

Commit what it wrote. This is the baseline every later review compares with.

```sh
git add .apicircle .gitignore
git commit -m "Add the Code graph"
```

## 2. See what it mapped

```sh
apicircle-lens codegraph list-endpoints
```

```text
DELETE /api/v1/widgets/:widgetId  (express, high)
GET    /api/v1/health  (express, high)
GET    /api/v1/widgets  (express, high)
GET    /api/v1/widgets/:widgetId  (express, high)
PATCH  /api/v1/widgets/:widgetId  (express, high)
POST   /api/v1/widgets  (express, high)
```

One endpoint, with the code it runs:

```sh
apicircle-lens codegraph endpoint "DELETE /api/v1/widgets/:widgetId"
```

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

The review in step 4 works from exactly this: which guards run before the
handler, what the handler calls, and what several endpoints share.

## 3. The pull request

A branch named `feature/widget-export` makes three changes.

```diff
 // src/routes/widgets.ts
+widgetsRouter.get('/export', exportWidgets);
 widgetsRouter.get('/:widgetId', getWidget);
-widgetsRouter.post('/', requireBearer, createWidget);
+widgetsRouter.post('/', createWidget);
```

```diff
 // src/handlers/widgets.ts, in deleteWidget
-  res.status(204).end();
+  res.status(200).json({ deleted: widgetId });
```

1. It adds `GET /widgets/export`, with a new `exportWidgets` handler that is not
   shown here.
2. It drops `requireBearer` from `POST /widgets`, by accident, while editing the
   line.
3. It makes `DELETE` answer `200` with a body where it answered `204`.

None of the three touches `openapi.json`.

## 4. Review it

Take the Code graph of the default branch, then review the checkout against it
and against the Spec:

```sh
git show main:.apicircle/workspace-lens-desktop/codegraph/endpoints.json > ../base.json
apicircle-lens review --base ../base.json --openapi openapi.json --spec-base-path /api/v1 --fail-on breaking
```

`--spec-base-path /api/v1` is there because the code registers
`/api/v1/widgets` and the Spec lists `/widgets`.

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

```sh
echo $?
```

```text
1
```

## 5. Read the result

**`POST /api/v1/widgets` is Critical.** `requireBearer` ran before this endpoint
on the base and does not on the branch. The diff showed one edited line; the
review says what it means: the endpoint may now answer callers it used to turn
away. This is the finding that failed the build.

**`DELETE /api/v1/widgets/:widgetId` is a Warning**, for two reasons. The
handler that writes data changed (`new-side-effect`), and the code and the Spec
now disagree about the response: the Spec promises `204`, the code returns
`200`. Each status line is Info on its own; the endpoint is a Warning because of
the changed write.

**`GET /api/v1/widgets/export` is a Warning.** A new route is Info
(`endpoint-added`). A new route the Spec does not describe is a Warning
(`openapi-drift`).

**The verdict is `diffracting`** because of that same route: the API grew in a
direction the contract does not cover.

The other four endpoints are not listed. Nothing the pull request did reaches
them.

## 6. What the same run does without the Spec

Drop `--openapi`, and the review can only compare code with code:

```text
PR Review
⛔ [BREAKING] POST /api/v1/widgets — auth-removed
    • Auth/access-control `requireBearer` removed in the pre-request chain
⚠️ [WARN] DELETE /api/v1/widgets/:widgetId — new-side-effect
    • Data-write `deleteWidget` changed in the handler
ℹ️ [INFO] GET /api/v1/widgets/export — endpoint-added
    • New route GET /api/v1/widgets/export

3 endpoint(s): 1 breaking, 1 added, 0 removed, 2 changed.

Focus shift
  Verdict: in-focus
  Focus: api +1
```

The removed guard is still caught. The `204` that became a `200` is gone from
the report, the undocumented route reads as routine, and the verdict is
`in-focus`. Pass the Spec.

## 7. Fix it and run again

The author restores the guard, and decides the other two changes are intended.
Intended changes belong in the contract, so `openapi.json` gets the new `200`
response for `DELETE` and a `GET /widgets/export` operation.

```sh
apicircle-lens verify --spec openapi.json
```

```text
✓ Contract is valid (3.0.3) — 9 operation(s), 0 warning(s).
```

```sh
apicircle-lens review --base ../base.json --openapi openapi.json --spec-base-path /api/v1 --fail-on breaking
```

```text
PR Review
⚠️ [WARN] DELETE /api/v1/widgets/:widgetId — new-side-effect
    • Data-write `deleteWidget` changed in the handler
ℹ️ [INFO] GET /api/v1/widgets/export — endpoint-added
    • New route GET /api/v1/widgets/export

2 endpoint(s): 0 breaking, 1 added, 0 removed, 1 changed.

Focus shift
  Verdict: in-focus
  Focus: api +1
  Toward spec: GET /api/v1/widgets/export
```

The command exits with `0`. `POST` is no longer listed, the drift lines are
gone, and the new route now counts as a move toward the Spec.

One Warning remains: the handler that writes data did change. That is a fact
about the pull request for a reviewer to read, and it is why `--fail-on warning`
would still stop this build while `--fail-on breaking` lets it through. Choose
the level that matches how your team reviews: see
[Choosing a level](findings.md#choosing-a-level-for-your-pipeline).

## 8. Put it in the pipeline

The two commands of step 4 are the whole job. Add `--comment` and the review is
posted on the pull request, as this table:

| Severity | Endpoint | Flags |
| --- | --- | --- |
| ⛔ BREAKING | `POST /api/v1/widgets` | auth-removed |
| ⚠️ WARN | `DELETE /api/v1/widgets/:widgetId` | new-side-effect, contract-drift |
| ⚠️ WARN | `GET /api/v1/widgets/export` | endpoint-added, openapi-drift |

A complete pipeline file for your CI service is in the
[CI recipes](ci/README.md).
