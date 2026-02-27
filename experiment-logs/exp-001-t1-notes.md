 diff:
 app/src/app/api/chat/route.ts  | 26 +++++++++++++++++++++++++-
 app/src/app/page.tsx           | 33 +++++++++++++++++++++++++++++----
 app/src/lib/db.ts              | 46 ++++++++++++++++++++++++++++++++++++++++++++++
 app/src/lib/model-constants.ts | 24 ++++++++++++++++++++++++
 app/tsconfig.tsbuildinfo       |  2 +-
 5 files changed, 125 insertions(+), 6 deletions(-)

 shows running cost in top right corner
 matches rest of UI
 persists on refresh, shutdown