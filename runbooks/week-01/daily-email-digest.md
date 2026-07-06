# Daily Email Digest

## Trigger Instructions

The workflow runs automatically at **8:00 AM daily**. To manually test:
1. Open the workflow in n8n editor
2. Click **Execute Workflow** (top right)
3. Verify the Gmail search fetches recent emails
4. Check Discord channel for the formatted digest message

## Prerequisites

- `DISCORD_DIGEST_WEBHOOK_URL` environment variable must be set in n8n
- The Discord webhook must be active and pointing to the correct channel

## Expected Behavior

1. Daily at 8 AM, searches Gmail for emails from the last 24 hours
2. Formats them into a Discord embed with fields per-email (sender, subject, label)
3. Sends the formatted payload to the configured Discord webhook URL
4. Digest provides a quick overview of what came in overnight

## Monitoring

- **Success indicators:** Discord channel shows the daily digest message
- **Failure indicators:** No message in Discord; execution error in n8n
- **Check:** n8n execution log; Discord channel

## Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| HTTP 400 from Discord | Malformed webhook payload | Check Code node output format matches Discord embed schema |
| No digest message | Webhook URL not configured | Set `DISCORD_DIGEST_WEBHOOK_URL` in n8n environment variables |
| No emails returned | Gmail search query too narrow | Adjust query to `newer_than:1d` or remove query to get all recent |
| Webhook fails | Discord webhook deleted/recreated | Update the webhook URL in n8n environment variables |

## Recovery

1. If the digest fails to send, check `DISCORD_DIGEST_WEBHOOK_URL` is set correctly
2. Test the webhook with curl: `curl -X POST -H "Content-Type: application/json" -d '{"content":"test"}' $WEBHOOK_URL`
3. Re-run the workflow manually after fixing the issue
4. Missed digests are not retroactively sent — only the current day's data

## Related

- Spec: `../../specs/week-01/daily-email-digest.md`
