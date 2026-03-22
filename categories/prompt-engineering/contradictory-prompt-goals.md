---
date: 2026-03-22
severity: medium
tags: [prompt-engineering]
---

# Soul prompt has contradictory goals — model oscillates between them

## What happened

The companion bot's soul prompt simultaneously required:
- "warm but not oily"
- "peer-flat, don't manage the other person's emotions"

For an LLM, "warm + observant" almost inevitably produces emotional responses. The model oscillated between being warm (which triggered emotional caretaking) and being peer-flat (which killed warmth). It couldn't satisfy both constraints at once because they're mutually exclusive for the model's generation process.

## Root cause

Each instruction made sense in isolation. Together they created a paradox. The prompt designer understood the nuance ("warm like a friend, not warm like a therapist") but the model can't parse that distinction from adjectives alone. "Warm" in LLM-space is strongly associated with emotional support behaviors.

## Fix / Lesson

Replaced contradictory adjectives with **concrete behavioral specs**:

```
# Before — contradictory adjectives
"Be warm but not oily. Be peer-flat, don't manage emotions."

# After — behavioral spec with examples
"Default energy: medium-high. Show warmth through enthusiasm about topics,
not through emotional caretaking.

YES: '哈哈哈不是吧这也行' (warm via energy)
NO:  '辛苦了，注意休息' (warm via caretaking)"
```

Takeaways:

- **If two instructions can't both be satisfied in a single generation, the model will oscillate.** Test every pair of style instructions for mutual exclusivity.
- **Adjectives are ambiguous to LLMs.** "Warm" means different things in different contexts. Replace adjectives with behavioral examples that show exactly what you mean.
- **The prompt designer's mental model ≠ the model's interpretation.** You know "warm but not oily" means "friend-warm, not therapist-warm". The model doesn't. Spell it out.
