# Changelog

## [Unreleased]

### Changed (2026-06-29 — Wait removal + sticky notes + Add to Processed fix)

- **CR-7**: Simplified loop architecture
  - Removed `Wait` node (3s delay had no purpose) — Loop Over Items now feeds directly to Get Full Message
  - Loop Over Items batch size: 25 (was 100), `reset: true`
  - Added 4 sticky notes as visual segment headers: Setup & Label Resolution (blue), Fetch & Transform (green), AI Classification & Apply Labels (orange), Qdrant Storage & Completion (purple)
  - Node count: 28→31 (removed 1, added 4 sticky notes)
- **CR-8**: Fixed `Add to Processed` — `messageId` now reads `$json.metadata.messageId` (was `$json.id` which didn't exist at that point), `labelIds` uses hardcoded `"Processed"` (was `$json.processedLabelId` which was dropped by Format for Qdrant)
- **CR-9**: Fixed `Archive Email` — `messageId` reads `$json.metadata.messageId || $json.id` to handle both pass-through and Gmail API response data shapes

### Fixed (2026-06-29 — Accumulator architecture + RAG integration + Prompt fixes)

- **CR-3**: SplitInBatches `reset: false` → `reset: true` — corrupted state from partial failures (exec 681) caused permanent `itemCount: 0` on subsequent executions (exec 800/801)
- **CR-4**: AI Agent system prompt — removed duplicated content, replaced broken `[...existing input fields...]` placeholder, added full 8-label taxonomy, added RAG examples section via `{{ $json.ragExamples }}`, added 3 concrete classification examples
- **CR-5**: Parse JSON Code node — removed stray extra `}` before `catch(error)` that caused syntax error (regression from LO-2 refactor)
- **CR-6**: Accumulator architecture — simplified after discovering `$credentials` is unavailable in Code nodes
  - `Find Unprocessed` query reverted to `in:inbox -label:Processed` (Gmail search handles filtering)
  - Accumulator simplified: no `$credentials`, no API calls, no pagination — just merges `processedLabelId` from dual input
  - Added parallel label resolution branch: Get Labels → Find Processed Label (Code) extracts label ID
  - Accumulator receives dual input (emails + processedLabelId), waits for both branches
  - `Enrich Emails` removed. Node count: 26→28 (removed 5, added 2).
- **HI-1**: Backfill `Parse JSON` (formerly `Parse JSON & Extract Labels`) catch block fixed — no longer references undefined `parsed`; added missing return statement after try/catch so items pass through on successful parse
- **ME-3**: Backfill `Add to Processed` uses `$json.processedLabelId` expression instead of hardcoded `Label_42`

### Added (Week 4)
- **RAG integration** in Backfill Email Classifier — 6 new nodes:
  - `Qdrant Vector Store` (retrieval) — searches `email-labels` collection with `search_query:` prefixed embeddings
  - `Embeddings Ollama` — nomic-embed-text:latest for both retrieval + document insertion
  - `Format RAG Examples` — formats Qdrant results into `ragExamples` text block for AI Agent few-shot
  - `Format for Qdrant` — creates `pageContent` with `search_document:` prefix + metadata for vector storage
  - `Qdrant Vector Store (insert)` — stores classified email + label as training vector
  - `Default Data Loader` — required document loader for insert-mode vector store
- **Parallel label resolution**: Get Labels + Find Processed Label runs alongside main pipeline, feeds `processedLabelId` into Accumulator via dual input
- **SplitInBatches `reset: true`**: Prevents state corruption from partial failures

### Added

- Initial repo scaffold: specs/ and runbooks/ directories with week-01 templates
- Gmail Manager workflow documentation (spec + runbook)
- Record Active Labels workflow documentation (spec + runbook)
- Delete Orphan Labels workflow documentation (spec + runbook)

### Changed (Week 2 — Hierarchical Multi-Label Design)

- **Gmail Manager**: Rewrote spec + runbook for 19-node architecture with hierarchical multi-label classification (8 MAIN_LABELs), inline Processed label resolution, Label Resolver sub-workflow integration, Extract Labels → Restore Context pipeline, archive routing per-label, and Parse JSON error fallback
- **Delete Orphan Labels**: Updated protected label list to 8 MAIN_LABELs pattern

### Added (Week 2)

- **Label Resolver** sub-workflow spec + runbook — parent-first label creation for Gmail `/` nesting
- **Backfill Email Classifier** spec + runbook — 28-node batch processor (100 emails/run, 30-min schedule)
- **Daily Email Digest** spec + runbook — 4-node Discord digest workflow (8 AM daily)

### Added (Week 3 — SCSM Integration)

- **SCSM Daily Completed** workflow spec + runbook — 4-node pipeline: Manual Trigger → SCSM Query (MS SQL, CiresonAnalytics) → Transform Results (Code) → Power Automate Webhook. Queries completed Service Requests + resolved Incidents for Potch Service Desks today, transforms to `{ID, User, Tech, DateResolved}`, POSTs to Power Automate
- **Power Automate Plan** spec — end-to-end plan for the downstream flow: HTTP trigger → Parse JSON → Apply to Each (Upsert to tracking table) → Send email notification
