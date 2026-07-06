# Gmail Manager

## Trigger Instructions

The workflow runs automatically on a **1-minute polling schedule**. To manually test:
1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right)
3. Pinned data on Gmail Trigger provides sample emails to test against
4. Trace execution through the pipeline: Transform → AI Agent → Parse JSON → Extract Labels → Label Resolver → Gmail1
5. Verify the final email carries up to 3 labels (2 classification + Processed)

## Expected Behavior

1. Every minute, Gmail Trigger checks for new inbox messages
2. Each new message is fetched (full content) and transformed into AI-friendly format
3. AI Agent classifies email into up to 2 hierarchical labels (e.g. `Finance/Invoices`, `Personal/Family`)
4. Parse JSON Code node handles both valid and malformed AI output
5. Extract Labels splits multi-label output into individual resolution paths
6. Label Resolver sub-workflow checks if each label exists in Gmail, creating parent-first if needed
7. Inline Processed label resolution finds or creates `Tracking/Processed`
8. All resolved labels are merged and applied to the message
9. Archive decision per-mainLabel — Promotional/Notifications emails are archived

A correctly classified email will appear in Gmail with the corresponding label(s) and possibly archived.

## Monitoring

- **Success indicators:** Labels appearing on processed emails in Gmail; execution log shows all 19 nodes passing
- **Failure indicators:** Unlabeled messages in inbox ("Needs Review" label applied = AI classification failed); error executions
- **Check:** n8n execution log — should see `success` for each poll cycle

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| AI Agent empty output | Ollama timeout or model unloaded | Check `evo:11434` reachable; `ollama ps` for `qwen2.5-coder:14b` |
| Gmail auth failure | OAuth token expired | Re-authenticate `gmailOAuth2` credential in n8n |
| Malformed AI output (Needs Review fallback) | Model returned invalid JSON | Check AI Agent output in execution log; `numPredict: 1000` should prevent truncation |
| Label Resolver fails | Gmail label creation rate-limited | Verify Gmail API quota; check Label Resolver execution log |
| No emails processed | Gmail Trigger filter too restrictive | Verify trigger has no active filters |

## Recovery

1. Navigate to n8n → Executions → filter by this workflow
2. Check latest execution status — if `error`, click to see which node failed
3. Most failures are at AI Agent step: verify Ollama is running (`ssh evo@evo ollama ps`)
4. If Label Resolver sub-workflow fails, inspect its execution log separately
5. To re-process emails, deactivate/reactivate the workflow
6. For stuck items with "Needs Review" label, manually reclassify or adjust prompt

## Related

- Spec: `../../specs/week-01/gmail-manager.md`
- Spec: `../../specs/week-01/label-resolver.md`
