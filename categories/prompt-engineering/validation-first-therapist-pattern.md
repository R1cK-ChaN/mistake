---
date: 2026-03-22
severity: medium
tags: [prompt-engineering, reliability]
---

# Bot opens with "完全理解" — therapist-style active listening breaks persona

## What happened

The bot frequently started replies with validation phrases: "完全理解", "我懂你的意思", "这很正常". This is classic therapist-style active listening — the exact opposite of the "real friend texting" persona. No friend texts you "完全理解你的感受" before responding. They just respond.

The pattern was invisible to all existing detection because nobody had explicitly flagged it. There was no keyword check, no reranker penalty, no style hint — it was a blind spot.

## Root cause

LLM default behavior. Models are trained on helpful-assistant data where "I understand your concern" is a standard opening. Without explicit suppression, this pattern leaks into any persona that involves responding to emotional content. The companion prompt said "don't be a therapist" but never named this specific surface form.

This is the flip side of the "negative constraints don't work" lesson: sometimes you DO need to name specific patterns to ban, because the model generates them reflexively. The key is to use bans as **one layer** of defense, not the only layer.

## Fix / Lesson

Applied defense-in-depth — five layers targeting the same pattern:

```python
# 1. Detection list
_VALIDATION_STARTERS = [
    "完全理解", "我懂你的意思", "这很正常", "我能理解",
    "你的感受很合理", "我明白你", "确实不容易",
]

# 2. Style hints — inject when pattern detected in recent history
if any(s in recent_bot_replies for s in _VALIDATION_STARTERS):
    inject_hint("不要以理解/认同对方感受开头，直接说你要说的")

# 3. Post-processing — strip as format cleanup
def strip_validation_starter(text: str) -> str:
    for starter in _VALIDATION_STARTERS:
        if text.startswith(starter):
            text = text[len(starter):].lstrip("，,。. ")
    return text

# 4. Reranker penalty
if candidate.startswith_any(_VALIDATION_STARTERS):
    candidate.score -= VALIDATION_PENALTY

# 5. Soul prompt — positive example showing the alternative
"直接回应，不要先验证对方的情绪。
 NO: '完全理解，加班确实辛苦' → YES: '几点了这是'"
```

Takeaways:

- **Some LLM defaults are so deeply trained that you need explicit, named bans.** "Don't be a therapist" is too abstract. "Don't start with 完全理解/我懂你的意思" is specific enough for the model to follow.
- **Defense-in-depth for persistent patterns.** A single ban layer isn't enough for deeply trained defaults. Detection + hints + post-processing + reranker + prompt examples — redundancy is the point.
- **Validation-first is the #1 tell that you're talking to an AI.** Real friends don't preface every response with "I completely understand your feelings." Eliminating this pattern has outsized impact on perceived authenticity.
- **Audit for blind spots by reading raw transcripts.** This pattern was invisible to all automated checks because nobody thought to look for it. Regular manual review of full conversations catches patterns that no detector covers yet.
