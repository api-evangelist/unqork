---
name: Authenticate to Unqork and read submission data
description: >-
  Get an OAuth 2.0 bearer token from an Unqork environment and read module or
  workflow submissions out of it, paging and filtering correctly. This is the
  foundation skill — every other Unqork skill assumes it.
api: openapi/unqork-customer-api-openapi.yml
generated: '2026-07-31'
method: generated
source: >-
  Grounded in openapi/unqork-customer-api-openapi.yml,
  conventions/unqork-conventions.yml, errors/unqork-problem-types.yml and
  rate-limits/unqork-rate-limits.yml
operations:
  - getModuleSubmissions
  - getModuleSubmission
  - getAllSubmissions
  - getWorkflowSubmissions
  - getMergedSubmissions
  - getModuleSubmissionRevisions
---

# Authenticate to Unqork and read submission data

## Before you start

Unqork is multi-tenant. There is no shared `api.unqork.com` — every environment
has its own subdomain, and the base URL is:

```
https://{subdomain}.unqork.io/api/1.0
```

You need the customer's subdomain and an API credential minted in
**Administration → API Access Management**. Credentials come in two flavours and
they are not interchangeable:

- **Express** credentials — for end-user-facing data (submissions).
- **Creator** credentials — for design-time resources (modules, applications,
  promotions).

Each credential is bound to at least one Express or Creator **role**. Unqork has
no OAuth scopes — the single declared scope is a placeholder (`none: N/A`), and
what a token can do is entirely determined by its roles. To get least privilege,
create a credential with a narrow role; you cannot narrow it at token-request
time.

## Step 1 — get a bearer token

```
POST https://{subdomain}.unqork.io/api/1.0/oauth2/access_token
Authorization: Basic base64({clientId}:{clientSecret})
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

Use the **client_credentials** grant. Unqork also supports the OAuth 2.0
**password** grant (username/password), but it must be switched on per
environment and it is deprecated by OAuth 2.1 — do not reach for it unless the
customer has no other option.

The response carries an `access_token`. **It expires after one hour.** Cache it,
track the expiry, and re-request rather than re-requesting on every call — five
failed client-secret attempts LOCK the credential for 30 minutes.

Send it on every subsequent request:

```
Authorization: Bearer {access_token}
```

## Step 2 — read submissions

A **submission** is the business record — the data a module or workflow captured.

Read the submissions of one module:

```
GET /modules/{moduleId}/submissions?limit=50&offset=0
```
→ `getModuleSubmissions`

Read one submission by id:

```
GET /modules/{moduleId}/submissions/{submissionId}
```
→ `getModuleSubmission`

Search across every module and workflow in the environment:

```
GET /submissions?limit=50&offset=0
```
→ `getAllSubmissions`

Read a workflow's submissions:

```
GET /workflows/{workflowId}/submissions
```
→ `getWorkflowSubmissions`

Merge several submissions into one view:

```
GET /submissions/merge/{submissionIds}
```
→ `getMergedSubmissions`

Read the revision history of a submission:

```
GET /modules/{moduleId}/submissions/{submissionId}/revisions
```
→ `getModuleSubmissionRevisions`

## Step 3 — page correctly

Unqork pages by **offset**, not cursor. There is no `has_more` flag, no `next`
link and no total count. Page by:

1. Request with `limit` and `offset=0`.
2. If the page came back full (`len(results) == limit`), add `limit` to `offset`
   and request again.
3. Stop on the first short page.

`limit` defaults and ceilings are per operation — `getUsers`, for example,
defaults to 50 with a maximum of 500. Do not assume a global ceiling.

Sort with `sort=field` (ascending) or `sort=-field` (descending); comma-separate
for multiple keys. Submission-oriented operations use `sortBy` + `sortOrder`
instead.

## Step 4 — filter and trim the payload

```
?filter=name=Bill;role=admin
```

- Conditions are `;`-separated.
- **Every filter condition is "starts with"** — there is no exact-match, no
  contains, no comparison operator. If you need exact match, filter server-side
  then verify client-side.
- Percent-encode reserved URL characters (`*`, `'`, `(`, `)`, `:`) inside values.
- When filtering on `created` or `modified`, all timestamps are UTC.

Useful payload controls on submission reads:

| Parameter | Effect |
|---|---|
| `dataFields` | Return only the named submission fields (sparse fieldset) |
| `metadataFilter` | Filter on submission metadata |
| `includeDeleted` | Include soft-deleted submissions |
| `includeRaw` | Return raw data alongside resolved data |
| `includeBase64` | Inline file content as base64 |
| `resolveCloudStorageUrls` | Resolve file references to signed cloud-storage URLs |

Prefer `dataFields` — submission payloads can be large, and `includeBase64` on a
document-heavy module will return megabytes per record.

## Step 5 — respect the rate limit

Every response carries:

```
x-ratelimit-limit: 1000
x-ratelimit-remaining: 998
x-ratelimit-reset: 1785547409
```

`x-ratelimit-reset` is Unix epoch seconds. The limit is **per source IP, per
server, per 60 seconds**, and it is an environment setting an administrator
tunes — the platform default is 1000 but the documented minimum is 100 and Unqork
actively recommends lowering it. **Do not hard-code 1000.** Read
`x-ratelimit-remaining` and slow down as it approaches zero.

There is no `Retry-After` header, and 429 is not declared in the OpenAPI. Sleep
until `x-ratelimit-reset` and resume.

## Handling errors

Errors are a flat JSON object — **not** RFC 9457 problem+json:

```json
{"code": 401, "message": "Unauthorized"}
```

`code` just restates the HTTP status; there is no machine-readable error
identifier.

| Status | Meaning | What to do |
|---|---|---|
| 401 | Missing/expired token | Re-request a token; tokens live one hour |
| 403 | Role doesn't grant this | Attach the required role to the credential, or use the other surface (Express vs Creator) |
| 404 | Not found | Check the id; soft-deleted submissions need `includeDeleted=true` |
| 412 | Validation/execution error | Read `validationErrors[].path` — see the execute-a-module skill |
| 429 | Rate limited | Back off to `x-ratelimit-reset` |

Operation descriptions in the spec name the required role under
`### Authorization Required:` — read it before assuming a 403 is a bug.

## Cross-references

- `conventions/unqork-conventions.yml` — full request/response semantics
- `errors/unqork-problem-types.yml` — the error catalogue
- `rate-limits/unqork-rate-limits.yml` — limits and headers
- `authentication/unqork-authentication.yml` — the auth profile
