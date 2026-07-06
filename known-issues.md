# Known Issues — n8n Email Classification Workflows

Each issue has a short code (`CR-n` for Critical, `HI-n` for High, `ME-n` for Medium, `LO-n` for Low) for easy cross-referencing.

---

## CR-1: Label Resolver — Field Name Mismatch (FIXED 2026-06-29)

**Workflow:** Label Resolver (`ZbxbmVOmibKzD3Wz`)
**Node:** `Resolve Main` (Code, position `[672, 192]`)
**Severity:** Critical — sub-workflow receives `undefined` for both label fields

**Status:** **FIXED** — `Resolve Main` now reads `labelName` and `mainLabelName` (matching caller field names) instead of `label_name` and `mainLabel`.

---

## CR-2: `operation: "getAll"` Missing (Two Nodes) (FIXED 2026-06-29)

**Workflow:** Label Resolver (`ZbxbmVOmibKzD3Wz`) + Backfill Email Classifier (`9MYs2bhr3OYLlJq8`) + Gmail Manager (`Io4HrrgLoqdMVg2B`)
**Nodes:** `Get All Labels` / `Get Labels` (all three workflows)
**Severity:** Critical — Gmail returns nothing, resolution finds no existing labels

**Status:** **FIXED** — `"operation": "getAll"` added to all three Gmail label-get nodes.

---

## HI-1: No Try/Catch on JSON Parse (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Parse JSON` (Code, position `[2496, 64]`, formerly `Parse JSON & Extract Labels`)
**Severity:** High — bad AI output crashes the workflow

**Status:** **FIXED** — `try/catch` now returns proper fallback values (`Needs Review`) instead of referencing the undefined `parsed` variable in the catch block. Node also renamed to `Parse JSON` (extraction split into new `Extract Labels` node).

---

## HI-2: `numPredict: 100` Truncates LLM Output (FIXED)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Ollama Chat Model` (position `[1872, 176]`)
**Severity:** High — truncated JSON combines with no try/catch for guaranteed crashes

**Status:** **FIXED** — `numPredict` is now `1000` (matching Gmail Manager's setting).

---

## ME-1: Transform Returns Flat Object (No `{json:}` Wrapper) (FIXED)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Transform` (Code, position `[1344, -48]`)
**Severity:** Medium — downstream `$json.*` references may resolve to `undefined`

**Status:** **FIXED** — Now properly wraps output in `{ json: { ... } }`.

---

## ME-2: Done Code Broken for Create Processed Path (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Done` (Code, position `[1344, -288]`)
**Severity:** Medium — `processedLabelId` is `undefined` when Processed label needs creation

**Status:** **FIXED** — Now correctly falls back to `$input.first().json.id` when `processedLabelId` is not present.

---

## ME-3: Hardcoded `Label_42` in Add to Processed (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Add to Processed` (Gmail, position `[3392, 64]`)
**Severity:** Medium — fragile across accounts / label deletions

**Status:** **FIXED** — Now uses `={{ $json.processedLabelId }}` expression instead of literal `["Label_42"]`.

---

## ME-4: Archive? Uses `strict` Validation + Backward Reference (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Archive?` (IF, position `[3616, 64]`)
**Severity:** Medium — may not archive correctly

**Status:** **FIXED** — Changed to forward reference (`$json.mainLabel`) with `typeValidation: "loose"`.

---

## LO-1: Inline Processed Resolution (REMOVED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Nodes:** `Get Labels` → `Find Processed` → `Exists?` → `Create Processed` / `Done`
**Severity:** Low — one-time setup; Processed label already exists

**Status:** **RESOLVED** — The original 5-node chain (with Create path + Exists? check) was replaced by a lean 2-node parallel branch:
- **Get Labels** (Gmail getAll labels) — parallel to Find Unprocessed
- **Find Processed Label** (Code) — extracts `processedLabelId` from label list
These feed into the Accumulator via a second input connection, ensuring the Accumulator waits for both branches. The Accumulator no longer uses `$credentials` (not available in Code nodes). Count: 26→28 nodes.

---

## LO-2: Single-Label Only (No Multi-Label Support) (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Status:** **FIXED** — New `Extract Labels` Code node splits `proposedLabels[]` array into individual items. `Parse JSON & Extract Labels` renamed to `Parse JSON` (parsing only). `Extract Labels` sits between `Parse JSON` and `Resolve Label`, creating one item per proposed label.

---

## LO-3: Documentation Out of Sync

**Status:** **FIXED 2026-06-29** — All specs were updated to reflect the actual workflow configurations:
- Gmail Manager: taxonomy corrected, RAG node documented, archive rules fixed, node count (19→20), model name fixed
- Backfill: node count corrected (28→25), trigger type corrected, model name fixed, Extract Labels splitter added for multi-label support
- Label Resolver: input/output fields corrected

---

## CR-3: SplitInBatches `reset: false` Corrupts Loop State on Partial Failure (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Loop Over Items` (SplitInBatches, position `[1408, -288]`)
**Severity:** Critical — permanently stuck at 0 items after any partial failure

**Observation:** Execution 681 (Jun 24, 19:19) ran for 20 minutes, successfully processing 17 loop nodes before failing on malformed JSON at `Parse JSON`. Execution 800 (Jun 29, 18:20) then failed instantly with `"Unknown error"` and `itemCount: 0` — all downstream nodes skipped. Execution 801 (same time) reproduced identically.

**Root cause:** `SplitInBatches` with `reset: false` persists loop state (batches remaining, current batch position) in the workflow's database. When execution 681 failed partway through, its incomplete state was stored. Subsequent executions restored this corrupted state, causing SplitInBatches to believe it had 0 items to process — even though the upstream nodes produced 100 items.

**Status:** **FIXED** — `reset: true` added to SplitInBatches configuration. On the fixed architecture (Accumulator pre-loop), each manual trigger execution starts fresh with full pagination, making state persistence unnecessary.

---

## CR-4: AI Agent Prompt — Duplicated Content + Broken Placeholders + Missing Taxonomy (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `AI Agent: First Try` (system prompt)
**Severity:** Critical — AI agent receives corrupted instructions

**Issues:**
1. **Duplication:** The full instruction block (labeling rules, roll precedent, output format) appeared twice verbatim
2. **Broken placeholders:** `[...existing input fields...]` remained from template copy-paste
3. **Missing taxonomy:** The 8-label hierarchical taxonomy (Action Required, Notifications, etc.) was entirely absent — the AI agent had no label definitions
4. **Missing RAG injection:** `ragExamples` variable was defined but never referenced in the prompt

**Status:** **FIXED** — Single clean prompt with 8-label taxonomy, 3 concrete examples, RAG examples section (`{{ $json.ragExamples }}`), input field mapping, output rules, and JSON format requirements. No duplication, no broken placeholders.

---

## CR-5: Parse JSON Syntax Error — Extra Closing Brace Before `catch` (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Node:** `Parse JSON` (Code)
**Severity:** Critical — syntax error prevents JSON parsing entirely

**Issue:** Code contained `}\n} catch (error)` — an extra `}` before `catch`, producing a syntax error. This was introduced in the HI-1 fix (LO-2 refactor) when `Parse JSON & Extract Labels` was split into `Parse JSON` and `Extract Labels`, and the refactored catch block was incorrectly closed before the catch clause.

**Status:** **FIXED** — Removed the extra `}` so catch properly follows the try block.

---

## CR-6: Accumulator Replaces Fragile Pre-Loop Chain (FIXED 2026-06-29)

**Workflow:** Backfill Email Classifier (`9MYs2bhr3OYLlJq8`)
**Nodes:** `Enrich Emails` → `Loop Over Items` → `Find Unprocessed`
**Severity:** Critical — Gmail search filter `-label:Processed` can lag behind label application

**Issues:**
1. **Gmail search index lag:** `in:inbox -label:Processed` relies on Gmail's search index, which may not be immediately consistent after label application
2. **No pagination:** Single `getAll` call could miss emails when inbox exceeds first page
3. **Non-deterministic batches:** No guarantee of exactly N unprocessed emails per trigger

**Fix:** Three coordinated changes:
1. **`Find Unprocessed` query changed** from `in:inbox -label:Processed` to `in:inbox` (removed label filter — now fetches raw inbox messages)
2. **New `Accumulator` Code node** (replaces `Enrich Emails`) — paginates Gmail API via `$credentials.gmailOAuth2`, manually filters `labelIds` for "Processed", accumulates up to 100 unprocessed, attaches `processedLabelId`
3. **`Enrich Emails` removed** — its merge-Purpose is subsumed by the Accumulator

**Status:** **FIXED** — All changes applied and connections rewired (`Find Unprocessed → Accumulator → Loop Over Items`). The inline Processed resolution chain (Get Labels → Find Processed → Exists? → Create Processed / Done) was also removed since the Processed label is a permanent fixture; the Accumulator now resolves the label ID internally via Gmail API.
