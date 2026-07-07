---
name: input-cache-hit
description: Optimize LLM prompts and system instructions for maximum automatic prompt cache hit rates. Use when the user asks to improve cache efficiency, reduce API costs/latency, restructure prompts for caching, optimize system messages, or audit an existing prompt for cache-friendliness. Covers OpenAI-compatible APIs with automatic prefix caching.
---

# Input Cache Hit

## Overview

OpenAI-compatible APIs automatically cache the longest matching prefix of a prompt. When a cached prefix is reused across requests, those tokens cost 50% less and process faster. This skill provides concrete strategies to restructure prompts so the cached portion is as large as possible.

## Core Rule

**Static content at the top, dynamic content at the bottom.** The cache matches from position 0. Every token change in the prefix invalidates everything after it.

```
[LONG, STABLE CONTENT -- gets cached]
  -> System instructions, tool definitions, reference docs, examples
[SHORT, VARIABLE CONTENT -- no cache benefit needed]
  -> User query, current context, fresh data
```

## Prompt Structuring Patterns

### System Message Design

- Keep the system message identical across as many requests as possible.
- If you need per-request variations in system behavior, append a short dynamic suffix rather than rewriting the whole message.
- Separate persistent rules from per-task instructions: put persistent rules first.

**Cache-friendly:**
```
System: [2000 tokens of stable rules] + "\n\nFor this task: {dynamic_instruction}"
```

**Not cache-friendly:**
```
System: "For this task: {dynamic_instruction}\n\n[2000 tokens of stable rules]"
```

### Multi-Message Ordering

- Place `system` messages before `user` messages.
- When using `developer` messages (o-series models), place them first and keep them stable.
- In multi-turn conversations, append new user/assistant turns rather than rewriting history.

### Tool/Function Definitions

- List function definitions in a stable, alphabetical order.
- Provide full schemas in the system message (not per-turn).
- When a subset of tools is needed, use tool_choice to restrict selection rather than removing definitions.

### Content Attachment Strategy

When attaching documents or context:

1. Place shared reference material (docs, schemas, codebase snippets) before the user query.
2. If the same reference is used across many requests, make it part of the system message.
3. For frequently used codebase files, consider preloading them in a cached system prefix.

### Structured Output and JSON Mode

- Place `response_format` definitions as early as possible (system message).
- Keep JSON schema definitions stable and alphabetically sorted.

## Anti-Patterns

Avoid these common patterns that destroy cache hits:

| Pattern | Problem | Fix |
|---|---|---|
| Timestamps/IDs in system message | New value per request invalidates entire cache | Move to end of user message |
| Variable injection in middle of prompt | Splits the cache into two fragments | Move all variables to the end |
| Rewriting entire conversation history | Breaks prefix match every time | Append new turns only |
| Randomizing tool definition order | No cache hit even with identical tools | Sort alphabetically |
| Prepending user-specific data to shared instructions | Shared prefix lost | Append user data after shared content |

## Optimization Workflow

When asked to optimize a prompt for cache hits:

1. Identify static vs. dynamic content in the prompt.
2. Move all static content to the top (system message first).
3. Consolidate all dynamic/variable content at the bottom.
4. Remove any identifiers, timestamps, or counters from the static section.
5. Sort structured content (tools, schemas) deterministically.
6. Verify the prompt produces identical behavior after restructuring.

## Quick Checklist

- [ ] System message is identical across requests in the same session
- [ ] No timestamps, UUIDs, or counters in the cached prefix
- [ ] Dynamic variables appended at the end, not interspersed
- [ ] Tool definitions sorted and stable
- [ ] Multi-turn history only appended to, not rewritten
- [ ] Reference documents in system or early user message (not late)