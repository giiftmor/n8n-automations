# Gmail Manager

## Trigger Instructions

The workflow runs automatically on a **1-minute polling schedule**. No manual trigger needed.

To manually test:
1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right)
3. Check pinned data on Gmail Trigger node for sample emails
4. Verify the AI Agent output produces a valid label ID

## Expected Behavior

1. Every minute, Gmail Trigger checks for new inbox messages
2. Each new message is passed through the AI Agent for classification
3. The AI performs prior-email-history checks (searches inbox + sent for sender)
4. Based on content + history, the AI selects one of six label IDs
5. The label is applied to the message in Gmail

A correctly classified email will appear in Gmail with the appropriate label. No visible output beyond the label being applied.

## Monitoring

- **Success indicators:** Labels appearing on processed emails in Gmail
- **Failure indicators:** Unlabeled messages in inbox, error executions in n8n
- **Check:** n8n execution log — should see `success` status for each poll cycle

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| AI Agent timeout | Ollama model not responding on evo | Check `evo:11434` is reachable; verify deepseek-coder-v2:16b is loaded |
| Gmail auth failure | OAuth token expired | Re-authenticate the `gmailOAuth2` credential in n8n |
| Invalid label ID | AI returned malformed output | Check AI Agent output in execution log; may need prompt tweak |
| No emails processed | Gmail Trigger filter too restrictive | Verify trigger has no active filters |

## Recovery

1. Navigate to n8n → Executions → filter by this workflow
2. Check the latest execution status — if `error`, click to see which node failed
3. Most failures are at the AI Agent step: verify Ollama is running (`curl evo:11434/api/tags`)
4. If Gmail rate-limited, the workflow will retry automatically on next poll
5. To re-process emails, deactivate/reactivate the workflow (or manually run individual items)

## Related

- Spec: `../../specs/week-01/gmail-manager.md`
