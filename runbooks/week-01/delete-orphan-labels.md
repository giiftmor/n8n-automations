# Runbook: Delete Orphan Labels

## Trigger Instructions

**Prerequisite:** Run `record-active-labels` first to populate the Active Labels data table.

1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right)
3. The workflow fetches the data table, then loops through every Gmail label checking deletion criteria
4. Only user labels with `messagesTotal === 0` that are neither in the table nor the protected list are deleted
5. Check the execution log to see which labels were deleted

## Expected Behavior

1. Manual trigger starts execution
2. **ActiveLabels** node reads all recorded active label names from the data table
3. **Get All Labels** fetches all Gmail labels in parallel
4. Each label's full details are fetched, then checked against the table + protected list
5. Matching labels are deleted with a 1-second delay between each

## Monitoring

- **Success indicators:** Execution log shows labels deleted; orphan labels no longer appear in Gmail
- **Failure indicators:** No labels deleted but empty orphan labels exist (check table is populated); Gmail API errors

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| No labels deleted | Table not yet populated | Run `record-active-labels` first |
| Wanted label deleted | `protectedNames` list missing it | Add the label name to the `protectedNames` array in the Check Deletable Code node |
| Gmail API 404 on delete | Label already deleted by another process | Safe to ignore — the label no longer exists |
| Gmail rate limit (429) | Too many delete calls | Increase Wait node delay to 2-3 seconds |

## Recovery

1. If a wanted label was deleted accidentally, recreate it in Gmail
2. Add its name to the `protectedNames` list in Check Deletable to prevent future deletion
3. Re-run `record-active-labels` to capture the recreated label (if it gets messages)

## Related

- Spec: `../../specs/week-01/delete-orphan-labels.md`
