# Runbook: Record Active Labels

## Trigger Instructions

1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right)
3. The workflow loops through every Gmail label — this takes ~2-3s per label (1s wait + API calls)
4. Verify in the execution log that labels with `messagesTotal > 0` were upserted to the data table

## Expected Behavior

1. Manual trigger starts execution
2. All Gmail labels are fetched (returns id, name, type only)
3. Each label is processed individually: full details are fetched, `type` and `messagesTotal` are evaluated
4. User labels with `messagesTotal > 0` are upserted to the Active Labels data table (keyed by `labelName`)
5. System labels and empty user labels are looped back without recording

## Monitoring

- **Success indicators:** Execution log shows `success` status; Active Labels data table contains entries with non-zero `messagesTotal`
- **Failure indicators:** Execution errors, Gmail API rate-limit errors, no rows in data table

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| Gmail rate limit (429) | Too many label/get calls | Increase Wait node delay to 2-3 seconds |
| Data table upsert fails | Schema mismatch | Verify table columns: `labelName` (string), `labelId` (string), `messagesTotal` (number) |
| Empty data table after run | Filter Code logic wrong or all labels are system-type | Run manually and inspect Filter Code output in n8n execution log |

## Recovery

1. If the workflow errors mid-run, check which label caused the failure (execution log)
2. Fix the issue and re-run — upsert means re-processing already-recorded labels is safe
3. To repopulate the table from scratch, delete all rows via the Data Table UI, then re-run

## Related

- Spec: `../../specs/week-01/record-active-labels.md`
