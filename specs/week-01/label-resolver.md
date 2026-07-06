# Label Resolver

## Overview

Sub-workflow that resolves a Gmail label name to its Gmail label ID, creating the label if it doesn't exist. Implements **parent-first creation**: when a hierarchical label like `Promotional/Marketing` needs creation, it first creates the parent (`Promotional`) then creates the child, ensuring proper Gmail `/` nesting.

## Trigger

| Property | Value |
|----------|-------|
| Type | Execute Workflow Trigger |
| Auth | Workspace-level n8n API key |

## Input

| Field | Type | Description |
|-------|------|-------------|
| `labelName` | string | Full label name, may include `/` for hierarchy (e.g. `Finance/Invoices`) |
| `fallbackLabelId` | string | Gmail label ID to use if creation fails (e.g. `Label_9` for "Needs Review"), or `NEW_LABEL_` prefix to force creation |
| `mainLabelName` | string | Parent label name (e.g. `Finance`) — used for parent-first creation |

## Output

| Field | Type | Description |
|-------|------|-------------|
| `messageId` | string | Passed through from trigger input |
| `labelId` | string | Resolved Gmail label ID (existing or newly created) |
| `labelName` | string | Resolved label name |

## Nodes (12)

### 1. Execute Workflow Trigger
- **Type:** `n8n-nodes-base.executeWorkflowTrigger`
- **Purpose:** Receives `{labelName, fallbackLabelId, mainLabelName}` from parent workflow

### 2. Use Fallback? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `$json.fallbackLabelId` is truthy AND does not start with `NEW_LABEL_` (i.e. a real Gmail label ID was passed directly)
- **TRUE:** → Format Result (early return with the fallback label ID)
- **FALSE:** → Get All Labels (normal resolution path)
- **Purpose:** Short-circuits when caller already knows the label ID (error fallback case)

### 3. Get All Labels
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `getAll` (label)
- **Return All:** `true`
- **Purpose:** Fetches all existing Gmail labels to check existence

### 4. Resolve Main (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Logic:**
  1. Reads `{labelName, mainLabelName}` from trigger input (`$("Execute Workflow Trigger").first().json`)
  2. If `mainLabelName` is empty or equals `labelName` → searches for `labelName` directly (single-level label)
  3. Otherwise → searches for `mainLabelName` in existing Gmail labels
  4. Sets `mainLabelId` to found label ID or `null`, `mainExists` to boolean
- **Output:** `{labelName, mainLabelName, mainLabelId, mainExists}`
- **Purpose:** Explicitly resolves the parent/main label only

### 5. Main Exists? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `$json.mainExists === false`
- **TRUE:** → Create Main (parent does not exist)
- **FALSE:** → Pipeline Main (parent already exists)
- **Purpose:** Routes parent creation separately

### 6. Create Main (Gmail)
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `create` (label)
- **Name:** `{{ $json.mainLabelName }}`
- **Visibility:** `labelListVisibility: "labelShow"`, `messageListVisibility: "show"`
- **Purpose:** Creates the parent label in Gmail (e.g. `Finance`)

### 7. Pipeline Main (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Logic:** Handles both incoming paths:
  - From Create Main: reads `$json.id` = new label ID, recovers original fields from Resolve Main via `$("Resolve Main").first().json`
  - From Main Exists? FALSE: passes through existing `mainLabelId`
- **Output:** `{mainLabelId, labelName, mainLabelName}`
- **Purpose:** Normalizes parent label ID regardless of source

### 8. Is Sub? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `$json.labelName === $json.mainLabelName`
- **TRUE:** → Format Result (single-level label, no sub-label to resolve)
- **FALSE:** → Resolve Sub (label has a `/` hierarchy)
- **Purpose:** Decides whether sub-label resolution is needed

### 9. Resolve Sub (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Logic:** Searches `$("Get All Labels").all()` for the full `labelName` (e.g. `Finance/Invoices`)
- **Output:** `{labelName, mainLabelName, parentId, labelId, subExists}`
- **Purpose:** Explicitly checks if the hierarchical child label exists

### 10. Sub Exists? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `$json.subExists === false`
- **TRUE:** → Create Sub (child does not exist)
- **FALSE:** → Format Result (child already exists)

### 11. Create Sub (Gmail)
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `create` (label)
- **Name:** `{{ $json.labelName }}` (e.g. `Finance/Invoices`)
- **Visibility:** `labelListVisibility: "labelShow"`, `messageListVisibility: "show"`
- **Purpose:** Creates the sub-label. Since parent already exists (created in step 6 or pre-existing), Gmail automatically nests it under the parent

### 12. Format Result (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Logic:** Handles 4 incoming paths: Use Fallback? (TRUE), Is Sub? (TRUE), Sub Exists? (TRUE), Create Sub. Resolves `labelId = $json.labelId || $json.mainLabelId || $json.id || $json.fallbackLabelId`
- **Output:** `{messageId, labelId, labelName}`
- **Purpose:** Normalizes output format regardless of resolution path

## Data Flow

1. **Execute Workflow Trigger** receives `{labelName, fallbackLabelId, mainLabelName}`
2. **Use Fallback?** — if fallbackLabelId is a real label ID, return immediately
3. **Get All Labels** fetches existing Gmail labels
4. **Resolve Main** reads `labelName`/`mainLabelName` and checks parent existence — sets `mainLabelId` and `mainExists`
5. **Main Exists?** — if parent doesn't exist, routes to Create Main
6. **Create Main** creates the parent label (e.g. `Finance`)
7. **Pipeline Main** normalizes parent label ID from both paths
8. **Is Sub?** — if `labelName === mainLabelName` (single label), skip to Format Result
9. **Resolve Sub** checks child existence — sets `labelId` and `subExists`
10. **Sub Exists?** — if child doesn't exist, routes to Create Sub
11. **Create Sub** creates the child label (e.g. `Finance/Invoices`)
12. **Format Result** normalizes to `{messageId, labelId, labelName}`

## Dependencies

- **Credentials:** `gmailOAuth2` — Primary Email Account [neo7moruri]
- **Called by:** Gmail Manager (`Io4HrrgLoqdMVg2B`), Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)

## Related

- Runbook: `../../runbooks/week-01/label-resolver.md`
- Spec: `../week-01/gmail-manager.md`
