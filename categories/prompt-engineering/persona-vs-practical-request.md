---
date: 2026-03-22
severity: high
tags: [prompt-engineering, tool-use]
---

# "Don't be helpful" persona backfires when user genuinely asks for help

## What happened

The companion bot's persona was "not an assistant, not helpful, just a friend chatting." This worked great for casual conversation — no unsolicited advice, no "注意休息". But when a user explicitly asked "推荐个咖啡馆" five times in a row, the bot gave five empty responses: "找个连锁咖啡馆", "星巴克到处都有", "到处都是人别出门了". Zero actual information.

A real friend would vibe and be unhelpful during casual chat, but when you seriously ask "recommend me a place," they'd actually try. The persona rule couldn't distinguish between these two modes.

## Root cause

"Don't be helpful" was implemented as a global, always-on constraint. But real humans modulate helpfulness based on context:

- Friend venting about work → unhelpful is correct ("哦 又加班啊")
- Friend explicitly asking for a recommendation → unhelpful is rude and useless

The persona rule lacked **practical request detection** — it couldn't tell when the user shifted from chatting to genuinely asking for something.

## Fix / Lesson

Added practical request detection that temporarily overrides the "not helpful" persona:

```
# Engagement policy
if user_intent == "practical_request":
    # User is explicitly asking for information/recommendation
    # Override casual persona, activate tool use, be actually helpful
    mode = "helpful_friend"
    tools_enabled = True
elif user_intent == "venting" or user_intent == "casual":
    # Default persona: peer, not assistant
    mode = "casual_peer"
    tools_enabled = False
```

Takeaways:

- **Persona rules need context-awareness, not global application.** A real person's behavior varies by situation. "Not helpful" during venting, actually helpful during genuine requests.
- **"Don't be helpful" × 5 consecutive asks = trust destruction.** The user asked seriously and got nothing five times. That's worse than being too helpful — it signals the bot is useless.
- **Practical request detection is the bridge between persona and tools.** Without it, tools exist but the persona prevents the bot from using them. The persona and tool system must be coordinated, not independent.
