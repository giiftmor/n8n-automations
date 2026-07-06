# Delete Orphan Labels

## Overview

Reads the **Active Labels** data table (populated by `record-active-labels`), then loops through every Gmail label. Deletes user-type labels with `messagesTotal === 0` that are not in the table and not in the protected list. This keeps the Gmail label namespace clean without risking active or wanted labels.

## Trigger

| Property | Value |
|----------|-------|
| Type | Manual |
| Auth | Workspace-level n8n API key |

## Nodes

### 1. Manual Trigger
- **Type:** `n8n-nodes-base.manualTrigger`
- **Purpose:** Starts execution on demand

### 2. ActiveLabels (Data Table Get)
- **Type:** `n8n-nodes-base.dataTable`
- **Operation:** `get` (row)
- **Table:** Active Labels (`3ZO7iFCJoBdqsqh4`)
- **Return All:** `true`
- **Purpose:** Fetches all recorded active label names for reference during the loop

### 3. Get All Labels
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `getAll` (label)
- **Return All:** `true`
- **Purpose:** Returns all Gmail labels — triggers in parallel with ActiveLabels node

### 4. Loop Over Labels (SplitInBatches)
- **Type:** `n8n-nodes-base.splitInBatches`
- **Batch Size:** 1
- **Purpose:** Iterates each label individually for detail fetch and delete decision

### 5. Get Label Details
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `get` (label)
- **Label ID:** `{{ $json.id }}`
- **Purpose:** Fetches full label resource including `messagesTotal` needed for the deletion check

### 6. Check Deletable
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Logic:** Reads `$("ActiveLabels").all()` to get the table's active label names. Sets `_isDeletable = true` only if all conditions met:
  - `type === 'user'`
  - `messagesTotal === 0`
  - Not in the Active Labels table
  - Not in the hardcoded `protectedNames` list

### 7. Is Deletable? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `{{ $json._isDeletable }} === true`
- **TRUE:** → Wait → Delete Label → loop back
- **FALSE:** → Loop Over Labels (loop back)

### 8. Wait
- **Type:** `n8n-nodes-base.wait`
- **Resume:** `timeInterval`, 1 second
- **Purpose:** Rate-limits Gmail API delete calls

### 9. Delete Label
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `delete` (label)
- **Label ID:** `{{ $json.id }}`
- **Purpose:** Permanently deletes the orphan label from Gmail

## Data Flow

1. **Manual Trigger** → starts parallel execution
2. **ActiveLabels** → reads previously-recorded active label names from data table
3. **Get All Labels** → fetches all Gmail labels (id, name, type)
4. **Loop Over Labels** → iterates one label per batch
5. **Get Label Details** → Gmail `label/get` returns full resource
6. **Check Deletable** → compares against data table + protected list
7. **Is Deletable?** → routes deletable labels to deletion, others back to loop
8. **Wait** → 1-second delay before each delete
9. **Delete Label** → removes user label from Gmail

## Protected Labels

The `protectedNames` list in the Check Deletable Code node prevents deletion of main hierarchical labels that may have `messagesTotal === 0` while sub-labels are active:

```
Finance, Personal, Tracking, Infrastructure, Meetings,
Promotional, Legal, Social, Needs Review, CATEGORIZED,
INBOX, UNREAD, Processed
```

## Output

Each deleted label is logged in the execution output. Labels not matching the deletion criteria are silently skipped (no data emitted).

## Dependencies

- **Credentials:** `gmailOAuth2` — Primary Email Account [neo7moruri]
- **Data Table:** `Active Labels` (ID: `3ZO7iFCJoBdqsqh4`)
- **Prerequisite:** `record-active-labels` must have been run at least once to populate the table

## Related

- Runbook: `../../runbooks/week-01/delete-orphan-labels.md`
