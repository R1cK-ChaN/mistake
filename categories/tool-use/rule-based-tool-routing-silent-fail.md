---
date: 2026-03-22
severity: high
tags: [tool-use, agent-orchestration, hallucination]
---

# Rule-based tool routing silently fails and causes hallucination

## What happened

Had a Google Places API integration for location queries (shops, restaurants, nearby places). But the model never knew it existed — there was only one exposed tool (`web_search`) with a keyword router sitting behind it that silently dispatched to different backends.

The keyword list for Places routing included things like "restaurant", "cafe", "hotel" — but missed everyday terms like "supermarket", "FairPrice", "near me", "nearby". Every query that didn't hit a keyword fell through to web search. The model had no idea a Places tool even existed, so it couldn't choose it.

Result: **all location queries got garbage web search results** instead of structured Places API data. The model then worked with those garbage results — confidently recommending places with wrong addresses, missing hours, no ratings. A silent failure that looked like it was working.

## Root cause

The architecture hid tools behind a keyword router instead of exposing them directly to the model. Two compounding problems:

1. **Keyword hit rate is inherently too low.** Natural language is too varied — users say "where can I buy groceries", "any supermarket around", "FairPrice near Tampines", none of which matched the keyword list. Every edge case demands a new rule, and you're permanently behind.

2. **The model couldn't reason about tool selection because it didn't know the tools existed.** It saw one tool (`web_search`) and used it for everything. The routing logic was invisible to the LLM — a black box doing bad classification that the model couldn't correct or override.

The deeper mistake: treating tool selection as a deterministic classification problem instead of a reasoning problem. The LLM is better at understanding intent than any keyword list.

## Fix / Lesson

Ripped out the keyword router entirely. Split into two independent tools with clear descriptions:

```
# Before — one tool, hidden router
tools = [
    {
        "name": "web_search",
        "description": "Search the web"
        # keyword router silently dispatches behind the scenes
    }
]

# After — two tools, model decides
tools = [
    {
        "name": "search_places",
        "description": "Search for nearby places, shops, restaurants, supermarkets, or any physical location. Returns structured info: name, address, rating, reviews, price range, opening hours, Google Maps link. Use this for any query about finding or recommending a physical place."
    },
    {
        "name": "web_search",
        "description": "Search the web for general information, news, articles, how-to guides, or anything not about finding a specific physical place."
    }
]
```

The model immediately started routing correctly — "FairPrice near me" goes to `search_places`, "how to cook laksa" goes to `web_search`. Zero keyword maintenance.

Key takeaways:

- **Never hide tools behind a router the model can't see.** If the model doesn't know a tool exists, it can't use it. Expose every tool directly with a clear description.
- **Tool descriptions ARE the routing logic.** Invest time writing specific, example-rich descriptions — the model uses them to decide. A good description replaces hundreds of keyword rules.
- **Keyword routers fail silently at scale.** They work for a demo with 3 perfect queries. They break in production where users phrase things in ways you never anticipated.
- **Silent fallback is the worst failure mode.** When the router fell through to web search, there was no error, no log — just degraded quality that looks like the system is working. At minimum, log when routing falls through to a default.
- **Split tools by domain, not by implementation.** The original sin was wrapping two completely different backends (Places API, web search) in one tool. Different data sources = different tools.
