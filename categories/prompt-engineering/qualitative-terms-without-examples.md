---
date: 2026-03-22
severity: medium
tags: [prompt-engineering, eval-and-testing]
---

# "Medium edge, not abrasive" — qualitative terms everyone interprets differently

## What happened

The bot's personality spec said "medium edge, not abrasive." Every team member had a different mental model of what this meant. When evaluating bot outputs, one person rated a reply as "perfect edge" while another flagged it as "too aggressive." No way to resolve disagreements because the standard was a subjective adjective.

## Root cause

Qualitative terms like "medium edge", "a bit sarcastic", "slightly playful" are not specifications — they're vibes. They work in human conversation because humans share cultural context. They fail as engineering specs because:

1. Different people calibrate "medium" differently
2. The LLM's calibration doesn't match any human's
3. There's no way to test or measure compliance

## Fix / Lesson

Replaced qualitative terms with **calibration examples** — specific conversation pairs that define the boundaries:

```
# Before
"medium edge, not abrasive"

# After — boundary examples
TOO SOFT:  User: "我又迟到了" → Bot: "没事啦下次早点"
TARGET:    User: "我又迟到了" → Bot: "你是不是闹钟放太远了"
TOO HARD:  User: "我又迟到了" → Bot: "你每次都这样 真服了"
```

Takeaways:

- **Any qualitative behavior spec needs at least 3 calibration examples**: too little, target, too much. Without these, the spec is unenforceable.
- **Calibration examples are also the eval criteria.** When team members disagree on a rating, point to the examples — not to adjectives.
- **This applies to any LLM persona, not just companion bots.** "Professional but friendly", "concise but thorough" — all of these need boundary examples or they're meaningless.
