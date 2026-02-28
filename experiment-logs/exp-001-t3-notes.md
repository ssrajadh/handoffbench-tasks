feature: add PII redaction before sending prompt to cloud model

diff:
 app/package-lock.json         | 35 +++++++++++++++++++++++++++++++++++
 app/package.json              |  1 +
 app/src/app/api/chat/route.ts | 13 ++++++++++---
 app/src/app/settings/page.tsx | 45 ++++++++++++++++++++++++++++++++++++++++++++-
 app/src/lib/db.ts             |  3 ++-
 app/tsconfig.tsbuildinfo      |  2 +-
 6 files changed, 93 insertions(+), 6 deletions(-)

changes:
  1. app/package.json — Added compromise (v14, ~200KB) as a dependency for NLP-based person name detection. Runs entirely locally.                       
  2. app/src/lib/pii.ts (new) — PII redaction module exporting redactPII(text): string. Uses:                                                            
    - Regex patterns for structured PII: SSNs → [SSN], credit cards → [CREDIT_CARD], emails → [EMAIL], US phone numbers → [PHONE], US street addresses → 
  [ADDRESS]                                                                                                                                              
    - compromise NLP for person names → [NAME] (deduplicates, replaces longest matches first)                                                            
    - Regex runs first so compromise doesn't try to match already-redacted placeholders                                                                  
  3. app/src/lib/db.ts — Added "pii_redaction_enabled" to SUPPORTED_SETTINGS_KEYS and updated getSettings() SQL query to 10 placeholders.                
  4. app/src/app/api/chat/route.ts — Hoisted getSettings() above history building (removed 3 duplicate calls in provider branches). After the history    
  array is fully built (including RAG context and reply context), checks if pii_redaction_enabled === "true" AND model is not local — if so, redacts     
  every entry in history via redactPII(). Original messages in the DB remain untouched.                                                                  
  5. app/src/app/settings/page.tsx — Populated the previously-empty General tab with a PII redaction toggle. Uses the same switch pattern as
  local_models_enabled: state (piiRedactionEnabled), loaded from settings, handlePiiRedactionChange callback for optimistic toggle + persist, with
  description explaining what it does.

feedback:
setting to toggle PII redaction included, and feature seems to work based on LLM response, but user sees original msg with no redactions being sent