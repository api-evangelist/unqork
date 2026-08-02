---
name: Start, resume and schedule an Unqork workflow
description: >-
  Drive Unqork's workflow surface end to end — start a workflow submission, resume
  a paused one, hand a module submission into a workflow, and control the
  timer-start nodes that fire workflows on a schedule.
api: openapi/unqork-customer-api-openapi.yml
generated: '2026-07-31'
method: generated
source: >-
  Grounded in openapi/unqork-customer-api-openapi.yml and
  errors/unqork-problem-types.yml
operations:
  - createWorkflowSubmission
  - createWorkflowSubmissionFromStep
  - resumeWorkflow
  - handoffSubmission
  - getWorkflowSubmissions
  - getWorkflowSubmission
  - updateWorkflowSubmission
  - getWorkflowSubmissionRevisions
  - listTimerStartNodes
  - listTimedEventsBySubmissionId
  - startTimerStartNode
  - stopTimerStartNode
  - runTimerStartNodeOnce
  - restoreDeletedWorkflow
---

# Start, resume and schedule an Unqork workflow

Assumes a bearer token — see `unqork-authenticate-and-read-submissions.md`.

## Identifiers: `workflowPath` vs `workflowId`

Unqork's workflow surface uses **two different identifiers**, and mixing them up
is the most common failure here:

- **`workflowPath`** — used by the *execution* operations under
  `/workflow-execute/` and `/workflow/`, and by every timer operation.
- **`workflowId`** — used by the *data* operations under `/workflows/`
  (submissions, revisions, restore).

Read the path segment carefully before substituting.

## Step 1 — start a workflow

```
POST /workflow-execute/{workflowPath}
Content-Type: application/json

{ "data": { ... } }
```
→ `createWorkflowSubmission`

Returns a `WorkflowCreateSubmissionResponse`, which wraps a
`SavedSubmissionResponse` — the workflow submission id you will need for every
later call.

To enter partway through the flow rather than at the beginning:

```
POST /workflow-execute/{workflowPath}/{stepPath}
```
→ `createWorkflowSubmissionFromStep`

Like every Unqork write, **this is not idempotent**. A retried start creates a
second workflow instance. Correlate on submission metadata and read back with
`getWorkflowSubmissions` before retrying.

## Step 2 — hand an existing module submission into a workflow

If the data was captured by a plain module and now needs to enter an
orchestration:

```
PUT /workflow/{workflowPath}/handoff
```
→ `handoffSubmission`

On failure this returns `FailedHandoffSubmissionToWorkflowResponse`, which adds
`submissionId` and `statusCode` to the standard validation envelope — so you can
tell *which* submission failed to hand off, not just that something did.

## Step 3 — resume a paused workflow

Workflows pause at named resume points — a human approval, an external callback,
a wait state:

```
POST /workflow-execute/{workflowPath}/resume/{resumePathName}/submission/{submissionId}
```
→ `resumeWorkflow`

On failure: `FailedResumeWorkflowResponse`, which carries `submissionId`, the
current `state`, `statusCode` and the validation envelope. **Read `state` before
retrying** — it tells you whether the workflow actually advanced. A resume that
returns an error but a changed `state` has partially progressed, and blindly
resuming again can double-advance it.

## Step 4 — inspect workflow submissions

```
GET /workflows/{workflowId}/submissions
GET /workflows/{workflowId}/submissions/{submissionId}
PUT /workflows/{workflowId}/submissions/{submissionId}
GET /workflows/{workflowId}/submissions/{submissionId}/revisions
```
→ `getWorkflowSubmissions`, `getWorkflowSubmission`, `updateWorkflowSubmission`,
`getWorkflowSubmissionRevisions`

Revisions are the audit trail for a workflow instance — use them to reconstruct
how a submission reached its current state before deciding to intervene.

## Step 5 — control scheduled triggers

Unqork's **timer-start nodes** are the platform's scheduled triggers. They are the
closest thing Unqork has to an event producer, and the Customer API can drive
them.

List them and their statuses:

```
GET /workflows/{workflowPath}/timerstart
```
→ `listTimerStartNodes`

Turn one on or off:

```
POST /workflows/{workflowPath}/timerstart/{timerStartNodePath}/start
POST /workflows/{workflowPath}/timerstart/{timerStartNodePath}/stop
```
→ `startTimerStartNode`, `stopTimerStartNode`

Fire one immediately as a test, without waiting for its schedule:

```
POST /workflows/{workflowPath}/timerstart/{timerStartNodePath}/run-once
```
→ `runTimerStartNodeOnce`

This is Unqork's substitute for a test clock. Two guardrails:

- A run-once against a node that is already running returns **400** with
  `"TimerStart run once is already running"`. Call `listTimerStartNodes` first.
- A bad `timerStartNodePath` returns **404** `"Timer start node not found"`.

Inspect what is queued for a specific submission:

```
GET /workflows/{workflowPath}/timedEvents/{submissionId}
```
→ `listTimedEventsBySubmissionId`

## Step 6 — recover a deleted workflow

```
POST /workflows/{workflowId}/restore
POST /workflows/{workflowId}/submissions/{submissionId}/restore
```
→ `restoreDeletedWorkflow`, `restoreDeletedWorkflowSubmission`

As with modules, deletes are soft by default and `destroy=true` is permanent.

## Consequence ranking

`startTimerStartNode` and `stopTimerStartNode` change the behaviour of a
**production schedule** — stopping a timer silently suspends business processing
with no error anywhere. Treat both as human-approval operations, not autonomous
agent actions. `runTimerStartNodeOnce` executes real workflow logic with real
side effects; it is a *test run* of the trigger, not a dry run of the workflow.

## Cross-references

- `errors/unqork-problem-types.yml` — the 412 / failed-handoff / failed-resume envelopes
- `data-model/unqork-data-model.yml` — how workflows, submissions and timed events relate
- `agentic-access/unqork-agentic-access.yml` — per-operation execution contracts
