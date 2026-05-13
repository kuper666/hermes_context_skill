---
name: hermes-retrieval-context
description: >-
  Modernize Hermes Agent chat context handling for Telegram and other gateway
  channels by replacing full session-history prompts with retrieval-based
  context assembly, source-filtered memory search, logical summarization triggers, and
  compact tail windows. Use when fixing bloated Hermes sessions, Telegram
  context drift, session_search behavior, ContextEngine plugins, memory
  providers, or prompt assembly in Hermes.
version: 1.0.0
author: operator
license: MIT
metadata:
  hermes:
    tags: [hermes, telegram, context-engine, memory, rag, session-search]
---

# Hermes Retrieval Context

## Goal

Change Hermes from session-based context to retrieval-based context:

```text
Telegram / Gateway -> Context Assembler -> Scoped Memory Retrieval -> LLM -> Memory Update
```

Core rule:

```text
Every LLM call must be built from scratch. Never replay channel history.
```

The active chat transcript remains the durable audit log. The model prompt must contain only:

- the system prompt and active skill/context files;
- a compact summary of relevant prior state;
- retrieved memories or past-session chunks that match the current request;
- the smallest recent message tail needed for conversational continuity;
- the new user message.

## First Steps

1. Read [RESEARCH.md](RESEARCH.md) before changing code.
2. Read [QUALITY_GUARDRAILS.md](QUALITY_GUARDRAILS.md) and treat it as the quality bar for hallucination prevention.
3. Locate the installed Hermes version and inspect the actual source before patching.
4. Find the prompt/context assembly path:
   - `run_agent.py`
   - `agent/context_engine.py`
   - `agent/context_compressor.py`
   - `agent/prompt_builder.py`
   - `gateway/run.py`
   - gateway Telegram session routing/session key code
   - `tools/session_search_tool.py`
5. Prefer the native Hermes extension points:
   - use a `ContextEngine` plugin when the version supports it cleanly;
   - otherwise add a small Context Assembler in the gateway/agent boundary.
6. Do not delete transcript persistence. Full chat history must remain searchable and auditable.
7. Treat Telegram as transport only. `chat_id` is a source/safety filter, not a semantic context boundary.

## Required Behavior

- Preserve all important prior context through durable summaries and searchable memory chunks.
- Stop sending whole gateway chat history to the LLM on every turn.
- Keep a configurable recent tail, defaulting to 6-12 user/assistant turns or less when the topic changes.
- Use platform/chat/thread/user identity to filter search sources and enforce isolation, not to decide semantic relevance.
- Support topic switches in the same Telegram chat without dragging unrelated prior topics into the prompt.
- Update memory after meaningful state changes, decisions, user preferences, completed work, or unresolved tasks.
- Keep tool call/result pairs valid whenever messages are trimmed or summarized.
- Use source-filtered transcript search for exact old details when summaries/retrieval do not contain enough evidence.
- Never answer exact recall questions from weak memory matches or guessed reconstruction.

## Implementation Workflow

Use [IMPLEMENTATION.md](IMPLEMENTATION.md) as the detailed checklist. High-level flow:

1. Audit current Hermes context flow and session persistence.
2. Identify channel source fields: platform, chat ID, thread/topic ID, user ID, session ID, lineage/root IDs.
3. Add a `ContextAssembler` that returns a fresh message list for each inbound user message.
4. Add logical summary checkpoints:
   - topic boundary;
   - task completed;
   - user explicitly changes topic;
   - tail token budget exceeded;
   - gateway session expiry/reset;
   - before context compression would drop details.
5. Store summaries/chunks in a persistent memory index with source identity metadata.
6. Retrieve only relevant chunks for the current request.
7. Build the final prompt from stable system context, retrieved context, recent tail, and current message.
8. Save the full turn to the transcript and update memory/index after the response.

## Context Assembly Contract

The assembled prompt must be deterministic and bounded:

```text
system prompt
active memory snapshot
retrieved source-filtered summaries, ordered by relevance and recency
optional current-topic working summary
recent tail messages
current user message
```

Hard limits:

- retrieved summaries: 2-6 chunks;
- recent tail: default 6-12 turns, configurable;
- unrelated old topic chunks: exclude unless the user explicitly asks to recall them;
- global search: opt-in only.
- source identity is a filter, not permission to include all channel history.

## Retrieval Rules

1. Search within the current channel source first, but include only semantically relevant chunks.
2. If confidence is low and the user asks about past work, expand to the same user across channels.
3. Use global search only when explicitly requested or clearly necessary.
4. Rank by semantic relevance, lexical match, recency, and unresolved-task flags.
5. Deduplicate chunks that represent the same decision or fact.
6. Include source metadata internally, but keep user-facing replies natural.
7. When retrieved context is low-confidence, expand source-filtered search or ask for clarification instead of guessing.

## Summarization Rules

Use [PROMPTS.md](PROMPTS.md) for summary schemas.
Use [QUALITY_GUARDRAILS.md](QUALITY_GUARDRAILS.md) for confidence, fallback, and exact-recall rules.

Summaries must capture:

- current goal;
- topic label;
- user preferences and constraints;
- decisions and rationale;
- files/services/accounts mentioned;
- resolved work;
- open tasks/blockers;
- exact critical values, commands, URLs, IDs, and errors when relevant.

Do not summarize secrets into memory. Store only references such as "API key exists in server env".

## Quality Bar

Do not mark implementation complete unless:

- there is a single auditable point where the final prompt is assembled;
- assembled prompt reports include token budget, tail size, retrieved chunk count, and source filter;
- low-confidence retrieval has explicit fallback behavior;
- full transcript recovery works for exact details;
- topic switches reduce unrelated context without losing recoverability.
- `chat_id` never acts as the semantic context boundary for prompt inclusion.

## Known Hermes Pitfalls

Check these before claiming the migration is complete:

- Telegram `session_search` may fall back to raw previews if auxiliary model/provider config is broken.
- `session_search` may search globally instead of using current Telegram channel identity as a source/safety filter.
- Compression-ended parent sessions may be excluded from search, creating a memory black hole.
- `MemoryProvider.on_pre_compress()` provider insights may be discarded in affected versions.
- Tool call/result pairs can become invalid if trimming splits them.
- Re-initializing memory providers on every gateway message can add avoidable latency.

## Validation Checklist

- Same Telegram chat, long conversation: prompt token usage stays bounded.
- Same Telegram chat, topic switch: old topic is not included unless requested.
- Same Telegram chat, unrelated old topic: `chat_id` alone does not cause inclusion in the prompt.
- User asks "what did we decide earlier?": relevant source-filtered summary is retrieved.
- User asks about exact old detail after compression: session search can recover it.
- User asks for an exact command/ID/error absent from summaries: agent searches source-filtered transcript instead of guessing.
- Group/thread chats: memory does not leak across users or threads.
- Restart Hermes/gateway: summaries and memory remain available.
- Broken auxiliary model config: failure is explicit, not silent raw-preview degradation.
- Full transcript remains persisted after every turn.

## Output Expectations

When applying this skill, report:

- files changed;
- chosen integration point and why;
- configured budgets;
- memory/index storage location;
- known risks left for the installed Hermes version;
- commands or tests run, if any.
