# OpenGrove — Agent Context Document

## 1. Project Overview

OpenGrove is a local-first AI chat application with conversation branching and RAG-based long-term memory. Users chat with multiple LLM providers (Gemini, OpenAI, local models), and the app automatically embeds older messages into a vector store so they can be retrieved when the conversation exceeds the context window.

**Tech stack:**
- **Runtime:** Next.js 14 (App Router), React 18, TypeScript 5.6
- **Database:** SQLite via `better-sqlite3`, vector search via `sqlite-vec` 0.1.7-alpha.2
- **LLM providers:** Google GenAI SDK (`@google/genai`), OpenAI SDK (`openai`), any OpenAI-compatible local endpoint
- **Embeddings:** OpenAI `text-embedding-3-small` (1536 dimensions)
- **UI:** Tailwind CSS, Radix UI primitives, React Markdown with remark-gfm + rehype-highlight, Lucide icons

The database file (`opengrove.db`) lives at the repo root. All data is local — no remote sync, no auth.

## 2. Architecture

```
opengrove/
├── .env.example          # Environment template
├── opengrove.db          # SQLite database (created at runtime)
├── app/                  # Next.js application
│   ├── package.json
│   ├── next.config.js    # Loads .env from parent directory
│   └── src/
│       ├── app/
│       │   ├── layout.tsx           # Root layout, dark mode
│       │   ├── page.tsx             # Main chat UI (client component, all state here)
│       │   ├── globals.css          # CSS variables, prose-chat styles
│       │   ├── settings/page.tsx    # Settings page (API keys, models, local runtime)
│       │   └── api/                 # API routes (see section below)
│       ├── components/
│       │   ├── Sidebar.tsx          # Tree-structured conversation list
│       │   ├── MessageList.tsx      # Message rendering, branch/copy buttons, markdown
│       │   ├── ChatInput.tsx        # Textarea, model selector, send button
│       │   ├── ConfirmModal.tsx     # Confirmation dialog
│       │   └── ui/                  # Radix UI primitive wrappers (button, input, etc.)
│       ├── lib/
│       │   ├── db.ts               # All SQLite operations (544 lines, the core)
│       │   ├── rag.ts              # RAG orchestrator: budget split, retrieval, context building
│       │   ├── tokens.ts           # Token estimation, message windowing
│       │   ├── embeddings.ts       # OpenAI embedding calls, chunking, store overflow
│       │   ├── model-constants.ts  # Model ID mappings for Gemini and OpenAI
│       │   └── utils.ts            # cn() utility (clsx + tailwind-merge)
│       └── types/
│           └── index.ts            # Conversation, Message, ClientMessage, LineageEntry
```

**What lives where:**
- `lib/` — All server-side logic. `db.ts` is the data layer. `rag.ts`, `tokens.ts`, `embeddings.ts` form the RAG pipeline. No client code imports from `lib/`.
- `components/` — React client components. `ui/` contains Radix primitive wrappers. Top-level components (`Sidebar`, `MessageList`, `ChatInput`) are the app's UI building blocks.
- `api/` — Next.js route handlers. Each route file exports `GET`, `POST`, or `DELETE` async functions. All routes use `export const dynamic = "force-dynamic"` to disable caching.
- `types/` — Shared TypeScript interfaces used by both client and server.

## 3. Database

SQLite with WAL mode implied by `better-sqlite3`. The schema is defined inline in `db.ts` with `CREATE TABLE IF NOT EXISTS` and migration-style `ALTER TABLE` wrapped in try-catch for idempotency.

### Core Tables

**conversations**
```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,                          -- UUID
  title TEXT NOT NULL DEFAULT 'New chat',
  model TEXT NOT NULL DEFAULT 'gemini-2.0-flash',
  created_at INTEGER NOT NULL DEFAULT (unixepoch()),
  parent_id TEXT DEFAULT NULL,                  -- NULL for root, FK for branches
  branch_point_index INTEGER DEFAULT NULL       -- message index where branch diverges
);
```

**messages**
```sql
CREATE TABLE messages (
  id TEXT PRIMARY KEY,                          -- UUID
  conversation_id TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
  content TEXT NOT NULL,
  created_at INTEGER NOT NULL DEFAULT (unixepoch()),
  is_embedded INTEGER NOT NULL DEFAULT 0,       -- 1 = already chunked and stored in vector table
  FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE
);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
```

**settings**
```sql
CREATE TABLE settings (
  key TEXT PRIMARY KEY,      -- Validated against SUPPORTED_SETTINGS_KEYS whitelist
  value TEXT NOT NULL,
  updated_at INTEGER NOT NULL DEFAULT (unixepoch())
);
```
Supported keys: `openai_api_key`, `gemini_api_key`, `anthropic_api_key`, `default_model`, `hidden_models`, `local_models_enabled`, `local_runtime`, `local_endpoint`, `local_models_hidden`.

**embedding_config** (singleton)
```sql
CREATE TABLE embedding_config (
  id INTEGER PRIMARY KEY DEFAULT 1 CHECK (id = 1),  -- Enforced single row
  model TEXT NOT NULL,
  dimensions INTEGER NOT NULL,
  updated_at INTEGER NOT NULL DEFAULT (unixepoch())
);
```

### Vector Table (sqlite-vec)

```sql
CREATE VIRTUAL TABLE message_chunks USING vec0(
  chunk_id text primary key,
  conversation_id text partition key,
  +chunk_text text,
  +start_msg_index integer,
  +end_msg_index integer,
  +embedding_model text,
  +created_at integer,
  embedding float[1536]
);
```

Key details:
- `conversation_id` is a **partition key** — sqlite-vec requires querying one partition at a time, so `queryChunks` loops over each conversation ID separately and merges results.
- `+column` syntax stores auxiliary data alongside vectors.
- Embeddings are stored as `Buffer.from(float32Array.buffer)`.
- KNN query syntax: `WHERE embedding MATCH ? AND conversation_id = ? AND k = ?`.

### Schema Migrations

Columns added after initial schema (`parent_id`, `branch_point_index`, `is_embedded`) use try-catch `ALTER TABLE` — if the column already exists, the error is silently caught. This is the project's migration pattern. Follow it for any new columns.

## 4. RAG Pipeline

Defined in `lib/rag.ts`. The entry point is `buildContextWithRAG()`.

### Flow

1. **Budget split:** Available tokens = `contextLimit - responseBuffer(4096)`. RAG gets 20% (`RAG_BUDGET_RATIO = 0.2`), recent messages get 80%.
2. **Window recent messages:** `buildMessageWindow()` fills the 80% budget with the most recent messages (see Token Windowing below). Messages that don't fit become `overflow`.
3. **Skip conditions:** RAG is skipped if: sqlite-vec not loaded, `OPENAI_API_KEY` missing, no overflow messages, or query is < 10 tokens.
4. **Retrieve chunks:** Embed the user query, resolve conversation lineage, run KNN against each ancestor's partition, filter by `maxMsgIndex` for branch-safety, sort by distance, take top-k (default 5) within token budget.
5. **Return:** `{ ragContext: string | null, recentMessages: Message[], overflow: Message[] }`.

### Context Injection

The chat route (`api/chat/route.ts`) prepends RAG context as a synthetic user/assistant pair at the start of the message history:

```ts
{ role: "user", content: "Relevant context from earlier in this conversation:\n" + ragContext }
{ role: "assistant", content: "Understood, I have that context." }
```

### Overflow Embedding (async)

After the assistant response is streamed, `embedAndStoreOverflow()` is called **fire-and-forget** (`.catch()` swallows errors). It:
1. Filters to unembedded messages only (checks `is_embedded` flag)
2. Chunks messages in groups of 4, formatted as `"USER: ...\nASSISTANT: ..."`
3. Batch-embeds all chunks in a single OpenAI API call
4. Stores each chunk with its `start_msg_index` / `end_msg_index`
5. Marks source messages as `is_embedded = 1`

### Graceful Degradation

If anything in the RAG path fails (sqlite-vec missing, API key missing, embedding call fails), the system logs the error and proceeds without RAG context. Chat never breaks because of RAG.

## 5. Token Windowing

Defined in `lib/tokens.ts`.

**Token estimation:** `Math.ceil(text.length / 4)` — rough approximation, intentionally imprecise. The 4096-token response buffer absorbs estimation error.

**`buildMessageWindow(messages, tokenBudget)`:**
1. Walk backwards from the most recent message.
2. If the current message is an `assistant` message and the previous is `user`, treat them as a **pair**. Compute combined cost. If the pair fits, include both. If not, stop.
3. Solo messages (consecutive same-role, or edge cases) are handled individually.
4. **Critical invariant: user/assistant pairs are never split.** If a pair doesn't fit, both are excluded.
5. Everything from index 0 through the cutoff point becomes `overflow` (fed to the embedding pipeline).
6. Return `{ window, overflow }` — window is in chronological order.

## 6. Embedding System

Defined in `lib/embeddings.ts`.

- **Model:** `text-embedding-3-small`, 1536 dimensions (constants exported as `EMBEDDING_MODEL`, `EMBEDDING_DIMENSIONS`)
- **API:** OpenAI embeddings API via the `openai` SDK. `embedTexts()` batches multiple texts in one call. `embedText()` is a single-text convenience wrapper.

### Config Lifecycle (`ensureEmbeddingConfig`)

Called once per process (guarded by `configChecked` flag). Handles three cases:
1. **First run:** Creates the `embedding_config` row and the `message_chunks` virtual table.
2. **Model changed:** Wipes all vectors (`resetAllEmbeddings` — sets `is_embedded = 0` on all messages, drops and recreates the virtual table), updates config.
3. **Same model:** Ensures the virtual table exists (idempotent no-op).

If you change the embedding model or dimensions, update the constants in `embeddings.ts`. The system auto-detects the mismatch on next startup and re-embeds everything.

### Chunking

`chunkMessages(messages, chunkSize=4)` groups messages into chunks of 4, serialized as:
```
USER: message content
ASSISTANT: response content
USER: next message
ASSISTANT: next response
```
Each chunk tracks `messageIds`, `startIndex`, and `endIndex` for later reference.

## 7. Branch-Aware Lineage

Conversations form a **tree**. Root conversations have `parent_id = NULL`. Branches store `parent_id` (which conversation they forked from) and `branch_point_index` (which message index in the parent's history the fork happened at).

### Key Operations

**`createBranch(parentId, branchPointIndex)`:** Creates a new conversation row with `parent_id` and `branch_point_index`. No messages are copied — the branch *references* its parent's messages up to the branch point via the lineage chain. Only new messages after the branch point live in the branch's own `messages` rows.

**`getConversationLineage(conversationId)`:** Walks up the `parent_id` chain. Returns `[self, parent, grandparent, ..., root]`. Each entry has `{ conversationId, branchPointIndex }`. Uses a `visited` set to prevent infinite loops.

**`getFullHistory(conversationId)`:** Reconstructs the complete message history by walking the lineage bottom-up (root first). For each ancestor, takes messages up to the next descendant's `branchPointIndex + 1`. For the conversation itself, takes all its own messages. Concatenates all segments.

**`deleteConversationTree(id)`:** BFS collects all descendants, deletes vector chunks for each, then batch-deletes all conversations. Message deletion cascades via FK.

### Branch-Aware RAG

When retrieving chunks for a branched conversation, the lineage is resolved and each ancestor is queried separately. For ancestor conversations, results are filtered to `start_msg_index <= branchPointIndex` to prevent cross-branch contamination.

## 8. Environment Setup

**`.env` location:** The app looks for `.env` at the **repo root** (one level above `app/`). This is configured in `next.config.js`:
```js
require("dotenv").config({ path: path.resolve(__dirname, "..", ".env") });
```
Copy `.env.example` to `.env` and fill in API keys.

**Required keys:**
- `GEMINI_API_KEY` — For Gemini models (also configurable via Settings UI → stored in `settings` table)
- `OPENAI_API_KEY` — For OpenAI models AND for embeddings (RAG requires this)

**Database path:** `path.join(process.cwd(), "..", "opengrove.db")` — assumes the Next.js dev server runs from `app/`. The DB is created automatically on first run.

**Running the app:**
```bash
cd app
npm install
npm run dev
```

## 9. Conventions

### Nullable Columns for Backward Compatibility

New columns are added via `ALTER TABLE ... ADD COLUMN ... DEFAULT NULL` wrapped in try-catch. This means:
- New columns must be nullable or have a default value.
- Existing rows get the default automatically.
- The try-catch swallows "column already exists" errors, making the migration idempotent.
- Follow this exact pattern for any schema additions.

### File Naming

- Database columns: `snake_case` (`conversation_id`, `branch_point_index`, `is_embedded`)
- TypeScript: `camelCase` (`conversationId`, `branchPointIndex`, `isEmbedded`)
- Constants: `UPPER_SNAKE_CASE` (`EMBEDDING_MODEL`, `RAG_BUDGET_RATIO`, `RESPONSE_BUFFER_TOKENS`)
- Components: `PascalCase` filenames (`Sidebar.tsx`, `MessageList.tsx`)
- Lib modules: `kebab-case` filenames (`model-constants.ts`)

### API Route Patterns

- One file per route: `app/api/[path]/route.ts`
- Export named async functions: `GET`, `POST`, `DELETE`
- Dynamic params use Next.js 15 pattern: `{ params }: { params: Promise<{ id: string }> }` — must `await params`
- All list/read routes use `export const dynamic = "force-dynamic"` to prevent caching
- Success responses: `NextResponse.json(data)` or `NextResponse.json(data, { status: 201 })`
- Error responses: `NextResponse.json({ error: "message" }, { status: 4xx/5xx })`
- Chat streaming: Returns a `Response` with `ReadableStream`, content type `application/x-ndjson`

### Streaming Protocol (Chat)

The chat endpoint returns newline-delimited JSON (NDJSON):
```json
{"type": "chunk", "text": "partial response..."}
{"type": "chunk", "text": "more text..."}
{"type": "done", "conversationId": "uuid", "message": {"role": "assistant", "content": "full text", "id": "uuid"}}
```
On error during streaming:
```json
{"type": "error", "error": "description"}
```

### Error Handling

- API routes: try-catch at the top level, return JSON error with status code.
- Embeddings: fire-and-forget with `console.error`. Never throws into the chat flow.
- RAG: catches errors and returns `ragContext: null`. Chat continues without RAG.
- sqlite-vec operations: guarded by `vecLoaded` boolean. All vector functions are no-ops when vec isn't loaded.

### DB Function Signatures

All `db.ts` functions are `async` even though `better-sqlite3` is synchronous. This is a consistency convention — it lets them be used with `await` uniformly and makes future migration to async drivers easier.

### Settings Architecture

API keys can come from two places (checked in order):
1. The `settings` table (set via the Settings UI)
2. Environment variables (`.env` file)

The chat route checks both: `settings.openai_api_key?.trim() || process.env.OPENAI_API_KEY`.

## 10. Key Gotchas

1. **DB path is relative.** `process.cwd()` must be `app/` for the path `../opengrove.db` to resolve correctly. If you change how the app is started, the DB path will break.

2. **sqlite-vec can fail to load.** If the native extension doesn't compile or isn't available, `vecLoaded` is `false` and all RAG features silently disable. Check the console for the warning message.

3. **Embeddings require OPENAI_API_KEY regardless of chat provider.** Even if you only use Gemini for chat, RAG embedding always uses OpenAI's embedding API. No OpenAI key = no RAG.

4. **The `embedding_config` table is a singleton.** The `CHECK (id = 1)` constraint enforces exactly one row. Use `upsertEmbeddingConfig`, not raw INSERT.

5. **Partition key queries.** sqlite-vec requires querying one `conversation_id` at a time. `queryChunks` handles this by looping, but if you write new vector queries, you must do the same — you cannot query across partitions in a single statement.

6. **Branch messages are not copied.** When a branch is created, no messages are duplicated. The branch references parent messages via lineage. This means deleting a parent conversation deletes the shared history for all its branches (cascade delete handles this).

7. **`getFullHistory` is the only correct way to get a conversation's messages.** Don't use `getMessages` alone for branched conversations — it only returns messages owned directly by that conversation, not inherited ancestor messages.

8. **User/assistant pair invariant.** The windowing algorithm assumes messages alternate user/assistant. If you insert messages that break this pattern (e.g., consecutive user messages), the windowing still works but may include unpaired messages as singletons.

9. **`hidden_models` is stored as JSON string.** The settings table stores it as a serialized JSON array. The settings API route parses/serializes it. Don't store raw arrays in the settings table.

10. **`params` is a Promise.** Next.js dynamic route params must be awaited: `const { id } = await params`. This is the Next.js 15 convention used throughout the codebase. Forgetting to await will cause runtime errors.

11. **Model context sizes are hardcoded.** `MODEL_CONTEXT_TOKENS` in `api/chat/route.ts` maps model IDs to context window sizes. If you add a new model, add its context size here or it defaults to 32,768 tokens.

12. **Fire-and-forget embedding.** `embedAndStoreOverflow` runs after the response is streamed. If the process dies mid-embedding, those messages stay `is_embedded = 0` and will be picked up next time there's overflow containing them.
