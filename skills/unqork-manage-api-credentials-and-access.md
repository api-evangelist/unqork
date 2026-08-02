---
name: Manage Unqork API credentials, users and access
description: >-
  Administer the Unqork access-control surface through the Customer API — mint,
  rotate and revoke API credentials, manage Express users and groups, and read the
  audit trail. Every operation here is privileged; treat all of them as
  human-approval actions.
api: openapi/unqork-customer-api-openapi.yml
generated: '2026-07-31'
method: generated
source: >-
  Grounded in openapi/unqork-customer-api-openapi.yml and
  https://docs.unqork.io/docs/api-access-management
operations:
  - credentialsGetAll
  - credentialsCreate
  - credentialsUpdate
  - credentialsRevoke
  - credentialsDeleteById
  - getUsers
  - getUser
  - createUser
  - updateUser
  - deleteUser
  - getUserPasswordStatus
  - getGroups
  - getGroup
  - createGroup
  - updateGroup
  - deleteGroup
  - listAuditLogs
  - generateReferString
---

# Manage Unqork API credentials, users and access

Requires a **Designer Administrator** role — most operation descriptions in the
spec state this explicitly under `### Authorization Required:`.

## The access model in one paragraph

Unqork has **no OAuth scopes**. The API's OAuth2 scheme declares a single
placeholder scope (`none: N/A`), and what a token can do is decided entirely by
the **Express or Creator roles** bound to the credential that minted it. Least
privilege is therefore a *provisioning-time* decision: you cannot request a
narrower token — you must create a narrower credential.

Two credential surfaces exist and they are not interchangeable:

| Surface | Bound to | Governs |
|---|---|---|
| **Express** | Express Role(s) | End-user-facing data — submissions, workflows |
| **Creator** | Creator Role(s) | Design-time resources — modules, applications, promotions |

At least one role is **required** to create a credential of either kind.

## Step 1 — inventory what exists

```
GET /credentials
```
→ `credentialsGetAll`

Each credential reports a status:

| Status | Meaning |
|---|---|
| `ACTIVE` | Usable |
| `EXPIRES SOON` | Under 15% of the validity window remains (14 days on a 90-day credential) |
| `EXPIRED` | Past expiry; cannot access the API |
| `REVOKED` | Access removed by an administrator (Creator credentials only) |
| `LOCKED` | Temporarily disabled after up to five failed client-secret attempts; auto-reverts to ACTIVE after 30 minutes |

`EXPIRES SOON` is the one to alert on — it is Unqork's built-in rotation warning
and it is the difference between a planned rotation and a 401 in production.

## Step 2 — create a credential

```
POST /credentials
Content-Type: application/json
```
→ `credentialsCreate` (body: `CredentialRequest`, returns `CredentialResponse`)

Expiration is set in **days**: minimum 1, maximum 730 (2 years). The
administration UI default is 90 days.

**The Client Secret is shown exactly once.** It is returned on creation and is not
retrievable afterwards — if it is lost, the only recovery is generating a new
secret. Capture it into a secret store in the same operation that creates it; do
not log it, do not put it in a transcript, and never write it into a repository.

Client IDs carry a `uq` prefix (e.g. `uq6478a3a52300ff7cf8ac55fc`). Note that the
prefix is the same in every environment, so a credential string alone does not
tell you whether it points at Production.

## Step 3 — rotate

Rotation is generating a new secret for the existing credential; the Client ID is
retained. Sequence it as:

1. Generate the new secret.
2. Deploy it to every consumer.
3. Confirm consumers are authenticating with it.
4. Only then revoke or delete the old credential.

Update the credential's metadata (name, description, role) with:

```
PUT /credentials/{clientId}
```
→ `credentialsUpdate`

## Step 4 — revoke and delete

```
PUT /credentials/{clientId}/revoke
```
→ `credentialsRevoke` — removes API access. **Creator credentials only.**

```
DELETE /credentials/{clientId}
```
→ `credentialsDeleteById` — **permanent. Deleted credentials cannot be restored.**

Prefer revoke over delete during incident response: revoke stops access
immediately while leaving the record in place for the investigation.

## Step 5 — users and groups

```
GET /users?limit=50&offset=0&filter=role=admin
GET /users/{userId}
POST /users
PUT /users/{userId}
DELETE /users/{userId}
GET /users/{userId}/passwordStatus
```
→ `getUsers`, `getUser`, `createUser`, `updateUser`, `deleteUser`,
`getUserPasswordStatus`

`getUsers` filters on `role`, `name`, `phone`, `email`, `userId`, `groups` and
`applicationRoles` — all "starts with" matching. `showServiceUsers` controls
whether service accounts appear; by default they do not, so a user audit that
omits it will miss every machine identity.

Two email side effects to be deliberate about on user writes:

- `shouldNotify=true` on `createUser` emails the user their temporary password.
- `shouldNotify` on `updateUser` controls the **New Account (Token Invitation)**
  email: `true` sends both the "New Account" and "User Changed" emails; `false`
  suppresses only the "New Account" email — the "User Changed" email still goes
  out either way.
- `skipTemporaryPassword=true` sets the password permanently and immediately; a
  password must be supplied in the body, and it **cannot be combined with**
  `shouldNotify`.

Groups are addressed by **name**, not id:

```
GET /groups
GET /groups/{groupName}
POST /groups
PUT /groups/{groupName}
DELETE /groups/{groupName}
```
→ `getGroups`, `getGroup`, `createGroup`, `updateGroup`, `deleteGroup`

Renaming a group changes its identifier — every reference to the old name breaks.

## Step 6 — audit

```
GET /logs/audit-logs
```
→ `listAuditLogs`

The environment audit trail, and the only after-the-fact verification the API
offers. There is no request-id header on responses, so audit logs plus the Logs
Dashboard Tool are how you reconstruct what happened.

## `generateReferString`

```
POST /referstring
```
→ `generateReferString`

Generates an encrypted referstring for authentication. This mints authentication
material — hold it to the same handling standard as a client secret.

## Agent guidance

Every operation in this skill is a privilege-management action. In
`mcp/unqork-mcp.yml` the credential, user, group and promotion operations are
deliberately **excluded** from the candidate agent tool surface: an agent that can
mint a credential can escalate its own privileges, and an agent that can revoke one
can cause an outage. Keep these behind human approval, log every invocation, and
never let a secret returned by `credentialsCreate` enter a model context.

## Cross-references

- `authentication/unqork-authentication.yml` — the auth profile
- `scopes/unqork-scopes.yml` — why there are no scopes
- `lifecycle/unqork-lifecycle.yml` — credential lifecycle timings
- `security/unqork-vulnerability-disclosure.yml` — where to report a finding
