# Gmail Manager

## Trigger Instructions

The workflow runs automatically on a **1-minute polling schedule**. To manually test:
1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right)
3. Pinned data on Gmail Trigger provides sample emails to test against
4. Verify the AI Agent output produces a valid label ID

## Expected Behavior

1. Every minute, Gmail Trigger checks for new inbox messages
2. Each new message is passed through the AI Agent for classification
3. Based on email content + existing Gmail labels, the AI selects one of six label IDs
4. The label is applied to the message in Gmail

A correctly classified email will appear in Gmail with the appropriate label.

## Monitoring

- **Success indicators:** Labels appearing on processed emails in Gmail
- **Failure indicators:** Unlabeled messages in inbox, error executions in n8n
- **Check:** n8n execution log — should see `success` status for each poll cycle

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| AI Agent timeout/empty output | Ollama model not responding on evo | Check `evo:11434` is reachable; verify `qwen2.5-coder:14b` is loaded (`ollama ps`) |
| Gmail auth failure | OAuth token expired | Re-authenticate the `gmailOAuth2` credential in n8n |
| Invalid label ID | AI returned malformed output | Check AI Agent output in execution log; may need prompt tweak |
| No emails processed | Gmail Trigger filter too restrictive | Verify trigger has no active filters |

## Recovery

1. Navigate to n8n → Executions → filter by this workflow
2. Check the latest execution status — if `error`, click to see which node failed
3. Most failures are at the AI Agent step: verify Ollama is running (`ssh evo@evo ollama ps`)
4. If Gmail rate-limited, the workflow will retry automatically on next poll
5. To re-process emails, deactivate/reactivate the workflow

## Related

- Spec: `../../specs/week-01/gmail-manager.md`
