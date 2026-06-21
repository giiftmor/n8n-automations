# Gmail Manager

## Overview

Polls Gmail inbox every minute, uses an Ollama AI agent (deepseek-coder-v2:16b) to classify each email into one of six categories, and applies the corresponding Gmail label.

## Trigger

| Property | Value |
|----------|-------|
| Type | Gmail Trigger (polling) |
| Schedule | Every minute |
| Filters | None — processes all new inbox messages |

## Nodes

### 1. Gmail Trigger
- **Type:** `n8n-nodes-base.gmailTrigger`
- **Purpose:** Polls Gmail for new messages
- **Config:** No filters, polls every minute

### 2. Gmail (Get full message)
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `get`
- **Input:** `{{ $json.id }}` from trigger
- **Purpose:** Retrieves full message content including headers, body, labels

### 3. AI Agent
- **Type:** `@n8n/n8n-nodes-langchain.agent`
- **Model:** Ollama Chat Model (`deepseek-coder-v2:16b`)
- **System prompt:** Long classification prompt with label descriptions, prior-email-history checks
- **Output parser:** Structured Output Parser (JSON schema: `{ label, label ID }`)
- **Tools:**
  - **Get Email** (`gmailTool`) — searches inbox for prior emails from sender
  - **Check Sent** (`gmailTool`) — searches sent folder for prior emails to sender

### 4. Gmail1 (Apply label)
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `addLabels`
- **Input:** `{{ $('Gmail').item.json.id }}`
- **Label:** `{{ $json.output['label ID'] }}` (from AI parser)

## Data Flow

1. **Gmail Trigger** polls inbox → new message IDs
2. **Gmail** fetches full message data (headers, body, labels)
3. **AI Agent** receives message data + runs classification using:
   - Prior email history lookups (Get Email + Check Sent tools)
   - 6-category classification system
4. **Structured Output Parser** extracts `{ label, label ID }` from AI response
5. **Gmail1** applies the label to the original message

## Classification Labels

| Label Name | Label ID | Criteria |
|------------|----------|----------|
| To Respond | `Label_5151750749488724401` | Requires direct action/reply |
| FYI | `Label_7518716752151077752` | Awareness only, no action needed |
| Comment | `Label_1470120230130814788` | Comment on document/task |
| Notification | `Label_9` | Service/account updates, policy changes |
| Meeting Update | `Label_7958083858424702466` | Meeting invites/changes |
| Marketing | `Label_5274550748854913384` | Unsolicited/cold promotional emails |

## Dependencies

- **Credentials:**
  - `gmailOAuth2` — Primary Email Account [neo7moruri]
  - `ollamaApi` — Ollama Models (evo:11434)
- **Model:** `deepseek-coder-v2:16b` on evo

## Related

- Runbook: `../../runbooks/week-01/gmail-manager.md`
