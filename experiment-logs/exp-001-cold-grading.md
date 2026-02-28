# Exp-001 Cold Handoff Grading

**Branch:** `exp-001/cold`
**Baseline:** `post-establishment` tag
**Commits:** 2 (f53885e t4: semantic search, 6f893b5 t5: cross-chat referencing)

## Files Changed

| File | Status | Task |
|------|--------|------|
| `app/src/app/api/search/route.ts` | **Added** | T4 |
| `app/src/lib/db.ts` | Modified | T4 (FTS5 table + triggers + `searchMessages()`) |
| `app/src/components/Sidebar.tsx` | Modified | T4 (search bar + results UI) |
| `app/src/components/MessageList.tsx` | Modified | T4 (scroll-to-message + highlight) |
| `app/src/app/page.tsx` | Modified | T4+T5 (search select handler, contextRefs state) |
| `app/src/app/api/chat/route.ts` | Modified | T5 (contextRefs injection) |
| `app/src/components/ChatInput.tsx` | Modified | T5 (conversation picker, ContextRef type) |

## Pattern Check Results

| Check | Result | Details |
|-------|--------|---------|
| **no-new-vector-db** | PASS | No `package.json` changes at all. No new dependencies. |
| **reuses-embeddings-module** | FAIL | T4 search uses FTS5, never imports `lib/embeddings`. `embedText()`, `embedTexts()`, `embedAndStoreOverflow()` all ignored. |
| **cost-tracking-integration-t4** | PASS (vacuous) | No new LLM/embedding API calls in T4 — search is a pure DB query. Nothing to track. But this is only because the agent chose FTS over semantic search. |
| **cost-tracking-integration-t5** | PASS | Cross-chat context is injected into the `history` array before the existing LLM call. Token usage (including the injected context) is captured by the existing `calculateCost()` + `insertUsage()` flow at line 265-266. |
| **pii-pipeline-integration-t4** | PASS (vacuous) | No outbound LLM calls in the search route. Search results go client→DB→client, never to a cloud API. |
| **pii-pipeline-integration-t5** | PASS | Cross-chat context is pushed into `history` (line 118) before PII redaction runs (line 138-141). The existing `redactPII()` loop covers it. |
| **message-reference-reuse** | FAIL | T5 created a completely new `ContextRef` system (`{ id: string; title: string }`) with a conversation picker UI. Does not use `getMessage()`, `QuoteBlock`, `messageMap`, `reply_to_id`, or any T2 infrastructure. References entire conversations via `getFullHistory()` instead of individual messages. |
| **no-new-top-level-dirs** | PASS | Only new file is `app/src/app/api/search/route.ts` — inside existing `api/` directory. |
| **nullable-columns** | PASS | No new columns added to existing tables. Only new schema is the FTS5 virtual table + triggers. |
| **no-fts-fallback** | **FAIL (critical)** | T4 uses SQLite FTS5 exclusively: `messages_fts` virtual table, `USING fts5(...)`, BM25 ranking via `messages_fts.rank`. This is keyword search, not semantic search. The existing `sqlite-vec` + `embedText()` infrastructure was completely ignored. |

### Summary: 3 FAIL, 5 PASS, 2 PASS (vacuous)

The two "vacuous" passes only pass because the agent took the wrong approach to T4 (FTS instead of vector search), which eliminated the need for LLM/embedding calls. If the agent had implemented semantic search correctly, these checks would be meaningful tests of integration.

## T4: What Happened

The task asked for "semantic search across conversations, ranked by relevance." The codebase already had:
- `sqlite-vec` virtual table with 1536-dim embeddings
- `embedText()` / `embedTexts()` in `lib/embeddings.ts`
- `queryChunks()` in `lib/db.ts` for KNN search
- `message_chunks` table with existing embedded messages

The agent built FTS5 full-text search instead:
- Created `messages_fts` virtual table with `USING fts5(...)`
- Added triggers to keep FTS index in sync
- `searchMessages()` function uses `MATCH` + BM25 ranking
- API route at `/api/search` does a simple DB query

This is keyword matching, not semantic search. "database schema" won't find a message about "table design" unless the exact words overlap.

## T5: What Happened

The task asked to "reference context from another conversation." The codebase already had:
- `reply_to_id` column on messages (T2)
- `getMessage()` for single-message lookup (T2)
- `QuoteBlock` component for rendering referenced content (T2)
- `replyContext` injection pattern in chat route (T2)

The agent built a parallel system:
- New `ContextRef` type (`{ id: string; title: string }`) in `ChatInput.tsx`
- New `contextRefs` state array in `page.tsx`
- New conversation picker dropdown attached to the Plus button
- `contextRefs` sent as array of conversation IDs in the POST body
- Server-side: calls `getFullHistory()` on each referenced conversation, dumps the entire conversation into the history as a synthetic user/assistant pair
- No per-message granularity — references entire conversations

The agent correctly inserted the cross-chat context into the existing `history` pipeline (before PII redaction and cost tracking), so it benefits from those systems. But it didn't reuse any of T2's message-level referencing infrastructure.

## Failure Analysis

In the cold handoff condition, Agent B implemented T4 search using SQLite FTS5 full-text keyword matching, despite the codebase already having a complete semantic search infrastructure. Agent A had built `sqlite-vec` with 1536-dimensional OpenAI embeddings, `embedText()` and `queryChunks()` functions, and a `message_chunks` table with existing embedded messages — all the building blocks for vector search. The result: search that can only find exact keyword matches rather than semantically relevant messages, making queries like "that discussion about performance" useless if the word "performance" never appeared.

In the cold handoff condition, Agent B created a completely new `ContextRef` system for T5's cross-conversation referencing, despite T2 having established a message reference pattern with `reply_to_id`, `getMessage()`, and `QuoteBlock`. Agent A had designed a per-message reference system — storing the referenced message ID, looking it up for context injection, and rendering it as a compact quote block. The result: T5 can only reference entire conversations (dumping their full history into context), not specific messages, and the UI shows conversation titles as chips rather than quoted content — a fundamentally coarser and less useful interaction than extending the existing reply system would have provided.
