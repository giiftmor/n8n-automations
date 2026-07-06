# Record Active Labels

## Overview

Loops through every Gmail label, fetches full details including `messagesTotal`, and persists user labels with `messagesTotal > 0` to the **Active Labels** data table. This workflow populates the reference table that `delete-orphan-labels` uses to decide what to delete.

## Trigger

| Property | Value |
|----------|-------|
| Type | Manual |
| Auth | Workspace-level n8n API key |

## Nodes

### 1. Manual Trigger
- **Type:** `n8n-nodes-base.manualTrigger`
- **Purpose:** Starts execution on demand

### 2. Get All Labels
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `getAll` (label)
- **Return All:** `true`
- **Purpose:** Returns all Gmail labels (id, name, type only — no `messagesTotal`)

### 3. Loop Over Items (SplitInBatches)
- **Type:** `n8n-nodes-base.splitInBatches`
- **Batch Size:** 1
- **Purpose:** Iterates each label individually so we can call the `get` endpoint per label

### 4. Wait
- **Type:** `n8n-nodes-base.wait`
- **Resume:** `timeInterval`, 1 second
- **Purpose:** Rate-limits API calls to avoid hitting Gmail rate limits

### 5. Get Label Details
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `get` (label)
- **Label ID:** `{{ $json.id }}`
- **Purpose:** Fetches full label resource including `messagesTotal` and `type`

### 6. Filter Code
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Logic:** Sets `_isActive = true` if `type === 'user' && messagesTotal > 0`
- **Purpose:** Flags labels that should be recorded in the data table

### 7. Is Active? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `{{ $json._isActive }} === true`
- **TRUE:** → Record Active (Upsert)
- **FALSE:** → Loop Over Items (loop back)

### 8. Record Active
- **Type:** `n8n-nodes-base.dataTable`
- **Operation:** `upsert` on `labelName`
- **Table:** Active Labels (`3ZO7iFCJoBdqsqh4`)
- **Columns:** `labelName`, `labelId`, `messagesTotal`
- **Purpose:** Saves or updates the active label record

## Data Flow

1. **Manual Trigger** → starts execution
2. **Get All Labels** → fetches all Gmail labels (id, name, type only)
3. **Loop Over Items** → iterates one label per batch
4. **Wait** → 1-second delay
5. **Get Label Details** → Gmail `label/get` returns full resource including `messagesTotal`
6. **Filter Code** → sets `_isActive` flag
7. **Is Active?** → routes active labels to upsert, inactive back to loop
8. **Record Active** → upserts to Active Labels table (persisted across executions)

## Output

| Field | Type | Description |
|-------|------|-------------|
| `labelName` | string | Gmail label name |
| `labelId` | string | Gmail label ID |
| `messagesTotal` | number | Current message count |

## Dependencies

- **Credentials:** `gmailOAuth2` — Primary Email Account [neo7moruri]
- **Data Table:** `Active Labels` (ID: `3ZO7iFCJoBdqsqh4`)
- **Related:** `delete-orphan-labels` consumes this table

## Related

- Runbook: `../../runbooks/week-01/record-active-labels.md`
