---
date: 2026-03-22
severity: high
tags: [agent-orchestration, reliability]
---

# Per-turn stateless evaluation can't prevent repetitive patterns — bot monologues for 5 turns

## What happened

The bot talked about playing Detroit: Become Human for **5 consecutive turns**. User responses degraded from "好的" → "嗯" → "ok", clear disengagement signals. The bot didn't notice. User finally asked "你还记得我是你谁吗" (do you even remember who I am).

Each individual reply scored fine in the reranker — on-topic, appropriate length, had personality. But the reranker evaluated each turn independently. It had no concept of "you've been talking about the same thing for 5 turns and the user stopped engaging."

## Root cause

Three compounding failures:

1. **Reranker was stateless per-turn.** Each candidate was scored in isolation. A reply about Detroit scored well because it was on-topic — the reranker didn't know the bot had already given 4 Detroit replies.

2. **Engagement modulator was blind to user signals.** It tracked topic relevance and time-of-day, but not user response length or engagement level. "嗯" and a 200-word enthusiastic reply were treated identically.

3. **"Don't ask follow-up questions" rule was absolute.** It killed topic invitations that would naturally shift the conversation. The bot had no mechanism to redirect.

## Fix / Lesson

Three fixes, each addressing one failure:

```python
# 1. User disengagement detection
if consecutive_short_replies(user_messages, n=2):
    # User is checked out — force a topic shift
    inject_topic_invite = True

# 2. Reciprocity enforcement
if consecutive_self_focused_replies(bot_messages, n=2):
    # Bot has been monologuing — must redirect to user
    next_reply_must_reference_user = True

# 3. History-aware reranking
def rerank(candidates, recent_bot_messages):
    for c in candidates:
        # Penalize if candidate topic == last 2 bot message topics
        if topic_overlap(c, recent_bot_messages[-2:]) > threshold:
            c.score -= REPETITION_PENALTY
        # Penalize if candidate + recent history = all statements, no questions
        if all_statements(recent_bot_messages[-2:]) and is_statement(c):
            c.score -= MONOTONY_PENALTY
```

Takeaways:

- **Any per-turn scoring system must also see recent history.** A candidate that looks good in isolation can be terrible in sequence. The reranker needs a sliding window of 2-3 recent bot messages minimum.
- **User response length is a strong disengagement signal.** Going from full sentences to "嗯" to "ok" is the clearest possible signal that the conversation needs to change direction. Monitor it.
- **Social reciprocity must be enforced mechanically.** LLMs have no built-in drive to balance self-disclosure with other-inquiry. Without explicit reciprocity rules, the model will happily monologue forever — every turn scores well individually.
