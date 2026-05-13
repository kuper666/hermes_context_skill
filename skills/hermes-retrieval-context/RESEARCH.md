# Hermes Retrieval Context Research

## Public Hermes Architecture Notes

Hermes already has several building blocks that should be reused instead of bypassed:

- `agent/context_engine.py` defines the `ContextEngine` extension point.
- `agent/context_compressor.py` is the default lossy compressor.
- `gateway/run.py` includes gateway session hygiene near the agent boundary.
- `run_agent.py` owns the agent loop, message assembly, compression checks, tool dispatch, and persistence.
- `agent/prompt_builder.py` builds the stable system prompt from memory, skills, context files, and personality.
- `session_search` can search persisted sessions and should be used for exact recovery from old transcripts.

The built-in compression model is still session-oriented. It reduces message volume only after pressure appears, while the desired architecture assembles a small context from scratch on every turn.

## Existing Context Compression

Hermes uses two compression layers:

- Gateway session hygiene at roughly 85% of model context length.
- Agent `ContextCompressor` at a configurable threshold, commonly 50%.

The default compressor protects recent messages, summarizes middle turns, preserves tool call groups, and inserts a structured summary. This prevents API failures, but it does not solve topic drift in a single Telegram chat because the active session remains the primary context container.

## ContextEngine Plugin Path

Current Hermes documentation describes `plugins/context_engine/<engine>/` as the preferred plugin path for replacing the built-in compressor. A context engine can implement:

- `should_compress()`
- `compress()`
- `update_from_response()`
- optional lifecycle hooks such as `on_session_start()` and `on_session_end()`
- optional agent-callable tools through `get_tool_schemas()` and `handle_tool_call()`

Use this when the installed Hermes version supports it reliably. If not, keep the same contract but wire a small `ContextAssembler` at the gateway/agent boundary.

## Relevant Community Issues

### Telegram session summarization raw-preview fallback

Reported behavior: Telegram sessions can show `[Raw preview - summarization unavailable]` even when JSONL transcripts exist. The likely cause is auxiliary model/provider resolution for `session_search`.

Implementation impact:

- Verify auxiliary model config for `session_search`.
- Fail visibly when summarization cannot run.
- Do not treat raw previews as valid durable summaries.

### `session_search` should be scoped to current chat

Reported behavior: gateway conversations may search globally across all sessions rather than the current Telegram DM, topic, Discord thread, or Slack thread.

Implementation impact:

- Retrieval must default to current chat/thread scope.
- Global retrieval must be explicit.
- Scope should use the same identity fields as gateway session routing, not only `user_id`.

### Compression-ended parents excluded from search

Reported behavior: after compression, parent sessions can be skipped by `session_search`, even though their original details are no longer in active context.

Implementation impact:

- Compression-ended parent sessions must remain searchable.
- Delegation parents and active visible context can still be excluded to avoid duplicates.
- Add regression coverage for "recover exact old detail after compression".

### `MemoryProvider.on_pre_compress()` return value discarded

Reported behavior: some Hermes versions call memory providers before compression but discard returned provider insights.

Implementation impact:

- Check whether the installed version captures provider context.
- If using memory providers, ensure provider insights enter summary/retrieval prompts.
- Preserve `focus_topic` when patching compression signatures.

### Memory provider re-initialization overhead

Reported behavior: gateway mode can create a new `AIAgent` on each inbound message, reinitializing memory providers and adding latency.

Implementation impact:

- Avoid expensive per-message initialization in the new assembler.
- Cache embedding clients/index handles safely per process.
- Keep writes durable and flush on gateway/session lifecycle events.

## Design Consequences

The practical target is not "better compression". It is a different source of truth for context:

```text
Full transcript: durable log
Memory index: searchable knowledge
Recent tail: continuity only
Prompt: compact assembled view
```

The implementation should keep the transcript complete while making prompt assembly retrieval-first.
