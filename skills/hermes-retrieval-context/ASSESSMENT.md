# Expected Result Assessment

## Expected Impact

If applied correctly, this upgrade should change Hermes gateway behavior from "send most of the session" to "assemble only what is relevant for this turn".

Expected practical result:

- prompt tokens in long Telegram chats become stable instead of growing with every message;
- the agent can continue a topic using recent tail and source-filtered summaries;
- old topics stop leaking into new topics in the same chat;
- important prior decisions remain recoverable through memory retrieval or `session_search`;
- full transcripts remain available for audit and exact recovery.

## Likely Token Savings

For long-running Telegram chats, expected input token reduction is high:

- short active topic: 70-95% less input context compared with full session history;
- long active task with tools/logs: 50-85% less input context, depending on retrieved chunks and tail size;
- topic switch after long prior thread: close to full reset behavior while preserving searchable memory.

Actual savings depend on Hermes version, model context size, tool-output volume, and memory index quality.

## Quality Risks

Primary risks:

- over-aggressive summarization can lose exact values;
- weak topic detection can omit useful recent context;
- global retrieval can reintroduce unrelated prior topics;
- broken auxiliary model config can silently degrade summaries;
- memory provider hooks may not behave as documented in affected Hermes versions.
- low-confidence retrieval can produce confident but unsupported recall answers if not blocked.

Mitigation:

- preserve recent tail;
- store exact critical details in summaries;
- keep raw transcripts searchable;
- use channel identity as a strict source/safety filter, not as a semantic context boundary;
- add explicit failure paths for unavailable summarization.
- require source-filtered transcript search before answering exact recall questions when summaries are insufficient.

## Readiness Criteria

Consider the implementation ready only when:

- gateway Telegram tests pass, not just CLI tests;
- channel identity is used for source/safety filtering by default;
- default retrieval target is current user + current task/topic;
- `chat_id` alone never causes old content to enter the prompt;
- compressed parent sessions remain searchable;
- prompt assembly has a clear token budget report;
- retrieval confidence and fallback behavior are implemented;
- memory/index persists across gateway restart;
- no full-session prompt path remains in normal gateway turns.

## Overall Confidence

Expected result: strong, provided the installed Hermes version exposes either `ContextEngine` plugins or a clean gateway/agent boundary for replacement.

If Hermes requires invasive `run_agent.py` changes, the result is still achievable, but regression risk is higher and tests around tool-call ordering and session persistence become mandatory.
