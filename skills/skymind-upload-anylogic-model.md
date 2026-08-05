---
name: Upload an AnyLogic model to Pathmind
description: >-
  Authenticate against the Pathmind API, find or choose a project, and upload an AnyLogic
  simulation model to create a reinforcement-learning experiment.
api: openapi/skymind-pathmind-openapi-original.yml
operations:
- getProjects
- uploadAlModel
generated: '2026-08-05'
method: generated
source: openapi/skymind-pathmind-openapi-original.yml
status: historical
---

# Upload an AnyLogic model to Pathmind

> **Service status — read this first.** Pathmind ceased operating on 2021-11-17 and the
> base host `https://api.pathmind.com` refuses connections (probed 2026-08-05). Do not
> attempt these calls against the live host expecting success. This skill documents a real,
> first-party published contract and is retained as a historical record. See
> `lifecycle/skymind-lifecycle.yml`.

## Authentication

Every operation requires an API token in a header. There is no OAuth and there are no
scopes.

```
X-PM-API-TOKEN: <your Pathmind access token>
```

The token came from the account page in the Pathmind web app. The first-party Python
client (`pip install pathmind`) reads it from the `PATHMIND_TOKEN` environment variable.

A missing or unknown token returns `401` with the body
`{"timestamp":"…","error":"no pathmind api user for api key","status":401,"path":"/al/upload"}`.
See `errors/skymind-problem-types.yml`.

## Step 1 — list the account's projects (`getProjects`)

```
GET https://api.pathmind.com/projects
X-PM-API-TOKEN: <token>
```

Returns `200` with an array of projects, each `{id, name, is_archived, date_created,
date_last_activity}`. The response is **unbounded** — there is no pagination, so read the
whole array. Filter out `is_archived: true` before offering a project as an upload target.

Choose the `id` of the project you want to upload into, or skip this step entirely and let
Pathmind create a new project.

## Step 2 — upload the model (`uploadAlModel`)

```
POST https://api.pathmind.com/al/upload
X-PM-API-TOKEN: <token>
Content-Type: multipart/form-data
```

Parts:

| part | required | type | notes |
|---|---|---|---|
| `file` | yes | binary | the exported AnyLogic simulation model |
| `projectId` | no | integer | omit to create a new project |

On success the API returns `201` with a `location` response header pointing at the web
app view for the created experiment, in the published form
`https://app.pathmind.com//editGoals/35323?experiment=33093` (the doubled slash is
verbatim from the spec's example). Parse `experiment=` out of that URL to get the new
experiment id — the API defines no response body and no Experiment schema.

## Rules an agent must follow

- **Never retry the upload blindly.** There is no idempotency key. `uploadAlModel` is a
  non-idempotent POST: a repeat creates a *second* experiment. On a timeout, call
  `getProjects` (or check the project in the web app) before re-uploading. See
  `conventions/skymind-conventions.yml`.
- **`403` means the token authenticated but is not allowed to touch that `projectId`.**
  Drop `projectId` and let Pathmind create a new project, or use a token for the owning
  account. Do not loop.
- **Read the `location` header, not the body.** The 201 has no JSON payload.
- **Do not assume pagination, filtering, expansion, metadata or rate-limit headers.** None
  of them exist in this API.
