feature: add ability to reply to a specific message in the chat

diff:
 app/src/app/api/chat/route.ts      |  26 +++++++++++++++++--
 app/src/app/page.tsx               |  24 +++++++++++++++--
 app/src/components/ChatInput.tsx   |  34 +++++++++++++++++++++++-
 app/src/components/MessageList.tsx | 155 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++-------------------------------------------------
 app/src/lib/db.ts                  |  23 ++++++++++++++---
 app/src/types/index.ts             |   2 ++
 app/tsconfig.tsbuildinfo           |   2 +-
 7 files changed, 187 insertions(+), 79 deletions(-)

changes:
  1. app/src/types/index.ts — Added reply_to_id?: string | null to Message and replyToId?: string | null to ClientMessage.
  2. app/src/lib/db.ts — Added ALTER TABLE messages ADD COLUMN reply_to_id migration. Added getMessage(id) helper to look up a single message. Updated
  insertMessage to accept optional replyToId parameter. Updated getMessages SELECT to include reply_to_id.
  3. app/src/app/api/chat/route.ts — Accepts optional replyToId in the request body. Passes it to insertMessage for the user message. Looks up the quoted
   message via getMessage, then prepends [Replying to {role}: "..."] context to the final user message in the LLM history so the model understands what's
   being referenced. The raw user text is stored in the DB without the prefix.
  4. app/src/app/page.tsx — Added replyTo state. handleSend captures the current reply, clears it, sets replyToId on the optimistic user message, and
  sends replyToId to the API. fetchMessages maps DB reply_to_id to client replyToId. Reply state is cleared on new chat. Passes onReply to MessageList
  and replyTo/onCancelReply to ChatInput.
  5. app/src/components/MessageList.tsx — Added onReply prop and "Reply" button in hover actions for all messages (alongside Branch and Copy). Added
  QuoteBlock component that renders a compact left-bordered quote above messages that have replyToId. Uses a messageMap (memoized) for O(1) lookup of
  quoted messages.
  6. app/src/components/ChatInput.tsx — Added replyTo and onCancelReply props. When replyTo is set, shows a dismissable preview bar above the input
  capsule with "Replying to {role}" label, truncated content, and an X button to cancel. Auto-focuses the textarea when entering reply mode.

feedback:
reply feature works on frontend, but unclear if it works on backend. 
button is well-placed but uses emoji which doesn't match design language of the rest of the app.
ui to show message being replied to is good