# SCSM Daily Completed

## Overview

Queries the CiresonAnalytics (SCSM) database for Service Requests completed today and Incidents resolved today for the Potch Staff and Student Service Desks, transforms the results into a uniform format, and forwards them to a Power Automate webhook for downstream processing (table registration and email notification).

## Trigger

| Property | Value |
|----------|-------|
| Type | Manual |
| Method | Execute Workflow button in n8n editor |

## Nodes

### 1. Manual Trigger
- **Type:** `n8n-nodes-base.manualTrigger`
- **Purpose:** Starts the workflow on demand
- **Config:** None — manual execution only

### 2. SCSM Query
- **Type:** `n8n-nodes-base.microsoftSql`
- **Operation:** `executeQuery`
- **Credential:** `Service Manager` (Microsoft SQL)
- **Purpose:** Runs two SELECT queries against the `CiresonAnalytics` database
- **Query 1 — Service Requests:** Fetches rows from `dbo.SM_WorkItem_ServiceRequest` where `Status = 'Completed'`, `CompletedDate` is today, and `Tier` is `Potch Staff Service Desk` or `Potch Student Service Desk`
- **Query 2 — Incidents:** Fetches rows from `dbo.SM_WorkItem_Incident` where `Status = 'Resolved'`, `ResolvedDate` is today, and `Tier` is in the same set
- **Output columns:** `WorkItemId`, `Title`, `CreatedByUser`, `Category`, `Tier`, `Description`, `AffectedUser`, `AssignedUser`, `Status`, `Created`, `CompletedDate/ResolvedDate`, `LastModified`, `ClosedDate`

### 3. Transform Results
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForAllItems`
- **Purpose:** Filters input items to only those with `Status = 'Completed'` or `Status = 'Resolved'`, then maps to a uniform output schema

**Code logic:**
```javascript
const pcm = $input.all();
const resolved = pcm.filter(i => {
  return i.json.Status && (i.json.Status === "Completed" || i.json.Status === "Resolved");
});
const response = resolved.map(i => {
  return {
    json: {
      ID: i.json.WorkItemId,
      User: i.json.AffectedUser,
      Tech: i.json.AssignedUser,
      DateResolved: i.json.CompletedDate || i.json.ResolvedDate,
    }
  };
});
return response;
```

- **Output per item:** `{ID, User, Tech, DateResolved}`

### 4. Power Automate Webhook
- **Type:** `n8n-nodes-base.httpRequest`
- **Method:** `POST`
- **URL:** Power Automate HTTP trigger endpoint
- **Body:** JSON array — each element `{ID, User, Tech, DateResolved}` passed via n8n expression
- **Purpose:** Sends the transformed items to a Power Automate flow for table registration and email dispatch

## Data Flow

```
Manual Trigger → SCSM Query (SQL) → Transform Results (filter+map) → Power Automate Webhook (POST)
```

1. User clicks **Execute Workflow** in the n8n editor
2. **SCSM Query** runs two SQL queries against the SCSM data warehouse, returning a combined result set of today's completed SRs and resolved incidents
3. **Transform Results** filters for `Completed`/`Resolved` states and projects to `{ID, User, Tech, DateResolved}`
4. **Power Automate Webhook** POSTs the array to Power Automate's HTTP trigger, which registers the data to a table and sends a notification email

## Output

| Field | Type | Description |
|-------|------|-------------|
| ID | string | SCSM WorkItemId (SR or Incident number) |
| User | string | Affected user name |
| Tech | string | Assigned technician |
| DateResolved | string | Completion/Resolution date (ISO) |

## Dependencies

- **Credentials:** `Service Manager` (Microsoft SQL) — connects to the CiresonAnalytics database on the SCSM data warehouse
- **External services:** Power Automate flow (receives HTTP POST with item array)
- **Database:** CiresonAnalytics (`dbo.SM_WorkItem_ServiceRequest`, `dbo.SM_WorkItem_Incident`)

## Related

- Runbook: `../../runbooks/week-01/scsm-daily-completed.md`
- Power Automate plan: `../../specs/week-01/scsm-power-automate-plan.md`
