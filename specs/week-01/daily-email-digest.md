# Daily Email Digest

## Overview

Sends a daily summary of the previous day's classified emails to a Discord channel via webhook. Runs every morning at 8 AM.

## Trigger

| Property | Value |
|----------|-------|
| Type | Schedule |
| Time | 8:00 AM daily |

## Nodes

### 1. Schedule Trigger
- **Type:** `n8n-nodes-base.scheduleTrigger`
- **Schedule:** Daily at 08:00

### 2. Gmail (Search)
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `getAll` (message)
- **Query:** `in:inbox after:yesterday` (or `newer_than:1d`)
- **Limit:** 50
- **Purpose:** Fetches emails classified in the last 24 hours

### 3. Format Digest (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForAllItems`
- **Purpose:** Structures email data into a formatted Discord message payload
- **Output:** Discord embed format with `{embeds: [{title, description, fields: [{name, value}]}]}`

### 4. HTTP Request (Discord Webhook)
- **Type:** `n8n-nodes-base.httpRequest`
- **Method:** POST
- **URL:** `{{ $env.DISCORD_DIGEST_WEBHOOK_URL }}`
- **Purpose:** Sends the digest to Discord

## Dependencies

- **Credentials:** `gmailOAuth2` — Primary Email Account [neo7moruri]
- **Env var:** `DISCORD_DIGEST_WEBHOOK_URL` must be set in n8n instance

## Related

- Runbook: `../../runbooks/week-01/daily-email-digest.md`
