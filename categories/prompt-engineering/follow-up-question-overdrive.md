---
date: 2026-03-22
severity: medium
tags: [prompt-engineering, agent-orchestration]
---

# Four systems compound to force questions every turn — bot becomes interrogator

## What happened

Bot ended 4 consecutive replies with a question. Not the simple boolean-switch problem (that was fixed earlier with budgets/cooldowns). This time, multiple systems independently pushed toward questions and overrode each other's cooldowns:

1. **Soul prompt** encouraged peer-style follow-up questions
2. **Disengagement detector** misread short replies to questions as disengagement → set `topic_invite` flag → forced another question
3. **`topic_invite` hint** overrode the style hint cooldown (more specific hint beats general hint)
4. **Style hint cooldown** correctly said "don't ask again" but was outranked by the topic_invite

The cascade: bot asks question → user gives short answer (normal for answering a question) → detector thinks user is disengaged → injects topic_invite → bot asks another question → loop.

## Root cause

A false positive feedback loop between the disengagement detector and the question system. The detector didn't distinguish between:
- Short reply because user is disengaged ("嗯", "ok" unprompted)
- Short reply because user is answering a direct question ("星巴克" in response to "你去哪家？")

A short answer to a question is not disengagement — it's a normal conversational response. The detector treated all short replies equally, creating a self-reinforcing cycle.

Additionally, hint priority wasn't defined: when `topic_invite` (specific) and `question_cooldown` (general) conflicted, the specific hint won by default. But the cooldown should have had higher priority — it's a safety constraint, not a default.

## Fix / Lesson

Four fixes for four failure points:

```python
# 1. Disengagement detector: skip short replies that answer a question
def is_disengaged(user_msg, bot_last_msg):
    if bot_last_msg.ends_with_question:
        # Short reply to a question is normal, not disengagement
        return False
    return len(user_msg) < SHORT_THRESHOLD and consecutive_short >= 2

# 2. topic_invite: respect question cooldown
def get_topic_invite(bot_history):
    if bot_history[-1].ends_with_question:
        # Bot just asked — use statement version, not question version
        return topic_invite_statement  # "说起来我最近..." not "你最近...？"
    return topic_invite_question

# 3. Style hint priority: cooldown > invite
hint_priority = {
    "question_cooldown": 10,   # safety constraint, high priority
    "topic_invite": 5,         # suggestion, lower priority
    "soul_prompt_encourage": 3 # default, lowest priority
}

# 4. Soul prompt: add consecutive constraint
"你可以追问，但不要连续两条都以问句结尾。"
```

Takeaways:

- **Short reply to a question ≠ disengagement.** Any disengagement detector must know whether the bot just asked a question. "星巴克" after "你去哪家？" is a perfect answer, not a red flag.
- **When multiple hint systems exist, define priority explicitly.** Safety constraints (cooldowns, limits) must outrank suggestions (topic invites, encouragements). Never let a suggestion override a constraint.
- **Test for feedback loops between systems.** Ask: "does system A's output trigger system B, which triggers system A again?" If yes, you have a loop. The question → short answer → disengagement → question cycle is a textbook example.
- **Compound failures look different from single failures.** The boolean-switch fix (budgets/cooldowns) worked for the simple case. This is a 4-system interaction that no single fix would catch — each system needed adjustment.
