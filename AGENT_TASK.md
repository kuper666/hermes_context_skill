# Task For Hermes Agent

Read this file first when this repository is provided to a Hermes-capable coding agent.

## Mission

Modernize the installed Hermes Agent gateway chat context flow so Telegram and other gateway channels no longer send full session history to the LLM on every user message.

Core rule:

```text
Every LLM call must be built from scratch. Never replay channel history.
```

Use the skill located at:

```text
skills/hermes-retrieval-context
```

## Required Reading Order

1. `skills/hermes-retrieval-context/SKILL.md`
2. `skills/hermes-retrieval-context/RESEARCH.md`
3. `skills/hermes-retrieval-context/QUALITY_GUARDRAILS.md`
4. `skills/hermes-retrieval-context/IMPLEMENTATION.md`
5. `skills/hermes-retrieval-context/PROMPTS.md`
6. `skills/hermes-retrieval-context/ASSESSMENT.md`

## Implementation Objective

Replace session-based prompt context with retrieval-based context assembly:

```text
Telegram / Gateway -> Context Assembler -> Scoped Memory Retrieval -> LLM -> Memory Update
```

The transcript remains complete and persisted. The main LLM prompt receives only the minimal context needed for the current turn:

- system prompt and active Hermes memory snapshot;
- source-filtered retrieved memory chunks;
- current-topic working summary;
- recent tail messages;
- current user message.

Telegram is transport, not the meaning of the conversation. `chat_id` and thread IDs are source/safety filters for search and isolation; they are not semantic session boundaries.

## Hard Requirements

- Do not delete, truncate, or stop writing raw session transcripts.
- Do not send full gateway chat history by default in normal turns.
- Never use `chat_id` as a context boundary. Use platform/chat/thread/user identity only to filter search sources and enforce safety/isolation.
- Global recall must be explicit, not accidental.
- Preserve recent tail and valid tool call/result pairs.
- Store summaries/chunks with source identity, topic, source session IDs, and turn ranges.
- Use source-filtered transcript/session search for exact old commands, IDs, errors, URLs, dates, file paths, and prior decisions when summaries are insufficient.
- Do not reconstruct exact old values from weak summaries or low-confidence retrieval.
- Make auxiliary summarization/retrieval failures explicit and safe.

## Suggested Work Plan

1. Inspect the installed Hermes source and identify the actual version.
2. Locate gateway Telegram session routing and the place where prompt messages are assembled.
3. Check whether `ContextEngine` plugins are supported in this version.
4. Choose the least invasive integration point:
   - `ContextEngine` plugin if it can control the relevant gateway path;
   - otherwise a gateway/agent boundary `ContextAssembler`;
   - patch `run_agent.py` only if no clean extension point exists.
5. Implement bounded context assembly with a budget report.
6. Add source-filtered memory retrieval and logical summary checkpoints.
7. Add exact-recall fallback through source-filtered transcript/session search.
8. Add tests or manual verification for Telegram/gateway behavior.

## Done Criteria

The task is complete only when:

- a normal Telegram turn no longer forwards full session history to the LLM;
- long Telegram chats have bounded prompt token usage;
- topic switches do not pull unrelated old topics into the prompt;
- "what did we decide earlier?" searches the current channel source safely, but includes only chunks relevant to the current user intent;
- exact recall questions use transcript search when memory is insufficient;
- broken auxiliary model config does not silently produce raw-preview memory as if it were valid;
- compressed parent sessions remain searchable or have an equivalent exact recovery path;
- the final report lists changed files, integration point, budgets, memory/index storage, and remaining version-specific risks.
