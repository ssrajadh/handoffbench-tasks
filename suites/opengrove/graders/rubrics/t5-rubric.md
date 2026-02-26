# T5 Cross-Chat Context Referencing — LLM Judge Rubric

You are evaluating a coding agent's implementation of cross-chat context referencing for OpenGrove, a chat application.

## What the agent was asked to do

The agent received this prompt:
[TASK_PROMPT]

## What the agent should have known

The codebase already has this infrastructure (established before the agent started):
- **sqlite-vec**: Virtual table for vector similarity search, already in lib/db.ts
- **Embeddings**: OpenAI text-embedding-3-small, implemented in lib/embeddings.ts
- **RAG pipeline**: lib/rag.ts with ~20% context budget allocation
- **Cost tracking** (added in T1): Tracks token usage and cost for every API call. All new API calls should integrate.
- **Quote-reply / message reference system** (added in T2): Components and schema for referencing specific messages or message fragments. New message referencing should extend this, not duplicate it.
- **PII guardrail** (added in T3): Intercepts prompts before cloud API calls to redact sensitive info. All outbound LLM calls should route through this.
- **Semantic search** (added in T4): Cross-conversation search using sqlite-vec and lib/embeddings.ts. Finding relevant messages across chats should reuse this, not reimplement it.
- **Branch-aware conversation model**: OpenGrove conversations are trees, not flat lists. Messages have parent pointers and conversations have lineage. Any feature that reads or injects conversation history must traverse the correct branch.

## The agent produced these changes
[GIT_DIFF]

## Ground truth from the establishment phase
[GROUND_TRUTH]

## Rate each dimension from 1 to 5:

### 1. Infrastructure Reuse
- **5**: Reuses sqlite-vec + lib/embeddings.ts for finding messages, AND reuses T4's search infrastructure for cross-chat lookup. No new vector DB. Extends existing patterns throughout.
- **4**: Reuses sqlite-vec and embeddings correctly but builds minor parallel logic on top of T4's search instead of importing it directly.
- **3**: Uses sqlite-vec but reimplements search logic already present from T4, or builds a significant new abstraction over existing infrastructure.
- **2**: Partially uses existing infrastructure but introduces redundant new code — e.g., a second embedding pipeline or a second vector search path alongside T4's.
- **1**: Installs a new vector database or creates entirely new embedding/search infrastructure, ignoring both the base codebase and T4's additions.

### 2. Message Reference Pattern Reuse
- **5**: Imports and extends T2's quote-reply components and message-reference schema to represent cross-chat references. The cross-chat reference UI and data model build directly on T2's system.
- **4**: Mostly reuses T2's system with minor additions or slight schema extensions that are compatible with T2's patterns.
- **3**: Partially reuses T2 — e.g., uses the same schema but reimplements the UI, or reuses the UI component but introduces a parallel schema for cross-chat references.
- **2**: Minimal reuse. Borrows a small piece of T2 (e.g., one component) but largely builds a new referencing system alongside it.
- **1**: Creates a completely new system for referencing messages across chats with no connection to T2's components or schema.

### 3. Cost Tracking Integration
- **5**: All new API calls (embedding lookups, any LLM calls for summarization or context injection) correctly integrate with the cost tracking system from T1. Token usage and cost are recorded.
- **4**: Most API calls integrated, minor gaps.
- **3**: Some calls tracked, some missed. Partial awareness of cost system.
- **2**: Minimal integration — one or two references but most calls bypass cost tracking.
- **1**: No awareness of cost tracking at all. New API calls completely bypass it. Cost dashboard becomes inaccurate.

### 4. PII Pipeline Compliance
- **5**: All cross-chat context that gets sent to cloud APIs (embedding calls, context injection into prompts) routes through the PII guardrail when enabled. Respects the user's PII setting.
- **4**: Most paths go through PII guardrail, minor edge cases missed.
- **3**: Partial — some API paths go through, some bypass.
- **2**: Minimal awareness. Maybe checks PII in one place but misses others.
- **1**: No awareness of PII pipeline. Cross-chat message content sent to cloud APIs without any PII check.

### 5. Branch/Lineage Awareness
- **5**: Correctly understands that OpenGrove conversations are trees. When pulling context from another conversation, it traverses the correct branch lineage (respects parent pointers, doesn't mix branches). Injected context reflects the actual conversation path, not all messages in a thread.
- **4**: Mostly correct branch traversal with minor edge cases (e.g., handles the common case correctly but may include a sibling branch message in rare situations).
- **3**: Partially aware — acknowledges branching but traversal logic has notable gaps or occasionally pulls messages from the wrong branch.
- **2**: Minimal awareness. Attempts some lineage handling but the logic is largely incorrect, leading to context from wrong branches being included.
- **1**: Treats conversations as flat lists. Ignores parent pointers and branch structure entirely, breaking for any branched conversation.

### 6. Functional Correctness
- **5**: Cross-chat referencing works end-to-end. User can browse or search for context from another chat, attach it to the current conversation, and the model correctly receives and uses that context.
- **4**: Works with minor issues (e.g., UI roughness, occasional stale results, minor context formatting issues).
- **3**: Partially works — the core flow functions but with notable bugs (e.g., context sometimes missing, navigation broken, or model doesn't receive the referenced context reliably).
- **2**: Significant functionality missing or broken. The feature exists in the UI but context injection doesn't work correctly or reliably.
- **1**: Feature doesn't work or breaks existing chat functionality.

## Instructions

Evaluate the git diff against the rubric above. For each dimension, provide:
1. Your score (1-5)
2. A brief justification (1-2 sentences)

Respond with ONLY a JSON object in this exact format:
{
  "infrastructure_reuse": {"score": N, "justification": "..."},
  "message_reference_reuse": {"score": N, "justification": "..."},
  "cost_tracking_integration": {"score": N, "justification": "..."},
  "pii_pipeline_compliance": {"score": N, "justification": "..."},
  "branch_lineage_awareness": {"score": N, "justification": "..."},
  "functional_correctness": {"score": N, "justification": "..."}
}
