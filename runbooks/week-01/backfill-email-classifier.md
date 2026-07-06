# Backfill Email Classifier

## Trigger Instructions

The workflow runs **manually only** (no schedule trigger). Two ways to test:

**Manual test (recommended):**
1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right) — triggers the Manual Trigger
3. The workflow fetches up to 100 unread inbox emails and processes them one-by-one
4. Watch execution progress in the n8n execution log

**Schedule test:** Not available — this workflow has no schedule trigger.

## Prerequisites

- The **Label Resolver** sub-workflow must be deployed and accessible
- Gmail must have labels matching the hierarchical taxonomy (Label Resolver creates missing ones)
- Ollama must be running on evo with `qwen2.5-coder:14b` loaded

## Expected Behavior

1. Fetches 100 unread inbox messages
2. For each message:
   a. Checks if already has "Processed" label — skips if so
   b. Transforms Gmail output for AI
   c. AI Agent classifies into up to 2 hierarchical labels
   d. Each label is resolved (or created) via Label Resolver sub-workflow
   e. Processed label is resolved inline
   f. All labels are applied to the Gmail message
    g. Promotional/Notifications emails are archived
3. 3-second delay between emails to avoid rate limits
4. Loop continues until all 100 emails are processed or time runs out

## Monitoring

- **Success indicators:** Labels appearing on inbox emails in Gmail; execution log shows 28-node completion
- **Failure indicators:** No labels applied despite unprocessed emails; "Needs Review" appearing (AI failed)
- **Check:** n8n execution log for each batch run

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| AI Agent empty/timeout | Ollama not responding | Verify `evo:11434` reachable; `ollama ps` shows the model |
| "Needs Review" on many emails | Prompt mismatch or model issue | Check AI Agent output in execution log; adjust prompt |
| Gmail API rate limit (429) | Too many requests | Increase Wait node delay beyond 3s |
| Label Resolver errors | Sub-workflow not deployed | Verify Label Resolver workflow exists and is accessible |
| Processed label not applied | Inline resolution failed | Check Get Labels → Find Processed chain in execution log |

## Recovery

1. If a batch execution errors, fix the issue and re-run — already-processed emails are skipped
2. If AI Agent produces systematic bad classifications, review and adjust the system prompt
3. If Gmail rate-limited, increase the per-email delay
4. To re-process a specific email, remove its "Processed" label in Gmail and re-run
5. Monitor n8n execution timeout — 100 emails × ~5s each = ~8-9 min total

## Related

- Spec: `../../specs/week-01/backfill-email-classifier.md`
- Spec: `../../specs/week-01/label-resolver.md`
- Spec: `../../specs/week-01/gmail-manager.md`
