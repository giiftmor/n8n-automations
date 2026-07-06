# RAG-Enhanced Email Classification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy a Qdrant vector database on evo, ingest historically classified emails as embeddings, and augment the Backfill Email Classifier (and optionally Gmail Manager) with RAG-based few-shot example retrieval to improve classification accuracy.

**Architecture:** Qdrant runs as a single Docker container on evo (port 6333). An n8n ingestion workflow fetches already-classified emails from Gmail, embeds them via `nomic-embed-text` through the Embeddings Ollama node, and stores vectors + payload in a Qdrant collection. The Backfill Email Classifier then adds a retrieval step: before calling the AI Agent, the email text is embedded and used to query Qdrant for top-5 similar classified emails; the retrieved examples are injected as few-shot context into the AI Agent's system prompt.

**Tech Stack:** Qdrant (Docker), n8n (Qdrant Vector Store + Embeddings Ollama nodes), Ollama (nomic-embed-text on evo), Gmail API

## Global Constraints

- Model runs on evo (`100.85.216.53`) via Ollama
- n8n ChatOllama LangChain integration does **not** support tool/function calling
- n8n Code node modes: `runOnceForEachItem` returns single `{ json }` object or `null`; `runOnceForAllItems` returns array `[{ json }, ...]` or `[]`
- Qdrant collection `email-labels` with 768-dimensional vectors (nomic-embed-text), cosine distance
- `numPredict: 100` on backfill's Ollama model — classify prompt must fit in ~100 output tokens
- `nomic-embed-text` model already installed on evo Ollama (274MB)
- All new n8n workflows use `typeVersion: 1.3` for Qdrant Vector Store node, `typeVersion: 1` for Embeddings Ollama

---

### Task 1: Deploy Qdrant on evo

**Files:**
- Create: `/home/evo/docker/qdrant/docker-compose.yml`
- Create: `/home/evo/docker/qdrant/storage/` (directory, created by Docker volume mount)

**Interfaces:**
- Consumes: evo SSH access, Docker Compose v5.1.1
- Produces: Qdrant HTTP API at `http://evo:6333`, Qdrant gRPC at `evo:6334`

- [ ] **Step 1: Create the Docker Compose file**

```bash
ssh evo@evo "mkdir -p /home/evo/docker/qdrant"
```

Then write `/home/evo/docker/qdrant/docker-compose.yml`:

```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - ./storage:/qdrant/storage
    restart: unless-stopped
```

- [ ] **Step 2: Start Qdrant**

```bash
ssh evo@evo "docker compose -f /home/evo/docker/qdrant/docker-compose.yml up -d"
```

Expected:
```
[+] Running 2/2
 ✔ Network qdrant_default  Created
 ✔ Container qdrant-qdrant-1  Started
```

- [ ] **Step 3: Verify Qdrant is running**

```bash
ssh evo@evo "curl -s http://localhost:6333/ | jq ."
```

Expected output:
```json
{
  "title": "qdrant - vector search",
  "version": "1.13.x",
  "commit": "..."
}
```

- [ ] **Step 4: Create the collection with correct vector config**

```bash
ssh evo@evo "curl -s -X PUT http://localhost:6333/collections/email-labels \
  -H 'Content-Type: application/json' \
  -d '{
    \"vectors\": {
      \"size\": 768,
      \"distance\": \"Cosine\"
    }
  }' | jq ."
```

Expected output:
```json
{
  "result": true,
  "status": "ok",
  "time": 0.001
}
```

---

### Task 2: Create Qdrant and Ollama Credentials in n8n

**Files:** (n8n internal — no files)

**Interfaces:**
- Consumes: n8n API access
- Produces: Qdrant credential ID, Ollama credential ID (referenced by workflows in Tasks 3-5)

- [ ] **Step 1: Create Qdrant credential in n8n**

```bash
N8N_URL="https://evo:5678"
API_KEY="[from env]"

curl -s -X POST "$N8N_URL/api/v1/credentials" \
  -H "X-N8N-API-KEY: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Qdrant - evo",
    "type": "qdrantApi",
    "data": {
      "qdrantUrl": "http://evo:6333"
    }
  }' | jq '{id: .id, name: .name}'
```

Note: No API key needed — Qdrant runs on evo's internal Docker network without auth. If n8n is running on evo too, use `http://qdrant-qdrant-1:6333` instead of `http://evo:6333` (Docker DNS). Verify by checking which network n8n is on:

```bash
ssh evo@evo "docker inspect n8n --format '{{json .NetworkSettings.Networks}}' | jq '. | keys'"
```

If n8n and qdrant are on the same Docker network, use container name `qdrant-qdrant-1` as host. If not, use `evo:6333` (mapped port).

- [ ] **Step 2: Create Ollama credential in n8n (if not existing)**

Check if an Ollama credential already exists:
```bash
curl -s -X GET "$N8N_URL/api/v1/credentials" \
  -H "X-N8N-API-KEY: $API_KEY" | jq '.data[] | select(.type == "ollamaApi") | {id: .id, name: .name}'
```

If none exists, create one:
```bash
curl -s -X POST "$N8N_URL/api/v1/credentials" \
  -H "X-N8N-API-KEY: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Ollama - evo",
    "type": "ollamaApi",
    "data": {
      "baseUrl": "http://evo:11434"
    }
  }' | jq '{id: .id, name: .name}'
```

---

### Task 3: Build Email Embedding Ingestion Workflow

**Files:** (n8n workflow JSON — created via API)

**n8n Workflow ID**: (assigned on creation)

**Interfaces:**
- Consumes: Gmail credentials, Qdrant credential (Task 2), Ollama credential (Task 2)
- Produces: Qdrant collection `email-labels` populated with classified email embeddings

**Node architecture:**

```
Manual Trigger
  │
  ├→ Get All Labels (Gmail)
  │    resource: label, operation: getAll
  │    output: array of {id, name, messagesTotal, type}
  │
  ├→ Code Node (Filter For Hierarchical)
  │    Mode: runOnceForAllItems
  │    Code:
  │      const items = $input.all();
  │      return items.filter(item => 
  │        item.json.name && 
  │        item.json.name.includes('/') && 
  │        item.json.messagesTotal > 0 &&
  │        !item.json.name.startsWith('CATEGORY_')
  │      ).map(item => ({ json: item.json }));
  │
  ├→ SplitInBatches (Batch Labels) — batch size: 3
  │
  │  ├→ Search Gmail (by label)
  │  │    resource: message, operation: get
  │  │    query: "label:{{ $json.name }} after:2026/01/01"
  │  │    simple: true
  │  │    output: array of messages
  │  │
  │  ├→ SplitInBatches (Batch Messages) — batch size: 5
  │  │
  │  │  ├→ Get Message (Gmail)
  │  │  │    resource: message, operation: get
  │  │  │    messageId: {{ $json.id }}
  │  │  │    format: full
  │  │  │    simple: true
  │  │  │
  │  │  ├→ Code Node (Format For Embedding)
  │  │  │    Mode: runOnceForEachItem
  │  │  │    Code:
  │  │  │      const email = $input.first().json;
  │  │  │      const labelName = $("<node>Batch Labels").first().json.name;
  │  │  │      const mainLabel = labelName.split('/')[0];
  │  │  │      return {
  │  │  │        pageContent: `Subject: ${email.Subject || ""}\nFrom: ${email.From || ""}\nDate: ${email.Date || ""}\n\n${(email.textPlain || email.snippet || "").substring(0, 2000)}`,
  │  │  │        metadata: {
  │  │  │          labelName: labelName,
  │  │  │          mainLabel: mainLabel,
  │  │  │          subject: email.Subject || "",
  │  │  │          fromAddress: email.From || "",
  │  │  │          date: email.Date || "",
  │  │  │          messageId: email.Id || ""
  │  │  │        }
  │  │  │      };
  │  │  │
  │  │  └→ Qdrant Vector Store (Insert)
  │  │       credential: Qdrant - evo
  │  │       mode: insertDocuments
  │  │       qdrantCollection: email-labels
  │  │       Options:
  │  │         contentPayloadKey: pageContent
  │  │         metadataPayloadKey: metadata
  │  │       AI Embedding sub-node:
  │  │         → Embeddings Ollama
  │  │            credential: Ollama - evo
  │  │            model: nomic-embed-text
```

- [ ] **Step 1: Create the workflow via n8n API**

Build the complete workflow JSON with the architecture above. Validate with:
```bash
# Use the n8n REST tool to validate the workflow JSON before creating
n8n-rest_n8n_validate_workflow(workflow={...})
```

- [ ] **Step 2: Create the workflow in n8n**

```bash
# Use n8n-rest_n8n_create_workflow
# name: "Ingest Classified Emails to Qdrant"
```

- [ ] **Step 3: Test with a single label**

Trigger the workflow manually. Verify in the execution:
- Gmail search returns emails for a test label
- Format Code node produces `{pageContent, metadata}` objects
- Qdrant insert succeeds (check Qdrant API: `curl http://evo:6333/collections/email-labels/points/count`)

- [ ] **Step 4: Verify embeddings stored in Qdrant**

```bash
ssh evo@evo "curl -s http://localhost:6333/collections/email-labels/points/count | jq ."
```

Expected: `{"result":{"count":N},"status":"ok"}` where N > 0.

- [ ] **Step 5: Set the workflow to run daily**

In n8n UI: replace Manual Trigger with Schedule Trigger, run at 2:00 AM daily.

---

### Task 4: Modify Backfill Email Classifier with RAG Retrieval

**Files:** (n8n workflow modification — existing workflow `9MYs2bhr3OYLlLJq8`)

**Interfaces:**
- Consumes: Backfill workflow `9MYs2bhr3OYLlLJq8`, Qdrant credential, Ollama credential
- Produces: RAG-enhanced Backfill workflow

**Current Backfill flow (paraphrased):**
```
Gmail Search → Loop → Get Message → Transform → Parse JSON → AI Agent → Label Resolver → Gmail Add Label
```

**Modified Backfill flow with RAG:**
```
Gmail Search → Loop → Get Message → Transform → Inject RAG Context [NEW] → AI Agent → Label Resolver → Gmail Add Label
```

**Inject RAG Context sub-chain:**

```
Transform → Qdrant Retrieve + Embeddings → Format RAG Examples (Code) → AI Agent
                   ↓
           (connected sub-node: Embeddings Ollama)
```

- [ ] **Step 1: Get the Backfill workflow structure**

```bash
n8n-rest_n8n_get_workflow(id="9MYs2bhr3OYLlLJq8", mode="structure")
```

- [ ] **Step 2: Add Qdrant Vector Store + Embeddings Ollama after Transform**

Using `n8n-rest_n8n_update_partial_workflow` with operations:

```javascript
// Operation: addNode for Qdrant Vector Store
{
  type: "addNode",
  node: {
    id: "qdrant-retrieve",
    name: "Qdrant Retrieve",
    type: "@n8n/n8n-nodes-langchain.vectorStoreQdrant",
    typeVersion: 1.3,
    position: [X, Y],  // after Transform node
    parameters: {
      mode: "load",
      qdrantCollection: { mode: "id", value: "email-labels" },
      prompt: "={{ $('Transform').item.json.Subject + ' ' + ($('Transform').item.json.textPlain || $('Transform').item.json.snippet || '') }}"
    },
    credentials: {
      qdrantApi: { id: "[qdrant-credential-id]", name: "Qdrant - evo" }
    },
    options: {
      topK: 5
    }
  }
}

// Operation: addNode for Embeddings Ollama (sub-node)
{
  type: "addNode",
  node: {
    id: "embeddings-ollama",
    name: "Embeddings Ollama",
    type: "@n8n/n8n-nodes-langchain.embeddingsOllama",
    typeVersion: 1,
    position: [X, Y],
    parameters: {
      model: "nomic-embed-text"
    },
    credentials: {
      ollamaApi: { id: "[ollama-credential-id]", name: "Ollama - evo" }
    }
  }
}

// Operation: addConnection from Qdrant Retrieve to Embeddings Ollama
{
  type: "addConnection",
  source: {
    node: "Qdrant Retrieve",
    outputIndex: "ai_embedding"
  },
  destination: {
    node: "Embeddings Ollama",
    inputIndex: "ai_embedding"
  }
}

// Operation: addNode for Format RAG Examples (Code)
{
  type: "addNode",
  node: {
    id: "format-rag",
    name: "Format RAG Examples",
    type: "n8n-nodes-base.code",
    typeVersion: 2,
    position: [X, Y],  // after Qdrant Retrieve
    parameters: {
      mode: "runOnceForEachItem",
      jsCode: `const qdrantResults = $input.all();
const ragExamples = qdrantResults
  .filter(item => item.json && item.json.metadata && item.json.metadata.labelName)
  .slice(0, 5)
  .map(item => {
    const meta = item.json.metadata;
    return "Example - " + meta.labelName + ":\n  Subject: " + meta.subject + "\n  From: " + meta.fromAddress + "\n  Date: " + meta.date;
  })
  .join("\n---\n");

const original = $json;

return {
  ragExamples: ragExamples || "No similar classified emails found.",
  ...original
};`
    }
  }
}

// Operation: addConnection from Qdrant Retrieve to Format RAG Examples
{
  type: "addConnection",
  source: {
    node: "Qdrant Retrieve",
    outputIndex: "main"
  },
  destination: {
    node: "Format RAG Examples",
    inputIndex: "main"
  }
}

// Operation: addConnection from Transform (source) to Format RAG Examples (destination)
// This ensures the original email data flows through even if Qdrant returns 0 results
{
  type: "addConnection",
  source: {
    node: "Transform",
    outputIndex: "main"
  },
  destination: {
    node: "Format RAG Examples",
    inputIndex: "main"  // secondary input — Code node merges from $input.all()
  }
}

// Operation: removeConnection from Transform to AI Agent (old edge)
{
  type: "removeConnection",
  source: {
    node: "Transform",
    outputIndex: "main"
  },
  destination: {
    node: "AI Agent",
    inputIndex: "main"
  }
}

// Operation: addConnection from Format RAG Examples to AI Agent
{
  type: "addConnection",
  source: {
    node: "Format RAG Examples",
    outputIndex: "main"
  },
  destination: {
    node: "AI Agent",
    inputIndex: "main"
  }
}
```

- [ ] **Step 3: Update AI Agent system prompt to include RAG examples**

Using `n8n-rest_n8n_update_partial_workflow`:
```javascript
{
  type: "updateNode",
  node: {
    name: "AI Agent"
  },
  // patch the prompt field to append RAG examples context
  parameters: {
    nodeType: "patchNodeField",
    fieldPath: "parameters.options.systemMessage",
    patches: [
      {
        find: '"### Classification Rules"',
        replace: '"{{ $json.ragExamples }}\\n\\n### Classification Rules"'
      }
    ]
  }
}
```

Actually, simpler: use `patchNodeField` to find the end of the prompt and append:
```
Here are the most similar previously classified emails as examples:

{{ $json.ragExamples }}

---

```
Just before `### Classification Rules`.

- [ ] **Step 4: Validate the modified workflow**

```bash
n8n-rest_n8n_validate_workflow(id="9MYs2bhr3OYLlLJq8")
```

Expected: 0 errors.

- [ ] **Step 5: Test with a manual execution**

Trigger the Backfill workflow manually. In the execution view:
- Verify Qdrant Retrieve node returns results (should show 0-5 items with `pageContent` and `metadata`)
- Verify Format RAG Examples produces `ragExamples` string
- Verify AI Agent receives `ragExamples` in `$json.ragExamples`
- Verify no increase in error rate from pre-RAG baseline

---

### Task 5: (Optional) Modify Gmail Manager with RAG Retrieval

**Files:** (n8n workflow modification — existing workflow `Io4HrrgLoqdMVg2B`)

**Interfaces:**
- Consumes: Gmail Manager workflow `Io4HrrgLoqdMVg2B`, Qdrant credential, Ollama credential
- Produces: RAG-enhanced Gmail Manager workflow

**Design:** Same pattern as Task 4, but applied to the live polling workflow instead of the batch backfill.

- [ ] **Step 1: Get the Gmail Manager workflow structure**

```bash
n8n-rest_n8n_get_workflow(id="Io4HrrgLoqdMVg2B", mode="structure")
```

- [ ] **Step 2: Apply same RAG additions as Task 4**

Follow the exact same addNode/addConnection pattern from Task 4, Steps 2-3, but inserting the RAG chain after the Transform node (or equivalent) in Gmail Manager.

:warning: **Note:** Gmail Manager runs on each incoming email individually (not in batches). The RAG retrieval must complete quickly (target < 2s) to avoid delaying the trigger loop. If latency is an issue, skip this task and rely on Backfill-only RAG.

- [ ] **Step 3: Validate**

```bash
n8n-rest_n8n_validate_workflow(id="Io4HrrgLoqdMVg2B")
```

- [ ] **Step 4: Test with a triggering email**

Send a test email to the monitored inbox. Verify in n8n execution log that Qdrant retrieval fires and examples appear in the AI Agent context.

---

## Self-Review

**1. Spec coverage:**
- Qdrant deployment on evo → Task 1 ✓
- Qdrant + Ollama credentials in n8n → Task 2 ✓
- Ingestion workflow for existing classified emails → Task 3 ✓
- RAG retrieval in Backfill → Task 4 ✓
- RAG retrieval in Gmail Manager → Task 5 ✓
- nomic-embed-text embedding model → Task 3/4 Embeddings Ollama config ✓
- 768d cosine collection → Task 1 Step 4 ✓
- Top 5 similar examples → Task 4 topK: 5 ✓
- Injection as few-shot context → Task 4 Step 3 ✓

**2. Placeholder scan:** No TBD, TODOs, or implementation-by-reference found.

**3. Type consistency:**
- Credential names consistent across Tasks 2→3→4→5
- `typeVersion` values correct: Qdrant 1.3, Embeddings Ollama 1, Code 2
- Collection name `email-labels` consistent everywhere
- Vector size 768 matches nomic-embed-text ✓
- Node IDs referenced correctly across operations ✓
