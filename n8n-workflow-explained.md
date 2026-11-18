# GitLab – n8n – OpenProject Integration
## Complete Node-by-Node Workflow Explanation

---

## Overview

This workflow automates the synchronization between **GitLab** (where code is written) and **OpenProject** (where tasks are tracked). It runs entirely inside **n8n** and eliminates the need for developers to manually update work package statuses.

The workflow is split into two main flows:

- **Flow A — GitLab → OpenProject**: Triggered when developers push code or open/update merge requests in GitLab
- **Flow B — OpenProject → OpenProject (Bidirectional Sync)**: Triggered when a work package is updated in OpenProject, automatically creating SQA Test Items

---

## Flow A — GitLab → OpenProject

### 1. `[UserSettings] Gitlab Webhooks`
**Type: Webhook**

The starting point of the entire workflow. This node listens for incoming HTTP requests from GitLab.

When a developer does anything in GitLab — pushes code, opens a merge request, or leaves a comment — GitLab sends a **webhook** (an automatic notification) to the URL registered in this node. That notification contains all the event data as a JSON payload.

This node receives that payload and passes it to the next node to begin processing.

> **What needs to be configured:** The webhook URL path must be updated to match your project name (e.g., `integration-yourproject`). This URL is then registered in your GitLab repository under Settings → Webhooks.

---

### 2. `Extract Gitlab Data`
**Type: Code**

Reads the raw GitLab payload and extracts only the fields that the rest of the workflow needs.

Different GitLab events have different payload structures, so this node handles each case separately:

- **Push Hook** — extracts branch name, commit messages, author info, SHA hashes
- **Merge Request Hook** — extracts MR title, source/target branch, state, action (opened/merged/closed), draft status, assignees, reviewers, dates
- **Note Hook** — extracts the comment text and the MR it belongs to
- **Job Hook** — extracts CI/CD build status and runner info

Most importantly, this node runs the **OPID extraction** function across the branch name, MR title, and commit messages to find the OpenProject Work Package ID (e.g., `OP-141973` → `141973`). This ID is the critical link between GitLab and OpenProject.

The output of this node is a clean, structured JSON object that all downstream nodes read from.

---

### 3. `Validate Gitlab OPID`
**Type: Code – IF**

A decision point. Checks whether the `opId` field extracted in the previous node has a value.

- **True path** → OPID found, continue workflow
- **False path** → OPID missing, redirect to error handling

If a developer pushed code on a branch with no OPID in the name (e.g., `fix-login-bug` instead of `feature/OP-141973-login-fix`), this node catches it and stops the workflow before it tries to call OpenProject with no ID.

---

### 4. `[UserSettings] Get OP Work Package`
**Type: HTTPS – GET**

Uses the OPID to fetch the corresponding work package from OpenProject via its API.

The API call looks like:
```
GET /api/v3/work_packages/{opId}
```

The response contains the work package's current status, type (Task / User Story / Issue), project name, and other metadata. This data is needed by downstream nodes to determine what status change to apply.

> **What needs to be configured:** Basic Auth credentials using `apikey` as the username and your OpenProject API token as the password.

---

### 5. `[User Settings] GET OP List Statuses`
**Type: HTTPS – GET**

Fetches the full list of available statuses configured in OpenProject (e.g., New, In Progress, Developed, Done, etc.).

This is needed because OpenProject status transitions require a **status ID** (a number), not just the status name. This node retrieves all available IDs so the transition logic later can look up the correct one.

---

### 6. `[User settings] Validate Project & OP Status`
**Type: Code**

Performs two jobs in one node:

**Validation:** Checks whether the work package belongs to a project that is allowed to be auto-updated by this workflow. This is an allowlist — if your project name isn't registered here, the workflow stops and sends an error notification instead of modifying random work packages.

**Extraction:** Pulls out the relevant status IDs from the list retrieved in node 5, mapping status names to their numeric IDs for use in the transition logic later.

> **What needs to be configured:** Add your OpenProject project name to the `allowedOpenProject` constant inside this node.

---

### 7. `Filter Gitlab Event Type`
**Type: Code – Switch Case**

Routes the workflow into different branches depending on what type of GitLab event arrived:

| Event Type | Next path |
|---|---|
| Merge Request Hook | MR processing branch |
| Push Hook | Push processing branch |
| Note Hook | Note processing branch |
| Anything else | Error handling |

This is the main fork in the workflow. Each event type follows its own logic from here.

---

### 8. `Determine Push Event`
**Type: Code**
*(Push branch only)*

Classifies the push event into one of three sub-types:

- **Branch Creation** — a brand new branch was pushed for the first time
- **Branch Deletion** — a branch was deleted
- **Normal Push** — regular commit update on an existing branch

Only **Branch Creation** triggers a status update in OpenProject. The other two types are passed to the next node to be filtered out.

---

### 9. `Validate Normal Push`
**Type: Code – IF**
*(Push branch only)*

Filters out normal pushes and branch deletions. Only branch creation events pass through to the status transition step.

- **True (branch creation)** → continue to status transition
- **False (normal push or deletion)** → stop, no update needed

---

### 10. `Extract Merge Request Context`
**Type: Code**
*(MR branch only)*

Reads the MR state and action fields and classifies the event into one of seven states:

| Flag | Meaning |
|---|---|
| `ismrNew` | MR was just created |
| `ismrDraft` | MR is in draft — not ready for review |
| `ismrOpened` | MR is open and ready for review |
| `ismrApproved` | MR has been approved by reviewers |
| `ismrMerged` | MR has been merged into the target branch |
| `ismrClosed` | MR was closed without merging |
| `ismrReopened` | A previously closed MR was reopened |

These flags are used by the transition logic nodes to determine which OpenProject status to move to.

---

### 11. `Filter WorkPackages Type`
**Type: Code – Switch Case**

Routes the workflow based on the **type** of the work package fetched from OpenProject:

| Work Package Type | Next path |
|---|---|
| Task | Handle Task Transition |
| User Story | Handle User Story Transition |
| Issue | Handle Issue Transition |

Each type has its own status transition rules, so they need separate handling.

---

### 12. `Handle Task Transition`
**Type: Code**

Calculates the correct next status for a **Task** work package based on the GitLab event context (which MR state or push type occurred).

---

### 13. `Handle User Story Transition`
**Type: Code**

Same as above but applies the transition rules specific to **User Story** work packages.

---

### 14. `Handle Issue Transition`
**Type: Code**

Same as above but applies the transition rules specific to **Issue** work packages.

---

### 15. `Build Work Package Patch Payload`
**Type: Code**

Constructs the JSON payload required to update the work package status in OpenProject.

The payload follows the OpenProject API format:
```json
{
  "status": {
    "href": "/api/v3/statuses/{newStatusId}"
  }
}
```

---

### 16. `Validate Changed Status`
**Type: Code – IF**

Before sending the update, checks whether the target status is actually different from the current status.

- **True (status will change)** → send the PATCH request
- **False (already at target status)** → skip the update

This prevents unnecessary API calls and avoids writing redundant activity log entries in OpenProject.

---

### 17. `[User Settings] Update OP Work Package`
**Type: HTTPS – PATCH**

Sends the PATCH request to OpenProject to apply the new status.

```
PATCH /api/v3/work_packages/{opId}
Body: { "status": { "href": "/api/v3/statuses/{newStatusId}" } }
```

---

### 18. `Build Work MR Comment`
**Type: Code**

Generates a human-readable comment describing what just happened. For example:

> *"Status updated to 'In Progress' triggered by MR #45 opened on branch feature/OP-141973-adding-information."*

This comment is posted to the work package so the team can see why the status changed.

---

### 19. `[User Settings] Post OP Comment`
**Type: HTTPS – POST**

Posts the comment generated in node 18 to the OpenProject work package activity stream.

```
POST /api/v3/work_packages/{opId}/activities
```

---

### 20. `[User Settings] Prepare Validation Error Message`
**Type: Code**

Builds the error message payload when something goes wrong — missing OPID, project not in allowlist, API failure, etc. Formats the message for Google Chat's webhook format.

---

### 21. `[User Settings] Send Validation Error Notification`
**Type: HTTPS – POST**

Sends the error message to the team's Google Chat space via a registered webhook URL.

> **What needs to be configured:** Register your Google Chat webhook URL inside this node.

---

### 22. `Validate Changed Description Gitlab`
**Type: Code – IF**

Checks the GitLab MR description to see if an OpenProject tracking link has already been added. Avoids adding it again if it's already there.

- **True (link already present)** → skip, go to No Operation
- **False (link not present)** → build and send the PATCH to add it

---

### 23. `Build Gitlab Patch Payload`
**Type: Code**

Constructs a PATCH payload to update the GitLab MR description, appending the OpenProject work package link while preserving any existing description text.

---

### 24. `No Operation, do nothing`
**Type: Built-in**

The termination point for branches that reach a valid end state but require no further action (e.g., status was already correct, or description already had the OP link).

---

---

## Flow B — OpenProject → OpenProject (Bidirectional Sync)

This is a separate flow triggered by OpenProject itself, not GitLab.

---

### 1. `[UserSettings] Open Projects Webhooks`
**Type: Webhook**

The entry point for the bidirectional sync. OpenProject sends a webhook here whenever a work package is updated (status changed, fields modified).

> **What needs to be configured:** An OpenProject administrator must register this webhook URL under Settings → API and Webhooks in OpenProject, scoped to your specific project.

---

### 2. `[User settings] Validate OP Fields & Extract OP`
**Type: Code**

Validates that the incoming OpenProject update is for a project that is allowed to trigger this flow. Also extracts the relevant data (work package ID, type, status) needed by downstream nodes.

Distinguishes between automated Test items and other work package types to decide how to route the workflow.

> **What needs to be configured:** Register your project name in the `allowedOpenProject` constant.

---

### 3. `Validate Test`
**Type: Code – IF**

Checks whether the updated work package is of type **Test**. Only Test items continue through this flow.

- **True (is Test type)** → continue
- **False (other type)** → stop

---

### 4. `[UserSettings] Get OP Parent Work Package`
**Type: HTTPS – GET**

Fetches the **parent** work package of the Test item — typically a User Story or Feature. This parent data is used as reference for creating or updating related Test items.

---

### 5. `Handle Test Transition`
**Type: Code**

Calculates the next appropriate status for the Test item based on its current status and the incoming webhook context.

---

### 6. `Validate Finished Item`
**Type: Code – IF**

Checks whether the Test item's status has reached a completed state (e.g., Solved, Done, Developed).

- **True (finished)** → proceed to check if a new Test item should be created
- **False (not finished)** → stop

---

### 7. `[User Settings] Check Test Items`
**Type: HTTPS – GET**

Queries OpenProject to see if Test items already exist for this work package — specifically looking for items that were previously created and marked as New or Failed.

---

### 8. `Validate to Create Test Items`
**Type: Code – IF**

Based on the result from node 7, decides whether to create a new Test item.

- **True (no existing item, or existing is completed)** → create new Test item
- **False (item already exists and is active)** → skip creation

---

### 9. `Build Test Items Patch Payload`
**Type: Code**

Constructs the JSON payload required to create a new Test item in OpenProject, including required schema fields and metadata linking it to the parent work package.

---

### 10. `[User Settings] Create Items Test`
**Type: HTTPS – POST**

Submits the payload to OpenProject to create the new Test item under the parent work package.

```
POST /api/v3/projects/{projectId}/work_packages
```

---

## Node Quick Reference

| # | Node Name | Type | Flow | Purpose |
|---|---|---|---|---|
| 1 | `[UserSettings] Gitlab Webhooks` | Webhook | A | Receives all GitLab events |
| 2 | `Extract Gitlab Data` | Code | A | Parses payload, extracts OPID |
| 3 | `Validate Gitlab OPID` | IF | A | Stops flow if OPID is missing |
| 4 | `[UserSettings] Get OP Work Package` | HTTPS GET | A | Fetches work package from OP |
| 5 | `[User Settings] GET OP List Statuses` | HTTPS GET | A | Fetches available status IDs |
| 6 | `[User settings] Validate Project & OP Status` | Code | A | Validates project allowlist |
| 7 | `Filter Gitlab Event Type` | Switch | A | Routes by event type |
| 8 | `Determine Push Event` | Code | A | Classifies push sub-type |
| 9 | `Validate Normal Push` | IF | A | Filters out non-creation pushes |
| 10 | `Extract Merge Request Context` | Code | A | Classifies MR state |
| 11 | `Filter WorkPackages Type` | Switch | A | Routes by work package type |
| 12 | `Handle Task Transition` | Code | A | Calculates Task status change |
| 13 | `Handle User Story Transition` | Code | A | Calculates User Story status change |
| 14 | `Handle Issue Transition` | Code | A | Calculates Issue status change |
| 15 | `Build Work Package Patch Payload` | Code | A | Builds status update payload |
| 16 | `Validate Changed Status` | IF | A | Skips if status unchanged |
| 17 | `[User Settings] Update OP Work Package` | HTTPS PATCH | A | Applies status update in OP |
| 18 | `Build Work MR Comment` | Code | A | Generates activity comment |
| 19 | `[User Settings] Post OP Comment` | HTTPS POST | A | Posts comment to OP |
| 20 | `[User Settings] Prepare Validation Error Message` | Code | A | Builds error message |
| 21 | `[User Settings] Send Validation Error Notification` | HTTPS POST | A | Sends error to Google Chat |
| 22 | `Validate Changed Description Gitlab` | IF | A | Checks if OP link already in MR |
| 23 | `Build Gitlab Patch Payload` | Code | A | Builds MR description update |
| 24 | `No Operation, do nothing` | Built-in | A | Terminal node for no-op paths |
| 25 | `[UserSettings] Open Projects Webhooks` | Webhook | B | Receives OP update events |
| 26 | `[User settings] Validate OP Fields & Extract OP` | Code | B | Validates project + extracts data |
| 27 | `Validate Test` | IF | B | Filters to Test type only |
| 28 | `[UserSettings] Get OP Parent Work Package` | HTTPS GET | B | Fetches parent work package |
| 29 | `Handle Test Transition` | Code | B | Calculates Test status change |
| 30 | `Validate Finished Item` | IF | B | Checks if item is complete |
| 31 | `[User Settings] Check Test Items` | HTTPS GET | B | Checks for existing Test items |
| 32 | `Validate to Create Test Items` | IF | B | Decides whether to create |
| 33 | `Build Test Items Patch Payload` | Code | B | Builds new Test item payload |
| 34 | `[User Settings] Create Items Test` | HTTPS POST | B | Creates Test item in OP |
