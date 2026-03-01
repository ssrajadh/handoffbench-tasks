# Exp-001 A2 (agents.md) Grading

**Branch:** `exp-001/agents-md`
**Baseline:** `post-establishment` tag
**Commits:** 4 (aa762ad add agents.md, 48d0500 rename agents to claude, 26a5ae0 t4: semantic search, e8122f9 t5: cross-chat referencing)
**Context provided:** Full CLAUDE.md with sections 10-12 documenting cost tracking, message reply system, and PII guardrail (including integration rules)

## Files Changed

| File | Status | Task |
|------|--------|------|
| `CLAUDE.md` | Modified | Setup (replaced with detailed agents.md content) |
| `app/src/app/api/search/route.ts` | **Added** | T4 (hybrid semantic + text search) |
| `app/src/lib/db.ts` | Modified | T4 (`getAllConversationIds()`, `searchMessagesByText()`, `getConversationTitleMap()`) |
| `app/src/types/index.ts` | Modified | T4+T5 (`SearchResult`, `ContextRef` types) |
| `app/src/components/Sidebar.tsx` | Modified | T4 (search bar + results UI with semantic/text badges) |
| `app/src/components/MessageList.tsx` | Modified | T4 (highlight animation, `id` attributes on messages) |
| `app/src/app/globals.css` | Modified | T4 (search-highlight-fade animation keyframes) |
| `app/src/app/page.tsx` | Modified | T4+T5 (search select handler, scroll-to-message, contextRefs state) |
| `app/src/app/api/chat/route.ts` | Modified | T5 (cross-conv context injection with token budget) |
| `app/src/components/ChatInput.tsx` | Modified | T5 (conversation picker, ContextRef chips) |

## Pattern Check Results

| Check | Result | Details |
|-------|--------|---------|
| **no-new-vector-db** | PASS | No `package.json` changes. Reuses `sqlite-vec`. |
| **reuses-embeddings-module** | PASS | `search/route.ts` imports `embedText` and `ensureEmbeddingConfig` from `@/lib/embeddings`, and `queryChunks` from `@/lib/db`. |
| **cost-tracking-integration-t4** | FAIL | `search/route.ts` calls `embedText(query)` which hits the OpenAI embedding API, but does not call `calculateCost()` or `insertUsage()` to record the embedding cost. No reference to cost tracking anywhere in the file. |
| **cost-tracking-integration-t5** | PASS | Cross-conv context is injected into the `history` array before the existing LLM call. The `crossConvTokens` are deducted from the context window, and the total token usage (including injected context) is captured by the existing `calculateCost()` + `insertUsage()` flow after streaming. |
| **pii-pipeline-integration-t4** | FAIL | `search/route.ts` sends the raw user query directly to `embedText(query)` (OpenAI embedding API) without checking `pii_redaction_enabled` or calling `redactPII()`. Settings are fetched (line 28) but only to check for the API key. User query with potential PII is sent to OpenAI unredacted. |
| **pii-pipeline-integration-t5** | PASS | Cross-conv context is pushed into `history` (line 140-141) before PII redaction runs (line 166-168). The existing `redactPII()` loop covers all injected cross-chat content. |
| **message-reference-reuse** | FAIL | T5 created a new `ContextRef` system (`{ id: string; title: string }`) with a conversation picker. Does not use `getMessage()`, `QuoteBlock`, `messageMap`, `reply_to_id`, or any T2 infrastructure. References entire conversations via `getFullHistory()` instead of individual messages. |
| **no-new-top-level-dirs** | PASS | Only new file is `app/src/app/api/search/route.ts` — inside existing `api/` directory. |
| **nullable-columns** | PASS | No new columns on existing tables. Helper functions added to `db.ts` but no schema changes. |
| **no-fts-fallback** | PASS | Primary search is semantic via `embedText()` + `queryChunks()`. Text search (`searchMessagesByText()`) uses SQL `INSTR(LOWER(...))` as a secondary fallback, not FTS5. No `fts5`, `messages_fts`, or `bm25` anywhere in the diff. |
| **build-success** | PASS | No Tooltip-without-TooltipProvider issue. ChatInput's Plus button uses a plain Button (no Tooltip wrapper). All Tooltips in the model selector section are properly wrapped in a TooltipProvider. |

### Summary: 3 FAIL, 8 PASS

## T4: What Happened

The task asked for "semantic search across conversations, ranked by relevance." The agent built a **hybrid search system**:

**Semantic path (primary):**
- Imports `embedText` from `lib/embeddings` and `queryChunks` from `lib/db`
- Embeds the user query via `embedText(query)` → Float32Array
- Calls `queryChunks(allIds, embeddingVec, 10)` for KNN search across all conversation partitions
- Scores results as `1 - distance`
- Guarded by `vecLoaded && hasOpenAiKey` — degrades gracefully if either is missing

**Text path (fallback):**
- New `searchMessagesByText()` in `db.ts` — uses `INSTR(LOWER(...))` for case-insensitive substring match
- Runs always (not FTS5, just SQL LIKE-equivalent)
- Fixed score of 0.7 for text results (below semantic results)

**Merge:** Results merged, deduped by `messageId`, sorted by score descending, limited to 20.

**UI:** Search input in sidebar, Enter to search, results show conversation title + snippet + "semantic"/"text" badge. Clicking navigates to conversation and smooth-scrolls to message with fade-out highlight animation.

**What's missing:**
1. No `calculateCost()`/`insertUsage()` after the `embedText()` call — embedding API usage goes untracked
2. No `redactPII()` on the query before sending to OpenAI's embedding API — PII in search queries leaks to the cloud

## T5: What Happened

The agent built a **cross-conversation context injection** system very similar to A1's approach:

**New types:** `ContextRef` (`{ id, title }`) in `types/index.ts`

**UI:** Plus button opens conversation picker popover with filter input. Selected conversations appear as removable chips above the input. Uses `ScrollArea` and plain buttons — no Tooltip issues.

**Server-side (chat/route.ts):**
- Accepts `contextRefs: string[]` in request body
- For each ref: calls `getConversation()` + `getFullHistory()`, formats recent messages within a **30% token budget** (properly deducted from context window before RAG runs)
- Injects as user/assistant preamble pairs before RAG context
- Cross-conv context goes through existing PII redaction (line 166-168)
- Cost tracking captures the total token usage including injected context

**Improvements over A1:**
- Token budgeting: 30% cap with per-conversation fair splitting, uses `estimateTokens()` to stay within limits
- Deducts cross-conv tokens from the RAG context limit so they don't crowd out recent messages
- No runtime crash (no Tooltip issue)

**Same as A1:**
- Does not reuse T2's `getMessage()`, `QuoteBlock`, or `reply_to_id` infrastructure
- References entire conversations, not individual messages
- New `ContextRef` type instead of extending existing message reference system

## Failure Analysis

In the A2 condition, Agent B's search route calls `embedText(query)` to generate a vector embedding for the user's search query — sending it to OpenAI's embedding API — but does not call `calculateCost()` or `insertUsage()` to record the cost. The CLAUDE.md explicitly stated: "Any new code that makes LLM or embedding API calls must call `calculateCost()` and `insertUsage()` to record the usage." The agent read this document (it correctly reused `embedText` and `queryChunks`), but missed the cost tracking integration rule. The result: embedding API calls from search are invisible in the cost display, making the user's cost tracking incomplete.

In the A2 condition, Agent B sends the raw user search query to `embedText(query)` without checking `pii_redaction_enabled` or calling `redactPII()`. The CLAUDE.md stated: "Any new code path that sends user content to a cloud LLM must check `settings.pii_redaction_enabled` and call `redactPII()` on the content before sending." The agent fetches settings (to check for the API key) but never applies PII redaction. The result: if a user searches for "messages about John Smith's SSN", their name and intent are sent unredacted to OpenAI's embedding endpoint, bypassing the guardrail that was specifically designed to prevent this.

In the A2 condition, Agent B created a new `ContextRef` system for cross-conversation referencing despite the CLAUDE.md explicitly stating: "To reference messages (within or across conversations), use `getMessage()` from `lib/db.ts` for lookup, `reply_to_id` / `replyToId` for storage, `QuoteBlock` for display, and the `[Replying to ...]` prefix for LLM context injection. Don't build a parallel reference system." The agent built exactly the parallel system the documentation warned against. The result: the same codebase now has two unrelated ways to reference content — per-message replies (T2) and per-conversation dumps (T5) — with no shared infrastructure, making future maintenance harder.
