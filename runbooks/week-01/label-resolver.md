# Label Resolver

## Trigger Instructions

This is a **sub-workflow** — never triggered manually. It is called by Gmail Manager and Backfill Email Classifier via the Execute Sub-workflow node.

To test in isolation:
1. Open the workflow in n8n editor
2. Pin sample input data on the Execute Workflow Trigger node:
   ```json
   {"labelName": "Finance/Invoices", "fallbackLabelId": "NEW_LABEL_Finance/Invoices", "mainLabelName": "Finance"}
   ```
3. Click **Execute Workflow** (top right)
4. Trace through the 12-node pipeline: Use Fallback? → Get All Labels → Resolve Main → Create Main (if needed) → Pipeline Main → Is Sub? → Resolve Sub → Create Sub (if needed) → Format Result
5. Verify the output returns `{labelId, labelName}` — either an existing ID or a newly created label

## Expected Behavior

1. Receives `{labelName, fallbackLabelId, mainLabelName}` from parent workflow
2. If `fallbackLabelId` is a real Gmail label ID (not `NEW_LABEL_` prefixed), returns it immediately — this bypasses all resolution/creation
3. Otherwise, normal resolution flow:
   a. Fetches all Gmail labels
   b. **Resolve Main** — checks if the parent label (`mainLabelName`) exists
   c. If parent doesn't exist, **Create Main** creates it
   d. **Pipeline Main** normalizes the parent label ID from both paths
   e. **Is Sub?** — if `labelName === mainLabelName` (single-level label), skips to Format Result
   f. **Resolve Sub** — checks if the full hierarchical label (`labelName`) exists
   g. If child doesn't exist, **Create Sub** creates it — Gmail nests it under the parent since parent already exists
   h. **Format Result** normalizes output to `{labelId, labelName}`
4. Parent-created-before-child is guaranteed by sequential IF routing through separate nodes

## Monitoring

- **Success indicators:** Execution returns `{labelId, labelName}` (never empty)
- **Failure indicators:** Execution errors at specific node (check which of the 12 nodes failed); Gmail API quota exceeded; label not created

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| Create Main fails | Gmail API quota or invalid name | Check Gmail API quota; verify `mainLabelName` is not empty |
| Create Sub fails but Create Main succeeded | Gmail API error on hierarchical name | Verify parent was created; check label name constraints |
| Format Result returns null | All labelId sources undefined | Check `fallbackLabelId`, `mainLabelId`, `labelId`, and `$json.id` in node output |
| Pipeline Main loses context | `$("code-main").first()` returns wrong item | Sub-workflow processes one item per execution — `.first()` should match |

## Recovery

1. If label creation fails mid-execution, check Gmail API quota in Google Cloud Console
2. If Pipeline Main fails, check that Resolve Main output retains `labelName`, `mainLabelName`, and `fallbackLabelId`
3. Manually create missing labels in Gmail, or re-trigger the parent workflow
4. Use n8n execution log to trace which node in the 12-node chain failed

## Related

- Spec: `../../specs/week-01/label-resolver.md`
