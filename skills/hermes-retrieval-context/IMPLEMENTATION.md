# Hermes Retrieval Context Implementation

## Target Architecture

```text
Inbound gateway message
  -> resolve channel source identity
  -> append raw message to transcript
  -> detect topic/task boundary
  -> summarize completed logical chunk when needed
  -> retrieve relevant source-filtered memory chunks
  -> assemble bounded prompt
  -> call LLM
  -> persist assistant response
  -> update memory index
```

The transcript remains complete. The LLM never receives the full transcript by default.

Every LLM call must be built from scratch. Never replay channel history. Telegram `chat_id` is only a source/safety filter for search and isolation, not a semantic session boundary.

## Step 1: Inspect Installed Hermes

Identify the actual source layout before editing:

- Hermes version and commit.
- `run_agent.py` entry point for `run_conversation()`.
- context engine loading path.
- gateway Telegram handler and session key builder.
- session persistence database/schema.
- `session_search` implementation and accepted parameters.
- memory provider manager and lifecycle hooks.

Do not assume upstream line numbers match public issue reports.

## Step 2: Choose Integration Point

Preferred order:

1. `ContextEngine` plugin if the version supports custom engines and lifecycle hooks cleanly.
2. Gateway/agent boundary assembler if full prompt assembly is still hard-wired to session history.
3. Minimal patch to `run_agent.py` only if no clean extension point exists.

The chosen code path must run for Telegram and other gateway channels, not only CLI.

## Step 3: Define Source Identity

Create a normalized channel source identity using the same fields as session routing:

```text
platform
chat_type
chat_id
thread_id or topic_id
user_id
session_id
lineage_root_id
```

Use this identity for reads, writes, search filtering, and safety isolation. Do not use it as the semantic boundary for what enters the prompt. In group chats, include user isolation rules exactly as Hermes gateway does.

## Step 4: Add Context Assembler

The assembler should expose a simple contract:

```python
class ContextAssembler:
    def assemble(self, *, source_identity, user_message, recent_messages, system_message, token_budget):
        ...
```

Recommended output fields:

```python
{
    "messages": assembled_messages,
    "retrieved_chunks": retrieved_chunks,
    "topic": topic_label,
    "budget": budget_report,
}
```

Keep this component deterministic and testable without network calls.

The assembler is the quality gate. It must be the single place where the final prompt is built, budgeted, and logged. Do not leave a parallel code path that still forwards full gateway history during normal turns.

The assembler must decide prompt inclusion by current user intent, topic continuity, recency, and retrieval relevance. It must not include old content merely because it shares the same `chat_id`.

## Step 5: Message Tail Policy

Default policy:

- keep the latest 6-12 complete user/assistant turns;
- always keep unresolved tool call/result groups together;
- shrink tail when the topic classifier detects a clear topic switch;
- expand tail only when the new message contains references such as "that", "continue", "same issue", or "what you just said".

If a tail trim would break provider alternation rules, trim one more complete turn instead.

## Step 6: Logical Summary Triggers

Summarize and index a chunk when one of these happens:

- user changes topic;
- current task is completed;
- assistant produces a decision or final outcome;
- token budget for the recent tail is exceeded;
- gateway session is about to expire;
- user asks to remember a fact;
- Hermes compression would otherwise discard details.

Use cheap deterministic signals first. Use an auxiliary LLM only when needed for boundary classification or summarization.

## Step 7: Memory Index

Use the best store already available in the installed Hermes:

- existing memory provider if it supports source-filtered semantic search;
- SQLite FTS5 with embeddings sidecar if no provider is configured;
- `session_search` only as a recovery tool, not as the primary every-turn context source.

Each chunk should store:

```text
chunk_id
source identity
topic label
summary text
keywords
created_at / updated_at
source session IDs
turn range
status: active | resolved | archived
importance: low | normal | high
```

Never store secrets. Store references to secret locations instead.

## Step 8: Retrieval Policy

For every inbound message:

1. Build a compact query from the current user message and the current topic label.
2. Filter search by current channel source identity first.
3. If the user explicitly asks about broader history, search same user across channels.
4. Use global search only for explicit global recall requests.
5. Rank results by semantic relevance, lexical match, recency, unresolved status, and topic match.
6. Return 2-6 chunks within budget.
7. Deduplicate facts and prefer newer resolved decisions over older guesses.

Return retrieval confidence with the assembled context. Low confidence must not be treated as valid memory. If confidence is low, expand source-filtered retrieval, search raw transcripts for exact details, or ask a concise clarification.

## Step 9: Prompt Shape

Assemble in this order:

```text
system message
memory snapshot from Hermes prompt builder
retrieved source-filtered context block
current-topic working summary
recent tail
current user message
```

The retrieved context block should be marked as reference context, not as user-authored instructions. It must not override system/developer instructions.

If the user asks for exact previous wording, commands, IDs, file paths, URLs, dates, or errors, the prompt must include either the exact retrieved source or a note that the exact detail was not found. Do not let the assistant reconstruct exact values from a summary.

## Step 10: Post-Response Update

After each assistant response:

- persist the raw turn to the normal Hermes session store;
- update chunk status when a task is resolved;
- create a new summary chunk for important new decisions;
- refresh the current-topic working summary;
- flush memory/index writes before returning success to the gateway.

If summary or memory update fails, persist the raw turn first and mark the chunk for retry. Never silently drop a conversation chunk because summarization failed.

## Step 11: Telegram Validation

Validate these scenarios manually or with integration tests:

- Long Telegram DM continues past the old compression threshold without full-history prompts.
- User switches from "server deployment" to "vacation planning"; old server details do not enter prompt.
- Old content from the same `chat_id` does not enter the prompt unless it matches current intent.
- User later asks "what did we decide about deployment?"; relevant server summary is retrieved.
- User asks for an exact error from an old compressed turn; session search can recover it.
- User asks for an old command that is absent from summaries; source-filtered transcript search runs before the answer.
- Telegram group topic A does not see memory from topic B.
- Two different users in the same group do not share private memory unless Hermes is configured for shared group memory.

## Step 12: Regression Tests

Add focused tests for:

- assembler budget enforcement;
- source filtering without treating `chat_id` as semantic context;
- topic switch tail shrink;
- tool call/result pair preservation;
- retrieval ranking;
- retrieval confidence fallback behavior;
- compression-ended parent searchability;
- auxiliary model failure path for summarization.

Do not mark the migration done if tests only cover CLI. Gateway coverage is required.
