---
date: 2026-03-22
severity: high
tags: [reliability, tool-use, prompt-engineering]
---

# LLM confidently fabricates facts rather than saying "I don't know"

## What happened

In a group chat, user asked "明天为什么图书馆关门". The bot confidently said "Good Friday" — it was actually Hari Raya Puasa. When corrected, the bot said "记岔了" (got mixed up) — pretending it had real knowledge but just made a mistake. This was **worse than the original error** because it implied the bot actually knows things and occasionally slips, rather than admitting it was guessing.

## Root cause

This is the deepest LLM habit: the model is trained on text where speakers state facts confidently. "I don't know" appears rarely in training data relative to confident assertions. No amount of prompt engineering alone can fully override this — the model will always have a bias toward producing an answer over producing uncertainty.

The "记岔了" follow-up is especially dangerous: the model learned that humans save face by framing errors as memory lapses. It's mimicking a social behavior that only makes sense if you actually have memories to get wrong.

## Fix / Lesson

Three-layer defense — no single layer is sufficient:

```
# Layer 1: Tool — provide real data so the model doesn't have to guess
tools = [system_clock, google_places, web_search, holiday_calendar]

# Layer 2: Prompt — explicit instruction
"If you are not certain about a fact (date, price, name, hours, event),
say you don't know or suggest checking. NEVER guess. NEVER fabricate.
Being wrong destroys trust faster than being unhelpful."

# Layer 3: Runtime validation
if response_contains_specific_facts and no_tool_called_this_turn:
    trigger_warning("Response contains facts but no tool was used")
    # Flag for review or inject tool call
```

Takeaways:

- **No single defense works against hallucination.** Prompt alone won't stop it (model ignores under pressure). Tools alone won't stop it (model may not call them). Runtime checks alone catch it too late. You need all three.
- **"I don't know" is a feature, not a failure.** A friend who says "不确定诶 你查一下" is more trustworthy than one who confidently says the wrong holiday. Frame uncertainty as a personality trait, not a limitation.
- **The "记岔了" pattern is uniquely corrosive.** It implies memory and knowledge the model doesn't have. If the bot can't know something, it should never frame errors as memory lapses. It should say "我不知道", not "我记错了".
