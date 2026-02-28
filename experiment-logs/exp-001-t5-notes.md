feature: reference context from other conversation

diff:
 app/src/app/api/chat/route.ts    |  31 ++++++++++++++++++++++++-
 app/src/app/page.tsx             |  20 +++++++++++++++-
 app/src/components/ChatInput.tsx | 140 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++--------
 app/tsconfig.tsbuildinfo         |   2 +-
 4 files changed, 180 insertions(+), 13 deletions(-)

changes:
1. ChatInput.tsx — Conversation picker + context chips                                                                                                 
  - The existing + button now opens a floating conversation picker panel                                                                                 
  - Picker shows all conversations (excluding current), filterable by title
  - Selecting a conversation adds it as a dismissible chip above the input capsule
  - Chips show conversation title with a MessageSquare icon and X to remove
  - Click outside or Escape closes the picker
  - Tooltip on the + button: "Add context from another chat"

  2. page.tsx — State management + API wiring
  - New contextRefs state (array of {id, title})
  - Handlers for add/remove passed to ChatInput
  - Context ref IDs sent in the API request body (contextRefs: string[])
  - Refs cleared after send (ephemeral, like reply-to)
  - Refs cleared on new chat

  3. chat/route.ts — Cross-conversation context injection
  - Accepts optional contextRefs array of conversation IDs
  - For each referenced conversation, fetches its full message history (respecting branching lineage via getFullHistory)
  - Injects the referenced context into the provider history as a user/assistant exchange before the main conversation messages
  - Format: "Context from other conversations..." with each ref labeled by title
  - Errors loading individual refs are logged but don't break the request

  How it works end-to-end

  1. In Chat B, click + → pick "Chat A" (database schema discussion)
  2. A chip appears: [MessageSquare] Chat A [X]
  3. Type your question about implementation → hit Enter
  4. The full history of Chat A is fetched server-side and injected as context
  5. The model sees both the referenced discussion and your current conversation
  6. The chip clears after send — attach again for the next message if needed