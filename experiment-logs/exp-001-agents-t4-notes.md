feature: chat history semantic search

diff:
 app/src/app/globals.css            |  10 ++++++++
 app/src/app/page.tsx               |  34 ++++++++++++++++++++++++++
 app/src/components/MessageList.tsx |  19 +++++++++++++--
 app/src/components/Sidebar.tsx     | 147 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++-------
 app/src/lib/db.ts                  |  42 ++++++++++++++++++++++++++++++++
 app/src/types/index.ts             |  11 +++++++++
 app/tsconfig.tsbuildinfo           |   2 +-
 7 files changed, 253 insertions(+), 12 deletions(-)

changes:
│            File            │                                                        Change                                                        │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ types/index.ts             │ Added SearchResult type                                                                                              │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ lib/db.ts                  │ Added getAllConversationIds(), searchMessagesByText(), getConversationTitleMap()                                     │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ api/search/route.ts        │ New — POST endpoint with hybrid search (semantic vectors + SQL text), merged and ranked                              │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ components/Sidebar.tsx     │ Search input between "New chat" and conversation list; results replace tree when searching; Enter to search, Escape  │
  │                            │ to clear                                                                                                             │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ components/MessageList.tsx │ id attributes on message divs, highlightMessageId prop with auto-fading highlight                                    │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ app/page.tsx               │ scrollToMessageId state, handleSearchSelect callback, scroll-into-view effect                                        │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ globals.css                │ search-highlight-fade animation keyframes                                                                            │
  └────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  How it works:
  - Type a query in the sidebar search bar and press Enter
  - Semantic search embeds your query and finds relevant chunks across all conversations via sqlite-vec (when available)
  - Text search always runs as fallback, doing case-insensitive substring matching across all messages
  - Results are merged, ranked (semantic scores higher), and shown with conversation title, content snippet, and relevance badge
  - Clicking a text result navigates to the conversation and smooth-scrolls to the exact message with a blue highlight that fades out
  - Clicking a semantic result navigates to the conversation (no scroll since chunks span multiple messages)
  - Press Escape or clear the search bar to return to the conversation list