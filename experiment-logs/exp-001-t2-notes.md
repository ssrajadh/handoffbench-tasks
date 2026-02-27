diff:
 app/src/app/api/chat/route.ts      |  26 +++++++++++++++++--
 app/src/app/page.tsx               |  24 +++++++++++++++--
 app/src/components/ChatInput.tsx   |  34 +++++++++++++++++++++++-
 app/src/components/MessageList.tsx | 155 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++-------------------------------------------------
 app/src/lib/db.ts                  |  23 ++++++++++++++---
 app/src/types/index.ts             |   2 ++
 app/tsconfig.tsbuildinfo           |   2 +-
 7 files changed, 187 insertions(+), 79 deletions(-)

reply feature works on frontend, but unclear if it works on backend. 
button is well-placed but uses emoji which doesn't match design language of the rest of the app.
ui to show message being replied to is good