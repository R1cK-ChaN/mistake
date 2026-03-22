---
date: 2026-03-22
severity: high
tags: [prompt-engineering, agent-orchestration]
---

# Boolean behavior switch flips from one extreme to the other — no middle ground

## What happened

The bot had a rule: "don't ask follow-up questions" (to avoid sounding like a therapist). This killed ALL questions including normal peer-like curiosity ("你说的是哪家？"). When we flipped it to "encourage peer-style follow-up questions", the bot immediately started ending every single reply with a question — 4 consecutive turns of interrogation.

```
# Phase 1: "don't ask questions"
User: 今天去了个新的咖啡馆
Bot: 哦  （← dead end, no engagement）

# Phase 2: "encourage questions"
User: 今天去了个新的咖啡馆
Bot: 哪家？好喝吗？
User: 还行吧
Bot: 什么豆子？手冲还是意式？
User: ...手冲
Bot: 浅烘还是深烘？你一般喝什么？  （← interrogation）
```

## Root cause

LLMs treat behavioral instructions as optimization targets, not soft guidelines. "Don't ask questions" = ask zero questions. "Encourage questions" = maximize questions. There's no built-in concept of "sometimes, when natural." Every boolean instruction gets pushed to its extreme.

## Fix / Lesson

Replaced boolean switches with **budgets and cooldowns**:

```
# Before — boolean
"Don't ask questions" / "Encourage questions"

# After — budgeted with cooldown
question_budget: 0-1 per turn
question_cooldown: after asking a question, next 2 turns must be statements
question_type_allowed: topic-invitation ("你说的哪家？")
question_type_banned: emotional-probing ("你是不是不开心？")
```

Also added **cross-turn pattern detection** in the reranker: if the bot asked a question in the last 2 turns, penalize candidates that end with questions.

Takeaways:

- **Never use boolean on/off for behavioral control.** LLMs will maximize whatever you enable and zero-out whatever you disable. Use budgets (0-1 per turn), cooldowns (not twice in a row), and type distinctions (this kind yes, that kind no).
- **Distinguish question types.** Topic invitations ("哪家？") and emotional probes ("你是不是不开心？") are completely different behaviors. Banning all questions to avoid probes also kills natural curiosity.
- **Cross-turn awareness is mandatory.** A per-turn rule can't prevent "question every turn" because each turn is evaluated independently. The reranker must see recent history.
