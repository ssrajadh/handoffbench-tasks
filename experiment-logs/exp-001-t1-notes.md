 diff:
 app/src/app/api/chat/route.ts               | 30 ++++++++++++++++++++++++++++++
 app/src/app/api/conversations/[id]/route.ts |  4 +++-
 app/src/app/page.tsx                        | 48 ++++++++++++++++++++++++++++++++++++++++++++----
 app/src/lib/db.ts                           | 46 ++++++++++++++++++++++++++++++++++++++++++++--
 app/src/types/index.ts                      | 19 +++++++++++++++++++
 app/tsconfig.tsbuildinfo                    |  2 +-
 6 files changed, 141 insertions(+), 8 deletions(-)

 shows running cost + token count in top right corner
 matches rest of UI
 persists on refresh, shutdown