---
date: 2026-03-22
severity: high
tags: [prompt-engineering, agent-orchestration]
---

# "Don't do X" rules don't work — model finds alternative paths around every ban

## What happened

The companion bot had dozens of negative rules: don't say 确实, don't use 文艺腔, don't give advice, don't end with a summary. The model obeyed the letter and violated the spirit every time.

Banned "赶紧去补一杯" (overt advice) → model switched to "找个背光的位置坐吧", "至少眼睛能放松点", "不想动就待着吧". Same steering behavior, softer wording. Every patch created a new variant.

## Root cause

LLMs process "don't do X" by first activating the representation of X, then trying to suppress it. This is structurally weaker than "do Y" which directly guides generation. Negative constraints are also unbounded — there are infinite ways to rephrase the banned behavior, and each ban only covers one surface form.

The deeper issue: we were doing **subtraction on the wrong distribution**. The model's base behavior was "helpful warm assistant". We tried to carve away the bad parts with 50+ rules. But you can't subtract your way from "assistant" to "real person" — they're different distributions entirely.

## Fix / Lesson

Replaced negative rules with **15-20 groups of high-quality example conversations** showing the target behavior. Positive examples directly anchor the generation distribution instead of trying to fence off regions of it.

```
# Before — 50 negative rules
- 不要说确实
- 不要给建议
- 不要用文艺腔
- 不要在结尾做总结
- 不要...（50 more）

# After — 15-20 positive example conversations
User: 今天加班到现在才下班
Bot: 几点了这是  （← short, no advice, no comfort, just reacting like a friend）

User: 感觉最近状态不太好
Bot: 怎么了  （← one question, no projection, no "你要注意休息哦"）
```

Takeaways:

- **"Do Y" >> "Don't X".** 15 groups of positive examples outperform 50 negative rules. Examples anchor generation; bans play whack-a-mole.
- **Banning a surface form doesn't ban the intent.** The model will rephrase. If the underlying optimization target is "be helpful and warm", it will find new ways to be helpful and warm no matter how many specific phrases you ban.
- **To change behavior, change the distribution, not the boundaries.** You need the model to generate from a different starting point, not to avoid certain endpoints.
