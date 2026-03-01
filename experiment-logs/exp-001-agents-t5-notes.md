feature: reference context from other conversation

diff:
 app/src/app/api/chat/route.ts    |  63 +++++++++++++++++++++++++++++++++++++++++++++++++++++---
 app/src/app/page.tsx             |  20 +++++++++++++++++-
 app/src/components/ChatInput.tsx | 125 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++---------
 app/src/types/index.ts           |   6 ++++++
 app/tsconfig.tsbuildinfo         |   2 +-
 5 files changed, 201 insertions(+), 15 deletions(-)

changes:
┌───────────────────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐  
  │       File        │                                                            Change                                                             │  
  ├───────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤  
  │ types/index.ts    │ Added ContextRef type ({ id, title })                                                                                         │  
  ├───────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤  
  │ page.tsx          │ Added contextRefs state, passes conversations + ref callbacks to ChatInput, includes contextRefs IDs in POST body, clears on  │
  │                   │ send/new chat                                                                                                                 │
  ├───────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ ChatInput.tsx     │ Wired Plus button to open conversation picker popover (positioned above, with text filter, click-outside dismiss), shows      │
  │                   │ attached refs as removable chips above input                                                                                  │
  ├───────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ api/chat/route.ts │ Accepts contextRefs string array, fetches full history for each via getFullHistory(), formats as ROLE: content lines within a │
  │                   │  30% token budget, injects as user/assistant preamble pairs before RAG context                                                │
  └───────────────────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  How it works:
  1. Click the + button next to the text input — a popover appears listing all other conversations (with a filter input)
  2. Select a conversation — it appears as a chip above the input showing "Context from: [title]"
  3. You can attach multiple conversations and remove them with the X button
  4. When you send a message, the backend fetches the full history of each referenced conversation, formats it within a token budget (30% of available
  context), and prepends it to the LLM prompt
  5. The model sees the cross-conversation context alongside RAG context and your current conversation
  6. PII redaction automatically applies to the injected content