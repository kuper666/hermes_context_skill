# Hermes Retrieval Context Skill

This repository contains a standalone Hermes Agent skill for modernizing chat context handling in Telegram and other gateway channels.

The skill is designed for Hermes installations where long-running gateway chats gradually become expensive, slow, and semantically noisy because the agent keeps sending most of the current session history to the LLM.

## What This Skill Fixes

Hermes can treat the active session as the main context container. In long Telegram chats this creates several problems:

- prompt tokens grow with every message;
- old topics keep leaking into new topics in the same chat;
- summarization only reacts after context pressure appears;
- exact old details can be lost or hallucinated after compression;
- `session_search` and memory retrieval may be too global unless chat scope is enforced.

This skill guides a Hermes-capable coding agent to replace that behavior with a retrieval-based context pipeline.

## Target Behavior

Instead of this:

```text
new user message + full Telegram session history -> LLM
```

Hermes should move toward this:

```text
new user message
+ recent chat tail
+ current-topic working summary
+ scoped retrieved memory chunks
+ exact transcript recovery when needed
-> LLM
```

The full transcript remains persisted as the audit log. It is not deleted and should remain searchable for exact recovery.

## Why It Is Effective

The skill changes the source of context:

```text
session-based context -> retrieval-based context
```

This means each turn receives a bounded, freshly assembled prompt instead of inheriting the entire growing chat. In practice this should reduce input token usage significantly in long Telegram chats while preserving recall through scoped memory, summaries, and transcript search.

Expected impact:

- Long active tasks: often 50-85% less input context.
- Topic switches inside old Telegram chats: often 70-95% less unrelated context.
- Same main LLM call count per user turn in normal cases.
- Occasional auxiliary calls for summarization, topic detection, or memory updates.

Actual savings depend on the installed Hermes version, model context size, tool-output volume, and memory backend.

## Repository Layout

```text
skills/
  hermes-retrieval-context/
    SKILL.md
    RESEARCH.md
    IMPLEMENTATION.md
    QUALITY_GUARDRAILS.md
    PROMPTS.md
    ASSESSMENT.md
AGENT_TASK.md
LICENSE
README.md
```

## Installation

Clone this repository on the machine where Hermes or your coding agent can read it:

```bash
git clone https://github.com/kuper666/hermes_context_skill.git
```

Then copy only the skill directory into the skill location used by your Hermes setup:

```bash
cp -R hermes_context_skill/skills/hermes-retrieval-context /path/to/hermes/skills/
```

If your Hermes agent can load or inspect skills directly from a GitHub URL, give it this repository URL and tell it to read `AGENT_TASK.md` first.

## How To Ask Hermes To Apply It

Use this task with your Hermes coding agent:

```text
Read this repository:
https://github.com/kuper666/hermes_context_skill

Use the skill in skills/hermes-retrieval-context.
Read AGENT_TASK.md first.

Modernize the installed Hermes gateway chat context flow so Telegram and other gateway channels stop sending full session history to the LLM on every turn. Implement a bounded retrieval-based Context Assembler with scoped memory retrieval, recent tail preservation, logical summary checkpoints, transcript recovery for exact details, and explicit low-confidence fallback behavior.

Do not delete or truncate raw transcripts. Do not implement a broad rewrite. Inspect the installed Hermes version first, choose the least invasive integration point, and validate Telegram/gateway behavior.
```

## Quality Guardrails

This skill is intentionally conservative. It requires the implementing agent to protect answer quality by:

- keeping raw transcripts searchable;
- preserving a recent tail for conversational continuity;
- storing exact critical details in summary chunks;
- defaulting retrieval to the current chat/thread scope;
- running scoped transcript search before answering exact recall questions when summaries are insufficient;
- refusing to reconstruct exact old commands, IDs, errors, URLs, dates, or file paths from weak memory matches.

## When Not To Use

Do not use this skill for small, short-lived CLI-only sessions where full conversation history is cheap and useful. It is intended for persistent gateway chats, especially Telegram, where session history grows over time.

## License

MIT