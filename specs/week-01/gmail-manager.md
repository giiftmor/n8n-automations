# Gmail Manager

## Overview

Polls Gmail inbox every minute, classifies each email using an Ollama-hosted LLM into a hierarchical multi-label taxonomy (up to 2 labels per email), and applies the corresponding Gmail labels. Includes inline Processed-label resolution, RAG-based few-shot example retrieval from Qdrant, parent-first label creation via sub-workflow, archive routing per-label, and error handling with a "Needs Review" fallback.

## Trigger

| Property | Value |
|----------|-------|
| Type | Gmail Trigger (polling) |
| Schedule | Every minute |
| Filters | None — processes all new inbox messages |

## Nodes (20)

### 1. Gmail Trigger
- **Type:** `n8n-nodes-base.gmailTrigger`
- **Purpose:** Polls Gmail for new message IDs
- **Config:** No filters, polls every minute

### 2. Gmail (Get full message)
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `get`
- **Simple mode:** `true` — outputs structured JSON
- **Input:** `{{ $json.id }}` from trigger
- **Purpose:** Retrieves full message content

### 3. Get All Labels
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `getAll` (label)
- **Return All:** `true`
- **Purpose:** Fetches all Gmail labels for inline Processed label resolution

### 4. Find Processed (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForAllItems`
- **Purpose:** Searches label list for `name === 'Processed'` — sets `processedLabelId` or `null`

### 5. Exists? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `$json.exists === true`
- **TRUE:** → Done (skip)
- **FALSE:** → Create Processed

### 6. Create Processed
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `create` (label)
- **Name:** `Processed`
- **Purpose:** Creates the "Processed" label if it doesn't exist

### 7. Done (Code)
- **Type:** `n8n-nodes-base.code`
- **Purpose:** Returns `{processedLabelId, exists: true}` from find or creation

### 8. Enrich (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Merges `processedLabelId` from Done node into each email item
- **Output:** Original email fields + `processedLabelId`

### 9. Transform (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Maps Gmail's flat simple-mode output (`From: "Name <email>"`, `Subject`, `labels: [{id,name}]`) to the nested format expected by the AI Agent prompt
- **Output fields:** `from.value[0].{name,address}`, `to.value[0].address`, `subject`, `labelIds: ["id1",...]`, `text: snippet`, `headers`

### 10. Embed & Search Qdrant (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Embeds the email text via Ollama (`nomic-embed-text`) and queries Qdrant `email-labels` collection for top-5 similar classified emails. Injects RAG examples as `ragExamples` field for AI Agent context.
- **Fallback:** If Qdrant/Ollama unavailable, proceeds with `"No similar classified emails found."`

### 11. AI Agent
- **Type:** `@n8n/n8n-nodes-langchain.agent`
- **Version:** 1.9
- **Model:** Ollama Chat Model (`hhao/qwen2.5-coder-tools:14b`) on evo:11434
- **Format:** `json`, **Temperature:** `0.1`, **numPredict:** `1000`
- **HasOutputParser:** `false` — output parsed by Code node instead
- **System prompt:** 8-label taxonomy; up to 2 labels per email; explicit JSON output format with `labels[]` array; RAG examples injected via `{{ $json.ragExamples }}`
- **On Error:** `continueErrorOutput` — routes to error handler

### 12. Code in JavaScript (Parse JSON)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Parses `$json.output` from AI Agent via `JSON.parse()`; produces `id`, `fromName`, `fromAddress`, `messageId`, `threadId`, `proposedLabels` (array of `{main_label, sub_label, label_name}`)
- **Error handling:** `try/catch` — malformed JSON falls back to `mainLabel: "Needs Review"`, empty `proposedLabels`

### 13. Extract Labels (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForAllItems` — splits `proposedLabels[]` into one item per label
- **Output per item:** `id`, `labelName`, `mainLabelName`, `fallbackLabelId`, `mainLabel`, `subLabel`
- **Purpose:** Breaks multi-label classification into individual execution items so each flows independently through label resolution

### 14. Resolve Label (Execute Workflow)
- **Type:** `n8n-nodes-base.executeWorkflow`
- **Workflow:** Label Resolver (`ZbxbmVOmibKzD3Wz`)
- **Input:** `{labelName, fallbackLabelId, mainLabelName}`
- **Purpose:** Calls sub-workflow to resolve each label — returns `{messageId, labelId, labelName}` (existing ID or newly created via parent-first strategy)
- **Mode:** `each` — one sub-workflow call per label item

### 15. Restore Context (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Re-attaches `mainLabel`/`subLabel` context after sub-workflow call (which only outputs `messageId`/`labelId`/`labelName`)
- **Output:** `{id, labelName, mainLabel, subLabel, labelId}`

### 16. Gmail1 (Apply labels)
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `addLabels`
- **Label IDs:** `{{ [$json.labelId, $('Enrich').item.json.processedLabelId].filter(Boolean) }}`
- **Purpose:** Applies the resolved classification label + Processed label to the original message

### 17. Handle Error (Code)
- **Type:** `n8n-nodes-base.code`
- **Purpose:** Fallback code node connected to AI Agent's error output
- **Action:** Returns `{labelId: "Label_9"}` (Needs Review) — ensures error doesn't crash workflow

### 18. Archive Router (Switch)
- **Type:** `n8n-nodes-base.switch`
- **Condition:** `{{ $json.mainLabel }}`
- **Rules:** `Promotional` or `Notifications` → Archive; all others → Keep in inbox
- **Purpose:** Archives promotional and notification emails; keeps actionable emails in inbox

### 19. Archive Email
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `removeLabel` (INBOX)
- **Message ID:** `{{ $json.id }}`
- **Purpose:** Removes INBOX label, archiving the email

No Merge Labels node — each label item flows independently, and `labelIds` is constructed directly in the Gmail1 node.

## Data Flow

1. **Gmail Trigger** polls inbox → new message IDs
2. **Gmail** fetches full message in simple mode
3. **Processed resolution** (inline, parallel) resolves Processed label ID
4. **Enrich** merges Processed label ID
5. **Transform** maps Gmail flat output to AI-friendly format
6. **Embed & Search Qdrant** retrieves top-5 similar classified emails via RAG
7. **AI Agent** classifies email using taxonomy + RAG examples
8. **Parse JSON** flattens AI output; malformed JSON falls back to "Needs Review"
9. **Extract Labels** splits `labels[]` into individual items (one per label)
10. **Label Resolver** resolves or creates each label (parent-first)
11. **Restore Context** re-attaches classification context
12. **Gmail1** applies labels (classification + Processed) to the email
13. **Handle Error** catches AI agent errors — applies "Needs Review" fallback
14. **Archive Router** checks mainLabel — archives Promotional/Notifications; keeps others in inbox
15. **Archive Email** removes from inbox

## Classification Taxonomy (Actual)

| MAIN_LABEL | Sub-labels | Label Name Pattern |
|------------|-----------|-------------------|
| Inbox Management | To Respond, FYI, Tasks, Comment | `Inbox Management/FYI` |
| Notifications | System | `Notifications/System` |
| Calendar | Meeting Update | `Calendar/Meeting Update` |
| Promotional | Marketing, Newsletters | `Promotional/Marketing` |
| Finance | Invoices, Payments | `Finance/Invoices` |
| IT Hosting | Infrastructure | `IT Hosting/Infrastructure` |
| Personal | Family, Health, Education | `Personal/Family` |
| Legal | Contracts, Policy | `Legal/Contracts` |

## Dependencies

- **Credentials:**
  - `gmailOAuth2` — Primary Email Account [neo7moruri]
  - `ollamaApi` — Ollama Models (evo:11434)
- **Model:** `hhao/qwen2.5-coder-tools:14b` on evo
- **Vector DB:** Qdrant (evo:6333), collection `email-labels`, `nomic-embed-text` embeddings
- **Sub-workflow:** Label Resolver (`ZbxbmVOmibKzD3Wz`)

## Related

- Runbook: `../../runbooks/week-01/gmail-manager.md`
- Spec: `../week-01/label-resolver.md`
