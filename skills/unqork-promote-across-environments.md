---
name: Promote Unqork configuration across environments
description: >-
  Move modules, application items, reference data, styles, groups and roles from
  one Unqork environment to the next (Development → Staging → UAT → Production)
  using the Promotions API, with dependency checks first. The highest-consequence
  surface in the Unqork API.
api: openapi/unqork-customer-api-openapi.yml
generated: '2026-07-31'
method: generated
source: >-
  Grounded in openapi/unqork-customer-api-openapi.yml,
  lifecycle/unqork-lifecycle.yml and sandbox/unqork-sandbox.yml
operations:
  - getPromotionHosts
  - applicationsDependencies
  - applicationsDataModelsDependencies
  - applicationModules
  - applicationsUniqueName
  - promoteModule
  - promoteApplicationByItems
  - promoteDataCollection
  - promoteStyle
  - promoteGroups
  - promoteRoles
  - promoteGlobalVariables
  - listAuditLogs
---

# Promote Unqork configuration across environments

Assumes a **Creator** bearer token — see
`unqork-authenticate-and-read-submissions.md`. Express credentials cannot promote.

## Why this skill is different

Unqork has no test mode and no key prefixes. Test-versus-live separation is done
with **whole environments**, each on its own subdomain
(`https://{subdomain}.unqork.io/api/1.0`), and configuration crosses the boundary
only through Promotions.

That makes this the most consequential surface in the API. A promotion writes into
UAT or Production, and there is no promotion rollback operation. **Every operation
in this skill should require human approval before an agent executes it.**

## Step 1 — discover the targets

Never hard-code a target host.

```
GET /promote/hosts
```
→ `getPromotionHosts`

Returns the `PromotionHosts` the current environment is permitted to promote to.
Confirm the target is the one you intend before going further — the difference
between UAT and Production is one string.

## Step 2 — resolve dependencies BEFORE promoting

This is the step that gets skipped and then breaks Production. An Unqork
application is a graph — modules reference other modules, data models, reference
data, styles and global variables. Promoting a module without its dependencies
ships a broken application.

```
GET /applications/{applicationId}/dependencies
GET /applications/{applicationId}/dataModels/dependencies
GET /applications/{applicationId}/modules
```
→ `applicationsDependencies`, `applicationsDataModelsDependencies`,
`applicationModules`

Check whether a resource is actually part of the application you think it is:

```
GET /applications/{applicationId}/{appType}/{resourceId}/isInApplication
```
→ `isInApplication`

Check a name is free on the target before creating anything there:

```
GET /applications/unique
```
→ `applicationsUniqueName`

## Step 3 — promote, narrowest scope first

Promote a single module:

```
POST /promote/module
```
→ `promoteModule` (body: `PromotionModuleExecuteRequest`)

Promote selected items of an application:

```
POST /promote/applicationByItems
```
→ `promoteApplicationByItems` (body: `PromotionApplicationByItemExecuteRequest`)

Promote reference data / data collections:

```
POST /promote/referenceData
```
→ `promoteDataCollection` (body: `PromotionReferenceDataExecuteRequest`)

Promote styling:

```
POST /promote/style
```
→ `promoteStyle` (body: `PromotionStyleExecuteRequest`)

Promote groups and roles — the access-control layer:

```
POST /promote/groups
POST /promote/roles
```
→ `promoteGroups`, `promoteRoles`
(bodies: `PromotionGroupExecuteRequest`, `PromotionRoleExecuteRequest`)

Recommended order for a full move: **roles → groups → reference data → modules →
application items → style.** Access control and data the configuration depends on
should exist on the target before the configuration that references them lands.

## Step 4 — global variables travel separately

```
GET /globalvars
```
→ `promoteGlobalVariables`

Despite the operationId, this is a **GET** that lists global variables — it reads
the environment-scoped configuration values that applications reference. Read them
on both source and target and reconcile before promoting anything that depends on
them. A module promoted into an environment whose global variables differ will run
against the wrong endpoints or credentials.

Manage them individually with `getGlobalVariable`, `createGlobalVariable`,
`updateGlobalVariable`, `deleteGlobalVariable`.

## Step 5 — verify on the target

Promotion is not transactional across resources. After promoting:

1. Re-point at the **target** environment's base URL and get a token there —
   credentials do not cross environments.
2. `applicationModules` / `applicationsDependencies` on the target to confirm the
   graph resolved.
3. `listAuditLogs` (`GET /logs/audit-logs`) on the target to confirm what actually
   landed and when.

## Guardrails

- **No rollback.** There is no `unpromote`. The recovery path is promoting a
  known-good version forward, which means you need the source environment to still
  hold it.
- **No idempotency.** A retried promotion re-executes. Verify with
  `listAuditLogs` before repeating a call whose outcome you are unsure of.
- **Credentials are per environment.** A token minted against Staging is
  meaningless against Production. Expect to hold several.
- **Rate limits are per environment too.** Production is often configured lower
  than the 1000/min default — read `x-ratelimit-remaining` on the target.
- **Time it against the release calendar.** Unqork ships platform GA releases
  quarterly through the same Staging → UAT → Production ladder, with weekly
  Tuesday patches. Promoting into an environment mid-platform-upgrade is asking
  for a confusing failure. See `changelog/unqork-changelog.yml`.

## Cross-references

- `lifecycle/unqork-lifecycle.yml` — release cadence and the environment ladder
- `sandbox/unqork-sandbox.yml` — how test/live separation actually works
- `data-model/unqork-data-model.yml` — what depends on what
