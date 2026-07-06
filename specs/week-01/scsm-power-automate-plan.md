# Power Automate: SCSM Completed Items Processor

## Overview

Power Automate flow triggered by the SCSM Daily Completed n8n workflow. Receives an array of `{ID, User, Tech, DateResolved}` items for today's completed Service Requests and resolved Incidents, registers each row in a tracking table, and sends a summary email.

## Trigger

| Property | Value |
|----------|-------|
| Type | When an HTTP request is received |
| Method | POST |
| Environment | `6eec0cd8b5ddec51a4f422b5f26e2b` |
| Workflow ID | `f269a61f2d304ac29b10039deca84af3` |
| Request Body JSON Schema | See below |

### Expected Payload

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "ID": { "type": "string", "description": "SCSM WorkItemId" },
      "User": { "type": "string", "description": "Affected user" },
      "Tech": { "type": "string", "description": "Assigned technician" },
      "DateResolved": { "type": "string", "description": "Completion/resolution datetime" }
    },
    "required": ["ID", "User", "Tech", "DateResolved"]
  }
}
```

## Flow Architecture

```
HTTP Trigger
  │
  ├→ Parse JSON (validate body against schema)
  │
  ├→ Apply to Each (loop over items)
  │    │
  │    └→ Upsert Row to Tracking Table
  │         - Check if ID already exists
  │         - If yes: update DateResolved, Tech
  │         - If no: insert new row
  │
  └→ Send Email Notification
       - To: [service desk team / manager]
       - Subject: "SCSM Daily Completed — {{count}} items ({{date}})"
       - Body: Summary table of today's items
```

## Nodes

### 1. HTTP Trigger (When an HTTP request is received)

- **Method:** POST
- **Relative path:** (empty — root trigger)
- **Request body JSON schema:** See above
- **Output:** `triggerBody()` — the JSON array of completed items

### 2. Parse JSON

- **Input:** `triggerBody()`
- **Schema:** The JSON schema above
- **Purpose:** Validates the incoming payload and provides strongly-typed token references

### 3. Apply to Each

- **Input:** `body('Parse_JSON')` — iterate over each item in the array
- **Purpose:** Process each SCSM item individually for table registration

#### Inside loop: Upsert Row to Tracking Table

**Choose your table backend:**

| Option | Connector | Notes |
|--------|-----------|-------|
| **Dataverse** (Recommended) | Microsoft Dataverse | Native Power Platform, relational, schema below |
| **SharePoint List** | SharePoint | Easier setup, good for reporting |
| **SQL Server** | SQL Server | If SCSM data warehouse access is available |

**Dataverse table schema — `scsm_daily_completed`:**

| Column | Type | Description |
|--------|------|-------------|
| `scsm_workitemid` | Text (Primary) | SCSM WorkItemId (SR/Incident number) |
| `scsm_user` | Text | Affected user name |
| `scsm_tech` | Text | Assigned technician |
| `scsm_dateresolved` | Date/Time | ISO datetime of completion/resolution |
| `scsm_createdon` | Date/Time (auto) | When this row was first created |
| `scsm_modifiedon` | Date/Time (auto) | When this row was last updated |

**Upsert logic:**
1. `List rows` filtered by `scsm_workitemid eq '<ID>'`
2. If row exists → `Update row` (set `scsm_tech`, `scsm_dateresolved`, `scsm_modifiedon`)
3. If row doesn't exist → `Add row` (insert all fields)

**SharePoint List schema — `SCSM Daily Completed`:**

| Column | Type | Description |
|--------|------|-------------|
| Title | Text (Single line) | WorkItemId (SR/Incident number) |
| User | Text (Single line) | Affected user name |
| Tech | Text (Single line) | Assigned technician |
| DateResolved | Date/Time | Completion/resolution datetime |
| CreatedOn | Date/Time (auto) | Row creation timestamp |
| ModifiedOn | Date/Time (auto) | Row update timestamp |

### 4. Send Email Notification

- **Connector:** Office 365 Outlook (or other email provider)
- **To:** `[service-desk-team@domain]`
- **Subject:** `SCSM Daily Completed — {{length(body('Parse_JSON'))}} items ({{formatDateTime(utcNow(), 'yyyy-MM-dd')}})`
- **Body (HTML):**

```html
<h2>SCSM Daily Completed — {{itemsCount}} items</h2>
<p>Run date: {{utcNow}}</p>
<table border="1" cellpadding="4" style="border-collapse: collapse; width: 100%;">
  <tr style="background: #f0f0f0;">
    <th>Work Item ID</th>
    <th>Affected User</th>
    <th>Assigned Technician</th>
    <th>Date Resolved</th>
  </tr>
  <tr>
    <td>{{item.ID}}</td>
    <td>{{item.User}}</td>
    <td>{{item.Tech}}</td>
    <td>{{item.DateResolved}}</td>
  </tr>
</table>
```

Use **Send an email with options** or compose the HTML table dynamically via a **Compose** + **Apply to Each** to build rows from every item.

## Implementation Steps

1. **Open Power Automate maker portal**
   - Go to `https://make.powerautomate.com`
   - Select the environment containing `6eec0cd8b5ddec51a4f422b5f26e2b`

2. **Locate or create the flow**
   - Open the existing flow `f269a61f2d304ac29b10039deca84af3` (already receiving webhook calls from n8n)
   - Or create a new **Instant cloud flow** → trigger **When an HTTP request is received**

3. **Add Parse JSON action**
   - Use the schema from above

4. **Add Apply to Each → Upsert logic**
   - Choose table backend (recommend Dataverse if available, otherwise SharePoint)
   - Implement the check-exists-then-update-or-insert pattern

5. **Add email notification**
   - Compose HTML table from all items
   - Send via Office 365 Outlook

6. **Test end-to-end**
   - Run the SCSM Daily Completed workflow in n8n
   - Verify items appear in the tracking table
   - Verify email is received with correct count and data

## Priority Expansion Roadmap

| Phase | Feature | Description |
|-------|---------|-------------|
| 1 (now) | Basic table + email | Register rows and send summary email as described above |
| 2 | Attach SCSM ticket PDF | Have Power Automate fetch ticket details from SCSM API/DB and attach to email |
| 3 | Dashboard (Power BI) | Connect tracking table to Power BI for trend reporting on completion volumes, tech workloads |
| 4 | Escalation alerts | If a tier has >N items, send additional alert to manager |
| 5 | SLA breach detection | Cross-reference resolved dates against SLA targets and flag breaches |

## Related

- Spec: `../../specs/week-01/scsm-daily-completed.md`
- Runbook: `../../runbooks/week-01/scsm-daily-completed.md`
