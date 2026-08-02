---
name: Create, execute and correct an Unqork submission
description: >-
  Write data into an Unqork module, execute the module server-side, and handle the
  412 validation envelope correctly — including the duplicate-write risk created
  by the absence of any idempotency contract.
api: openapi/unqork-customer-api-openapi.yml
generated: '2026-07-31'
method: generated
source: >-
  Grounded in openapi/unqork-customer-api-openapi.yml,
  conventions/unqork-conventions.yml and errors/unqork-problem-types.yml
operations:
  - applicationsSubmissionDataModel
  - createModuleSubmissions
  - updateModuleSubmission
  - updateModuleSubmissions
  - executeModule
  - getModuleSubmissions
  - deleteModuleSubmission
  - restoreDeletedModuleSubmission
---

# Create, execute and correct an Unqork submission

Assumes you already have a bearer token — see
`unqork-authenticate-and-read-submissions.md`.

## Read this first: there is no idempotency

The Unqork Customer API has **no `Idempotency-Key` header**, no `If-Match`, no
`ETag` and no conditional requests of any kind. A retried `POST` **will create a
second submission**.

That single fact shapes this entire skill:

- Never blind-retry a create on a timeout or a connection reset.
- Put a caller-generated correlation value into the submission data or metadata
  on every create.
- On an uncertain outcome, **read back before retrying**: call
  `getModuleSubmissions` with a `metadataFilter` (or filter on your correlation
  field) and only re-create if nothing came back.

## Step 1 — learn the shape before you write

Do not guess field keys. Fetch the application's submission data model:

```
GET /applications/{applicationId}/submissionDataModel
```
→ `applicationsSubmissionDataModel`

This returns the schema the module expects. Field keys in Unqork are Creator-
defined component keys (e.g. `textfieldA`), not conventional snake_case names, so
this call is not optional.

You can also list the modules in the application first:

```
GET /applications/{applicationId}/modules
```
→ `applicationModules`

## Step 2 — create the submission

```
POST /modules/{moduleId}/submissions
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "data": { "<componentKey>": "<value>" },
  "metadata": { "<yourCorrelationKey>": "<uuid>" }
}
```
→ `createModuleSubmissions`

The operation creates **one or more** submissions. The `metadata` object is your
friend: it is filterable via `metadataFilter` on the read side, which is the only
practical way to make a create recoverable in the absence of idempotency.

Returns `201` with a `SavedSubmissionResponse`.

## Step 3 — execute the module

Creating a submission stores data. Running the module's logic — calculations,
validations, outbound integrations — is a separate call:

```
PUT /modules/{moduleId}/execute
Content-Type: application/json

{ "data": { ... } }
```
→ `executeModule`

Returns `200` with a `ModuleExecuteResponse` on success.

## Step 4 — handle 412, the Unqork failure code

`412 Precondition Failed` is the code that matters here. It means the module
**ran** but failed validation or execution, and the body is a structured envelope,
not a message string:

```json
{
  "validationErrors": [
    { "id": "...", "path": "...", "label": "...", "parent": "...", "message": "..." }
  ],
  "invalidNavigationPanels": { },
  "executionError": {
    "type": "...", "url": "...", "component": "...", "message": "...", "code": 0
  }
}
```

How to react:

- **`validationErrors[]` present** — the payload is wrong. Each entry's `path`
  addresses the exact component that rejected the value. Map `path` back to the
  key you sent, correct it, and re-execute. Do **not** re-create the submission;
  correct it with `updateModuleSubmission`.
- **`executionError` present** — the module ran but an integration failed.
  `executionError.url` is the outbound endpoint and `executionError.component` is
  the component that called it. This is usually a downstream problem, not a
  payload problem — retrying the same payload is reasonable, with backoff.
- **`invalidNavigationPanels` present** — a multi-panel form has panels that did
  not validate; treat like `validationErrors`.

## Step 5 — correct in place

```
PUT /modules/{moduleId}/submissions/{submissionId}
```
→ `updateModuleSubmission`

Update several at once:

```
PUT /modules/{moduleId}/submissions
```
→ `updateModuleSubmissions`

Watch `replaceData`. Setting it `true` **completely replaces** the submission data
with the object you pass, rather than merging. It is Administrator-only and it is
how partial updates silently become data loss. Default is merge — leave it alone
unless you mean it.

## Step 6 — deleting is recoverable, unless it isn't

```
DELETE /modules/{moduleId}/submissions/{submissionId}
```
→ `deleteModuleSubmission`

By default this is a **soft** delete. The record is hidden but recoverable:

```
POST /modules/{moduleId}/submissions/{submissionId}/restore
```
→ `restoreDeletedModuleSubmission`

Deleted submissions are visible on reads with `includeDeleted=true`.

**`destroy=true` makes the delete permanent.** There is no restore path after
that. Treat `destroy=true` as a human-approval operation, never an automatic one.

Also avoid `deleteModuleSubmissions` (the bulk, no-id form) in automated flows —
it deletes by filter, and a filter bug is unbounded.

## Safe-operation summary

| Operation | Retry-safe? | Reversible? |
|---|---|---|
| `getModuleSubmissions` | yes | n/a |
| `createModuleSubmissions` | **NO — duplicates** | via delete |
| `updateModuleSubmission` | yes (last write wins) | via revisions |
| `updateModuleSubmission` + `replaceData=true` | yes | **data loss risk** |
| `executeModule` | depends on side effects | no |
| `deleteModuleSubmission` | yes | yes, via restore |
| `deleteModuleSubmission?destroy=true` | yes | **NO** |

## Cross-references

- `errors/unqork-problem-types.yml` — the full 412 envelope
- `conventions/unqork-conventions.yml` — idempotency section
- `data-model/unqork-data-model.yml` — the entity graph
