# SCSM Daily Completed

## Trigger Instructions

The workflow runs **manually only** (no schedule). To execute:

1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right)
3. The workflow will:
   - Query SCSM for today's completed Service Requests + resolved Incidents
   - Transform the results to `{ID, User, Tech, DateResolved}`
   - POST to the Power Automate webhook
4. Check the Power Automate flow's run history to verify table registration and email delivery

## Expected Behavior

1. Manual trigger starts the pipeline
2. **SCSM Query** returns a combined result set — rows from both Service Requests (Status = Completed) and Incidents (Status = Resolved) for today, filtered to Potch Staff/Student Service Desk tiers
3. **Transform Results** drops any rows with unexpected Status values and maps to the uniform `{ID, User, Tech, DateResolved}` schema
4. **Power Automate Webhook** sends the array to Power Automate, which registers each item to a tracking table and sends a summary email

A successful execution shows all 4 nodes green with the Power Automate returning HTTP 200.

## Monitoring

- **Success indicators:** Power Automate flow completes successfully; email notification received
- **Failure indicators:** n8n execution shows error on SCSM Query (DB connection), Transform Results (code error), or Power Automate Webhook (network/auth failure)
- **Check:** n8n execution log — `success` status with 4 nodes passed

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| SCSM Query fails | Database connection issue or credential expired | Verify `Service Manager` credential in n8n; check SCSM data warehouse is reachable from evo |
| Transform Results errors | Unexpected Status value or missing field | Check SQL output in execution log; verify column names match expected schema |
| Power Automate Webhook returns 401/403 | Power Automate trigger URL expired or revoked | Re-generate the HTTP trigger URL in Power Automate and update the node's URL parameter |
| No items sent | No SRs completed or Incidents resolved today | This is normal on quiet days — the workflow sends an empty array or nothing |
| Power Automate flow doesn't run | Flow was turned off or deleted | Open Power Automate maker portal and verify the triggered flow is turned on |

## Recovery

1. Navigate to n8n → Executions → filter by this workflow
2. Check the latest execution status — if `error`, click to see which node failed
3. **SCSM Query failure:** click the node to see the database error message; re-test credentials
4. **Transform Results failure:** inspect the node's input data in the execution log; the Code node will show the raw SQL output that caused the issue
5. **Power Automate Webhook failure:** copy the payload from the node output and test manually with `curl` against the trigger URL
6. To re-send missed data, re-run the workflow (click Execute Workflow again)
7. If no email was sent despite a successful workflow run, check the Power Automate flow's run history (maker portal → Flows → Runs)

## Related

- Spec: `../../specs/week-01/scsm-daily-completed.md`
- Power Automate plan: `../../specs/week-01/scsm-power-automate-plan.md`
