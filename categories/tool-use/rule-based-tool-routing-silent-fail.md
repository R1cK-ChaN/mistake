---
date: 2026-03-22
severity: high
tags: [tool-use, agent-orchestration, hallucination]
---

# Rule-based tool routing silently fails and causes hallucination

## What happened

Used keyword-matching rules to decide which tool to call. The hit rate was too low — users phrase the same intent in countless ways, and rules can never cover them all. When no keyword matched, the router silently skipped tool calling entirely. The LLM had no idea a tool should have been used, so it hallucinated an answer instead of saying "I don't know" or calling the right tool.

The worst part: it was a **silent failure**. No error, no log, no fallback — just confidently wrong answers.

## Root cause

Assumed user queries could be bucketed by keywords. They can't. Natural language is too varied. Rule-based routing is brittle by nature — every new edge case demands a new rule, and you're always playing catch-up.

The deeper mistake: treating tool selection as a deterministic classification problem instead of a reasoning problem.

## Fix / Lesson

Switched to **LLM-driven tool selection**. All tools are registered with clear descriptions in a tools registry. The model reads the descriptions and decides which tool to call based on the user's intent — no hardcoded keyword matching.

Key takeaways:

- **Let the LLM reason about tool selection** — that's what it's good at. You already trust it to generate responses; trust it to pick tools too.
- **Every tool needs a good description** — the description IS the routing logic now. Invest time writing clear, specific tool descriptions with usage examples.
- **Silent failures are worse than loud errors** — if a tool router falls through with no match, it should at minimum log a warning, not silently proceed without tools.
- **Rule-based routing doesn't scale** — it works for 3 tools with distinct keywords, it breaks at 10+ tools with overlapping domains.
