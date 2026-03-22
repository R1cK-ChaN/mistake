---
date: 2026-03-22
severity: high
tags: [prompt-engineering, tool-use, reliability]
---

# Prompt says "act like you casually know this" — model ignores search results and hallucinates

## What happened

The bot called `web_search`, got real results back, then **completely ignored them**. It fabricated store names, prices, and addresses — 4 consecutive turns of self-contradicting hallucinations. The tool worked. The data was there. The model just didn't use it.

Simultaneously, the hallucination detector had a blind spot: `if tool_audit: return False` — if any tool was called, it assumed the response was grounded. So the detector saw "web_search was called" and waved through a response full of fabricated data.

## Root cause

The soul prompt said: "像碰巧知道这个地方一样随口说" (mention it casually, as if you happen to know). This was meant to make tool-assisted responses sound natural — not like "According to my search results..." But the model interpreted it as **permission to not cite the data at all**. "Act like you casually know" + actual search results = model generates from its own knowledge and ignores the results entirely.

Two contradictory instructions:
1. Tool system: "Here are search results, use them"
2. Soul prompt: "Talk as if you casually know this, don't reference sources"

The model resolved the contradiction by prioritizing the persona instruction (don't reference sources) over the tool data (use these results). The persona gave it permission to hallucinate.

The detector made it worse: `if tool_audit: return False` meant "if a tool was called, the response is grounded by definition." This is logically wrong — calling a tool doesn't mean the model used the results.

## Fix / Lesson

### 1. Rewrote grounding prompt — data first, tone second

```
# Before — persona overrides data
"像碰巧知道这个地方一样随口说"

# After — data is mandatory, tone is optional
"搜索结果返回什么就用什么，没有就说没查到。
 你可以用随意的语气转述，但信息必须来自搜索结果，不能自己编。"
```

The key distinction: **tone can be casual, data cannot be fabricated.** These are independent dimensions — you can sound natural while still being faithful to the source.

### 2. Fixed hallucination detector — check content, not just tool call

```python
# Before — tool called = grounded (wrong)
if tool_audit:
    return False  # blindly skip

# After — check if response actually uses tool results
def check_grounding(response: str, tool_results: list[dict]) -> bool:
    if not tool_results:
        return True  # no tool called, normal hallucination check

    # Tool was called — verify response contains substance from results
    result_content = extract_key_facts(tool_results)
    if not any(fact in response for fact in result_content):
        trigger_warning("Tool called but results not reflected in response")
        return False
    return True
```

Takeaways:

- **"Act like you casually know X" is a hallucination prompt in disguise.** It tells the model to not reference its source — which the model interprets as "generate from internal knowledge." If you want casual tone, say "use casual tone when presenting search results", not "act like you already knew this."
- **Persona instructions and tool grounding instructions will conflict. Grounding must win.** Explicitly state: "data accuracy overrides persona tone. Never fabricate facts to sound more natural."
- **"Tool was called" ≠ "response is grounded."** A hallucination detector that skips checks when a tool was called has a massive blind spot. Check whether the response actually reflects the tool results, not just whether a tool was invoked.
- **This is the same "contradictory prompt" pattern** as warm-but-peer-flat. Two instructions that make sense individually but are mutually exclusive in practice. Always test: "if the model can only obey one of these, which one wins?" and make that explicit.
