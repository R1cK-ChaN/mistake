---
date: 2026-03-22
severity: medium
tags: [prompt-engineering, agent-orchestration]
---

# Calibrate cold first, then add warmth — not the other way around

## What happened

After 6 phases of making the bot behave like a real person — killing assistant behaviors, adding opinion, enforcing reciprocity — the bot was realistic but **cold**. User feedback: "太冷了，根本没有一点聊天欲望，还我快乐小狗".

We had to add warmth back. But the warmth we needed was different from the warmth we removed:

| Removed (service warmth) | Wanted (puppy warmth) |
|---|---|
| "辛苦了" "会好起来的" "注意休息" | 主动分享, 容易兴奋, 话多好奇 |
| Reactive: you're sad → I comfort | Proactive: I'm high-energy → you feel it |
| Therapist/assistant pattern | Friend-who-texts-you-first pattern |

## Root cause

Not exactly a mistake — more of a deliberate strategy that paid off. Starting cold and adding warmth was the right sequence, but it's counterintuitive enough to document.

If we had started with a warm persona, we would never have been able to distinguish "genuine friend warmth" from "LLM default assistant warmth" — they look identical on the surface. By going cold first, we built the behavioral control infrastructure (engagement modulator, reranker penalties, stage constraints) that could keep warmth from sliding back into assistant mode.

## Fix / Lesson

Added warmth back through **controlled channels**, not by loosening constraints:

1. Internal state events upgraded from neutral to emotionally colored
2. Soul prompt anchor: "话多、好奇心强、容易兴奋的人" (not "warm and caring")
3. Engagement modulator default energy: low → medium-high
4. Added proactive sharing triggers: 1-2 unsolicited messages per day
5. Reranker: added "energy" dimension, reduced weight on "shortest = best"

Takeaways:

- **Cold → warm is safer than warm → cold.** LLM's default failure mode is fake warmth. Starting cold forces you to build control infrastructure first. Adding warmth back through controlled channels means you can distinguish genuine personality warmth from assistant warmth.
- **"Warm" is not one thing.** Service warmth (reactive comfort) and personality warmth (proactive energy) are completely different behaviors. Specify which one you mean.
- **Going cold first also sets a behavioral baseline.** Once you know the bot can be appropriately restrained, you can selectively loosen constraints with confidence that the infrastructure will catch regressions.
