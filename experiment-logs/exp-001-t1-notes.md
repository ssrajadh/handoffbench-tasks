 diff:
 app/src/app/api/chat/route.ts               | 29 ++++++++++++++++++++++++++++-
 app/src/app/api/conversations/[id]/route.ts |  4 +++-
 app/src/app/page.tsx                        | 30 ++++++++++++++++++++++++++----
 app/src/lib/db.ts                           | 49 +++++++++++++++++++++++++++++++++++++++++++++++++
 app/src/lib/model-constants.ts              | 33 +++++++++++++++++++++++++++++++++
 app/tsconfig.tsbuildinfo                    |  2 +-
 6 files changed, 140 insertions(+), 7 deletions(-)

 shows running cost in top right corner
 matches rest of UI
 persists on refresh, shutdown