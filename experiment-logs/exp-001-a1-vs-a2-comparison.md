# Exp-001: A1 (Cold) vs A2 (agents.md) Comparison

## Side-by-Side Pattern Check Results

| Check | A1 (Cold) | A2 (agents.md) | Change |
|-------|-----------|-----------------|--------|
| **no-new-vector-db** | PASS | PASS | — |
| **reuses-embeddings-module** | FAIL (FTS5) | PASS (embedText + queryChunks) | **Fixed** |
| **cost-tracking-integration-t4** | PASS (vacuous) | FAIL (embedText call untracked) | **Regressed** (but more honest) |
| **cost-tracking-integration-t5** | PASS | PASS | — |
| **pii-pipeline-integration-t4** | PASS (vacuous) | FAIL (query sent unredacted) | **Regressed** (but more honest) |
| **pii-pipeline-integration-t5** | PASS | PASS | — |
| **message-reference-reuse** | FAIL | FAIL | No change |
| **no-new-top-level-dirs** | PASS | PASS | — |
| **nullable-columns** | PASS | PASS | — |
| **no-fts-fallback** | FAIL (critical) | PASS | **Fixed** |
| **build-success** | FAIL (critical) | PASS | **Fixed** |

### Raw Score

| | A1 (Cold) | A2 (agents.md) |
|---|-----------|-----------------|
| PASS | 5 | 8 |
| PASS (vacuous) | 2 | 0 |
| FAIL | 3 | 3 |
| FAIL (critical) | 2 | 0 |
| **Genuine PASS** | **5** | **8** |

## What agents.md Fixed

### 1. Semantic search architecture (reuses-embeddings-module + no-fts-fallback)

The single biggest improvement. A1 built FTS5 full-text keyword search, completely ignoring the existing sqlite-vec + embeddings infrastructure. A2 correctly imported `embedText` from `lib/embeddings` and `queryChunks` from `lib/db` to build vector search. The agents.md had extensive documentation of the embedding system (section 6) including function signatures, and the cost tracking integration rule (section 10) explicitly mentioned embedding API calls. The agent followed the architecture.

A2 also added text search as a graceful fallback (SQL `INSTR` substring match, not FTS5) — a good design decision that ensures search works even without an OpenAI key or when sqlite-vec fails to load.

### 2. No runtime crash (build-success)

A1 crashed because the ChatInput conversation picker used a Radix Tooltip without a TooltipProvider wrapper. A2 used a plain Button for the Plus button (no Tooltip needed). The agents.md documented the Radix UI primitives pattern, and the codebase conventions helped the agent avoid the mistake.

### 3. Token budgeting for cross-conv context (quality improvement, no dedicated check)

A1 dumps entire referenced conversation histories into the LLM prompt with no size limit. A2 implements proper token budgeting: 30% of available context for cross-conv references, deducted from the RAG window, with fair splitting across multiple referenced conversations and `estimateTokens()` for sizing. This follows the RAG budget split pattern documented in sections 4-5 of agents.md.

## What agents.md Didn't Fix

### 1. Message reference reuse (FAIL in both)

Both conditions created a new `ContextRef` system for T5 instead of extending T2's `reply_to_id` / `getMessage()` / `QuoteBlock` infrastructure. This is despite agents.md section 11 containing an explicit integration rule: _"Don't build a parallel reference system."_

The likely reason: the T5 task inherently requires referencing entire conversations, not individual messages. T2's infrastructure is message-level (`reply_to_id` points to one message). An agent interpreting "reference context from another conversation" may reasonably conclude that conversation-level referencing is a different problem than message-level replying. The agents.md integration rule may be too prescriptive — it says to extend T2's system, but the T5 use case (pulling in full conversation histories) doesn't map cleanly onto per-message references.

This suggests the pattern check should be refined: does T5 need to reuse the exact T2 components, or just follow the same architectural principles (stored references, rendered previews, context injection)?

## What agents.md Partially Fixed

### 1. Cost tracking (PASS for T5, FAIL for T4)

**T5:** Both A1 and A2 pass — cross-conv context goes through the existing chat flow, so `calculateCost()` + `insertUsage()` capture the full token usage including injected context.

**T4:** A1 passes vacuously (FTS5, no API calls). A2 fails genuinely: the search route calls `embedText(query)` (OpenAI API) but never calls `calculateCost()` or `insertUsage()`. The agents.md section 10 explicitly states: _"Any new code that makes LLM or embedding API calls must call `calculateCost()` and `insertUsage()` to record the usage."_ The agent read this (it correctly reused the embeddings module) but didn't follow the cost tracking rule for the embedding call.

Note: A2's T4 failure is more "honest" than A1's vacuous pass — A2 actually did the right thing architecturally (semantic search) but missed a detail. A1 avoided the failure entirely by taking the wrong approach.

### 2. PII pipeline (PASS for T5, FAIL for T4)

Same pattern. T5 works in both conditions because cross-conv context flows through the existing `history` array pipeline where `redactPII()` runs.

T4 in A2: the search route sends the raw query to `embedText()` without checking `pii_redaction_enabled` or calling `redactPII()`. The agents.md section 12 states: _"Any new code path that sends user content to a cloud LLM must check `settings.pii_redaction_enabled` and call `redactPII()` on the content before sending."_ The agent fetches settings (for the API key check) but doesn't apply PII redaction. This is a subtler failure — the agent may not have considered embedding API calls as "sending to a cloud LLM" even though the data goes to OpenAI either way.

## New Issues Introduced by agents.md

None. A2 has no new failures that A1 doesn't also have. The two "new" FAILs (cost-tracking-t4, pii-t4) replace vacuous PASSes that only existed because A1 took the wrong approach. These are not regressions in any meaningful sense — they're newly exposed gaps because the agent is now doing the right thing (semantic search) and encountering integration requirements that FTS5 avoided entirely.

## Summary

| Metric | A1 (Cold) | A2 (agents.md) |
|--------|-----------|-----------------|
| Critical failures | 2 (FTS5, runtime crash) | 0 |
| Genuine failures | 1 (message-ref reuse) | 3 (message-ref, cost-t4, pii-t4) |
| Vacuous passes | 2 | 0 |
| Genuine passes | 5 | 8 |
| Architecture correct | T4 no, T5 partial | T4 yes, T5 partial |
| App works | No (crash) | Yes |

**Bottom line:** agents.md eliminated both critical failures (FTS5 architecture and runtime crash) and produced a working application with correct semantic search architecture. The remaining failures are integration-detail misses (cost tracking and PII on the search embedding path, message reference reuse) rather than architectural mistakes.

The agents.md didn't help with T5's message-reference-reuse despite explicitly documenting the integration rule. This suggests that explicit "don't do X" instructions may not be sufficient when the agent's own design intuitions point in a different direction. The T5 task naturally suggests conversation-level referencing, and both agents converged on that design independently.

The T4 cost-tracking and PII failures reveal a pattern: agents read and follow architectural instructions (use embeddings, use queryChunks) but miss cross-cutting integration rules (track costs, redact PII) for novel code paths. The agents.md documented these rules clearly, but the agent applied them selectively — following the architecture rules but not the integration rules for the same code.
