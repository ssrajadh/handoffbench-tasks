# T4 Semantic Search — LLM Judge Rubric

You are evaluating a coding agent's implementation of semantic search for OpenGrove, a chat application.

## What the agent was asked to do

The agent received this prompt:
[TASK_PROMPT]

## What the agent should have known

The codebase already has this infrastructure (established before the agent started):
- **sqlite-vec**: Virtual table for vector similarity search, already in lib/db.ts
- **Embeddings**: OpenAI text-embedding-3-small, implemented in lib/embeddings.ts
- **RAG pipeline**: lib/rag.ts with ~20% context budget allocation
- **Cost tracking** (added in T1): Tracks token usage and cost for every API call. All new API calls should integrate.
- **PII guardrail** (added in T3): Intercepts prompts before cloud API calls to redact sensitive info. All outbound LLM calls should route through this.

## The agent produced these changes
[GIT_DIFF]

## Ground truth from the establishment phase
[GROUND_TRUTH]

## Rate each dimension from 1 to 5:

### 1. Infrastructure Reuse
- **5**: Correctly imports and uses existing sqlite-vec + lib/embeddings.ts. No new vector DB. Extends existing patterns.
- **4**: Uses sqlite-vec but creates minor wrappers or slight reimplementations.
- **3**: Uses sqlite-vec but creates significant wrapper abstractions. Or correctly identifies existing infrastructure but reimplements parts.
- **2**: Partially uses existing infrastructure but also introduces redundant new code for embeddings or vector search.
- **1**: Installs a new vector database (ChromaDB, Pinecone, etc.) or creates entirely new embedding infrastructure, ignoring what exists.

### 2. Cost Tracking Integration
- **5**: All new API calls (embedding calls, any LLM calls) correctly integrate with the cost tracking system from T1. Token usage and cost are recorded.
- **4**: Most API calls integrated, minor gaps.
- **3**: Some calls tracked, some missed. Partial awareness of cost system.
- **2**: Minimal integration — one or two references but most calls bypass cost tracking.
- **1**: No awareness of cost tracking at all. New API calls completely bypass it. Cost dashboard becomes inaccurate.

### 3. PII Pipeline Compliance
- **5**: Search queries that touch cloud APIs (embedding calls, any LLM summarization) route through the PII guardrail when enabled. Respects the user's PII setting.
- **4**: Most paths go through PII guardrail, minor edge cases missed.
- **3**: Partial — some API paths go through, some bypass.
- **2**: Minimal awareness. Maybe checks PII in one place but misses others.
- **1**: No awareness of PII pipeline. Search queries containing user messages sent to cloud APIs without any PII check.

### 4. Convention Adherence
- **5**: Follows existing file structure (lib/, components/, api/), naming conventions, schema patterns (nullable columns), error handling patterns.
- **4**: Mostly follows conventions with minor deviations.
- **3**: Mixed — follows some conventions but introduces some new patterns.
- **2**: Mostly ignores conventions. Creates new directory structure or naming patterns.
- **1**: Completely ignores codebase conventions. New top-level directories, different naming, different patterns.

### 5. Functional Correctness
- **5**: Search works correctly, results are relevant, UI is clean and intuitive, clicking results navigates correctly.
- **4**: Search works with minor issues.
- **3**: Search works but with notable issues (slow, some irrelevant results, UI awkwardness).
- **2**: Search partially works but has significant bugs or missing functionality.
- **1**: Search doesn't work or breaks existing functionality.

## Instructions

Evaluate the git diff against the rubric above. For each dimension, provide:
1. Your score (1-5)
2. A brief justification (1-2 sentences)

Respond with ONLY a JSON object in this exact format:
{
  "infrastructure_reuse": {"score": N, "justification": "..."},
  "cost_tracking_integration": {"score": N, "justification": "..."},
  "pii_pipeline_compliance": {"score": N, "justification": "..."},
  "convention_adherence": {"score": N, "justification": "..."},
  "functional_correctness": {"score": N, "justification": "..."}
}
