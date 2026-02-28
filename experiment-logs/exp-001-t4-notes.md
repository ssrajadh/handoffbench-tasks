feature: chat history semantic search

diff:
app/src/app/page.tsx               |  12 +++++++
app/src/components/MessageList.tsx |  37 ++++++++++++++++++---
app/src/components/Sidebar.tsx     | 143 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++------
app/src/lib/db.ts                  |  85 ++++++++++++++++++++++++++++++++++++++++++++++++
app/tsconfig.tsbuildinfo           |   2 +-
5 files changed, 264 insertions(+), 15 deletions(-)

changes:
  1. Database layer (app/src/lib/db.ts)                                                                                                                  
  - FTS5 virtual table (messages_fts) — a content-synced full-text search index on the messages table                                                    
  - Triggers to keep FTS in sync on INSERT/UPDATE/DELETE                                                                                                 
  - Rebuild on startup to backfill any existing messages                                                                                                 
  - searchMessages(query, limit) — queries FTS5 with BM25 relevance ranking, joins with conversations for titles, supports prefix matching               
                                                                                                                                                         
  2. Search API (app/src/app/api/search/route.ts)
  - GET /api/search?q=<query> — returns ranked results with messageId, conversationId, conversationTitle, role, content, and rank

  3. Sidebar search UI (app/src/components/Sidebar.tsx)
  - Search input with magnifying glass icon between "New chat" button and conversation list
  - Debounced (250ms) search-as-you-type
  - Clear button (X) to dismiss search
  - When searching: conversation tree is replaced with ranked results showing conversation title + message snippet with role label
  - Clicking a result navigates to that conversation and scrolls to the message, then clears the search

  4. Navigation wiring (app/src/app/page.tsx)
  - New scrollToMessageId state
  - handleSearchSelect(conversationId, messageId) — sets the current conversation and scroll target
  - Props passed through to MessageList

  5. Scroll-to-message (app/src/components/MessageList.tsx)
  - data-message-id attribute on each message element
  - When scrollToMessageId is set, scrolls smoothly to the target message and applies a brief blue highlight ring that fades after 2 seconds