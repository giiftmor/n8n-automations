# Backfill Email Classifier

## Overview

Batch-processes existing unprocessed inbox emails through the same AI classification pipeline as Gmail Manager. Runs manually (no schedule), processes 100 emails per run in batches of 25, and applies hierarchical multi-label classification (up to 2 labels + Processed label). Skips already-processed emails and handles errors with a secondary AI Agent for JSON repair.

## Trigger

| Property | Value |
|----------|-------|
| Type | Manual (no schedule trigger) |
| Batch Size | 100 emails per run (25 per iteration) |

Note: This workflow has no schedule trigger. It must be triggered manually from the n8n editor.

## Nodes (27 functional + 4 sticky notes = 31)

### 1. Manual Trigger
- **Type:** `n8n-nodes-base.manualTrigger`
- **Purpose:** Starts execution on demand

### 2. Find Unprocessed
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `getAll` (message)
- **Query:** `in:inbox -label:Processed`
- **Limit:** 100
- **Purpose:** Fetches up to 100 unprocessed inbox messages (Gmail search handles the `-label:Processed` filter natively)

### 3. Get Labels
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `getAll` (label)
- **Return All:** `true`
- **Purpose:** Fetches all Gmail labels for Processed label ID resolution (parallel branch from Manual Trigger)

### 4. Find Processed Label (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** Run Once for All Items (default)
- **Logic:** Finds label with `name === 'Processed'`, extracts its Gmail ID, outputs `{processedLabelId}`
- **Error:** Throws if Processed label is missing

### 5. Accumulator (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** Run Once for All Items (default)
- **Input:** Receives email items from Find Unprocessed + `{processedLabelId}` from Find Processed Label (dual connection — waits for both)
- **Logic:** Separates the two input types, merges `processedLabelId` into each email item
- **Output:** Emails with `processedLabelId` attached

### Segment A: Setup & Label Resolution *(Sticky Note)*
Nodes 1-5: Resolves Processed label ID, fetches unprocessed emails, merges processedLabelId into each email.

**Visual:** Blue sticky note above nodes 1-5.

### Segment B: Fetch & Transform *(Sticky Note)*
Nodes 6-8: Batch loops 25 items at a time, fetches full email via Gmail get, transforms fields for AI.

**Visual:** Green sticky note above nodes 6-8.

### 6. Loop Over Items (SplitInBatches)
- **Type:** `n8n-nodes-base.splitInBatches`
- **Batch Size:** 25
- **Options:** `reset: true`
- **Purpose:** Processes 25 emails per batch iteration, loops back for remaining batches

### 7. Get Full Message
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `get` (message)
- **Simple mode:** `true`
- **Input:** `{{ $json.id }}`
- **Purpose:** Retrieves full message content for AI classification (text body, headers, labels)

### 8. Transform (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Output:** `{json: {messageId, fromName, fromAddress, subject, text, labelIds, snippet, internalDate, threadId}}`
- **Purpose:** Maps Gmail simple-mode output to format expected by AI prompt

### Segment C: AI Classification & Apply Labels *(Sticky Note)*
Nodes 9-20: RAG retrieval → Two-pass AI classification → Parse → Resolve label name → Apply to Gmail.

**Visual:** Orange sticky note above nodes 9-20.

### 9. Already Processed? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `{{ $json.labelIds }} contains "Processed"`
- **TRUE:** → Loop Over Items (skip, loop back to next email)
- **FALSE:** → Qdrant Vector Store (continue classification)
- **Purpose:** Belt-and-suspenders check — skips any email that somehow has the Processed label already

### 10. Qdrant Vector Store (RAG Retrieval)
- **Type:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant`
- **Mode:** `load` (similarity search)
- **Collection:** `email-labels` (768-dim Cosine, on evo:100.85.216.53:6333)
- **Prompt:** `search_query: {{ $json.subject }} {{ $json.text }}`
- **Credentials:** Qdrant account (`4qd-api-key`)
- **Embeddings:** Embeddings Ollama (nomic-embed-text:latest)
- **Purpose:** Retrieves up to 4 previously classified emails similar to the current one

### 11. Embeddings Ollama
- **Type:** `@n8n/n8n-nodes-langchain.embeddingsOllama`
- **Model:** `nomic-embed-text:latest`
- **Connection:** `ai_embedding` → Qdrant Vector Store + Qdrant Vector Store (insert)
- **Purpose:** Generates 768-dim embeddings for both retrieval queries and stored documents

### 12. Format RAG Examples (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** Run Once for All Items
- **Logic:** Takes Qdrant retrieval results, formats them as `Example N: labelName\n  Subject: "..."\n  From: ...\n  Date: ...`, merges with original email data, outputs `ragExamples` for AI Agent
- **Fallback:** `"No similar classified emails found."` when Qdrant is empty

### 13. AI Agent: First Try
- **Type:** `@n8n/n8n-nodes-langchain.agent`
- **Version:** 1.9
- **Model:** Ollama Chat Model (`hhao/qwen2.5-coder-tools:14b`) on evo:11434
- **Options:** `format: json`, `temperature: 0.1`, `numPredict: 1000`
- **System prompt:** Deduplicated prompt with full 8-label taxonomy, RAG examples section (`{{ $json.ragExamples }}`), input field mapping, output rules, 3 concrete examples, JSON format requirements
- **Purpose:** Primary classification — assigns up to 2 hierarchical labels per email

### 14. Ollama Chat Model
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOllama`
- **Model:** `hhao/qwen2.5-coder-tools:14b`
- **Connection:** `ai_languageModel` → AI Agent: First Try

### 15. AI Agent: Error Fixes
- **Type:** `@n8n/n8n-nodes-langchain.agent`
- **Version:** 3.1
- **Purpose:** Connected to First Try's error output — validates and repairs malformed JSON from the primary agent
- **System prompt:** JSON validation/repair instructions

### 16. Ollama Chat Model1
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOllama`
- **Model:** `hhao/qwen2.5-coder-tools:14b`
- **Connection:** `ai_languageModel` → AI Agent: Error Fixes

### 17. Parse JSON (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Parses `$json.output` via `JSON.parse()` and outputs `{proposedLabels[], messageId}`
- **Error handling:** `try/catch` — malformed JSON falls back to `{proposedLabels: [{mainLabel: "Needs Review", label_name: "Needs Review"}]}`

### 18. Extract Labels (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForAllItems`
- **Purpose:** Splits `proposedLabels[]` array into individual items, one per label. Creates `{labelName, mainLabelName, fallbackLabelId, messageId, mainLabel, subLabel}` per item. Falls back to `"Needs Review"` if array is empty.

### 19. Resolve Label (Execute Workflow)
- **Type:** `n8n-nodes-base.executeWorkflow`
- **Workflow:** Label Resolver (`ZbxbmVOmibKzD3Wz`)
- **Input:** `{labelName, fallbackLabelId, mainLabelName}`
- **Mode:** `each`
- **Purpose:** Calls sub-workflow to resolve/create each label (parent-first)

### 20. Restore Context (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Re-attaches `mainLabel`/`subLabel` after sub-workflow call

### 21. Apply Labels
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `addLabels`
- **Label IDs:** `{{ $json.labelId }}`
- **Purpose:** Applies the resolved classification label to the message

### Segment D: Qdrant Storage & Completion *(Sticky Note)*
Nodes 21-27: Store as RAG example → Mark as Processed → Archive if Promotional/Notifications → Loop back.

**Visual:** Purple sticky note above nodes 21-27.

### 22. Format for Qdrant (Code)
- **Type:** `n8n-nodes-base.code`
- **Mode:** `runOnceForEachItem`
- **Purpose:** Creates `pageContent` with `search_document:` prefix and `metadata` object (labelName, subject, from, date, messageId) for vector storage
- **References:** `$('Transform').item.json` for email content

### 23. Qdrant Vector Store (insert)
- **Type:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant`
- **Mode:** `insert`
- **Collection:** `email-labels`
- **Options:** `contentPayloadKey: "pageContent"`, `metadataPayloadKey: "metadata"`
- **Credentials:** Qdrant account (`4qd-api-key`)

### 24. Default Data Loader
- **Type:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`
- **Connection:** `ai_document` → Qdrant Vector Store (insert)

### 25. Add to Processed
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `addLabels`
- **Message ID:** `{{ $json.metadata.messageId }}`
- **Label IDs:** `"Processed"` (hardcoded label name)
- **Purpose:** Applies the Processed label to mark email as classified

### 26. Archive? (IF)
- **Type:** `n8n-nodes-base.if`
- **Condition:** `$json.mainLabel === "Promotional"` OR `$json.mainLabel === "Notifications"`
- **TRUE:** → Archive Email
- **FALSE:** → Loop Over Items (loop back)

### 27. Archive Email
- **Type:** `n8n-nodes-base.gmail`
- **Operation:** `removeLabel` (INBOX)
- **Message ID:** `{{ $json.metadata.messageId || $json.id }}` (works with either data shape)
- **Purpose:** Archives promotional and notification emails

## Data Flow

1. **Manual trigger** starts execution; forks into two parallel branches:
   - **Main pipeline:** Find Unprocessed → Accumulator
   - **Label resolution:** Get Labels → Find Processed Label → Accumulator
2. **Find Unprocessed** fetches unprocessed inbox emails (`in:inbox -label:Processed`, max 100)
3. **Get Labels** (parallel) fetches all Gmail labels; **Find Processed Label** extracts the Processed label ID
4. **Accumulator** waits for both branches (dual input), merges `processedLabelId` into each email, passes to loop
5. **Loop Over Items** (SplitInBatches, batchSize: 25, reset: true) processes 25 emails per batch
6. **Get Full Message** retrieves email content
7. **Transform** maps Gmail output to AI-friendly format
8. **Already Processed?** belt-and-suspenders check — skips if Processed label is present
9. **Qdrant Vector Store** (RAG) retrieves up to 4 similar classified emails
10. **Format RAG Examples** injects into AI Agent context
11. **AI Agent: First Try** classifies email
12. **AI Agent: Error Fixes** repairs malformed JSON
13. **Parse JSON** → **Extract Labels** → **Resolve Label** (sub-workflow) → **Restore Context**
14. **Apply Labels** applies classification label to message
15. **Format for Qdrant** + **Qdrant Vector Store (insert)** stores embedding for future RAG
16. **Add to Processed** applies the Processed label (referenced by name via `$json.metadata.messageId`)
17. **Archive?** routes Promotional/Notifications to archive
18. **Archive Email** removes INBOX label (archives) or continues directly
19. **Loop back** for next batch

## Classification Taxonomy

Same as [Gmail Manager](gmail-manager.md#classification-taxonomy-actual).

## Dependencies

- **Credentials:**
  - `gmailOAuth2` — Primary Email Account [neo7moruri]
  - `ollamaApi` — Ollama Models (evo:11434)
- **Model:** `hhao/qwen2.5-coder-tools:14b` on evo
- **Sub-workflow:** Label Resolver (`ZbxbmVOmibKzD3Wz`)

## Related

- Runbook: `../../runbooks/week-01/backfill-email-classifier.md`
- Spec: `../week-01/label-resolver.md`
