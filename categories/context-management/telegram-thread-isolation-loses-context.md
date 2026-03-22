---
date: 2026-03-22
severity: high
tags: [context-management, deployment]
---

# Telegram reply creates new thread — bot loses the conversation it's replying to

## What happened

In a group chat:

```
[thread=main]
User: 新加坡哪里有胖哥兩
Bot:  PangPang, 在 xxx 路...

[user replies to bot's message → Telegram creates thread=1534]
User: gmap 链接

Bot:  *searches "Japanese restaurant near National Library Singapore"*
      — completely unrelated topic
```

The user replied to the bot's PangPang answer asking for the Google Maps link. A simple, obvious follow-up. The bot searched for something entirely unrelated — it had no idea what PangPang was or that it had just recommended it.

Same root cause also made the bot give near-identical replies to different users in the group — each thread had the same empty context, so the model generated the same default response.

## Root cause

Telegram's reply mechanism creates a new `message_thread_id`. When a user replies to a message, the reply lives in a new thread (e.g., `thread=1534`), not in the original thread (`thread=main`).

The code used `thread_id` to isolate conversation history:

```python
# Simplified
history = get_history(chat_id=chat_id, thread_id=message.message_thread_id)
```

When the user replied to the bot's PangPang message, the new thread's history contained only:
- "只有一家吗" (a previous reply in that thread)
- "gmap 链接" (the current message)

The original question ("新加坡哪里有胖哥兩") and the bot's answer ("PangPang at xxx") were in `thread=main` — invisible to `thread=1534`. The model saw "gmap 链接" with zero context and hallucinated a topic from prior conversations.

```
thread=main history:     [user: 新加坡哪里有胖哥兩, bot: PangPang...]
thread=1534 history:     [user: gmap 链接]  ← no context, model guesses
```

## Fix / Lesson

When a new thread is created (first message in a non-main thread), inject the **original replied-to message** as a seed:

```python
def get_thread_history(chat_id: int, thread_id: int, message) -> list:
    history = get_history(chat_id=chat_id, thread_id=thread_id)

    if not history and message.reply_to_message:
        # New thread — seed with the message being replied to
        original = message.reply_to_message
        role = "assistant" if original.from_user.id == BOT_ID else "user"
        history = [{
            "role": role,
            "content": original.text
        }]

    return history
```

Key details:
- Check `reply_to_message.from_user.id` to set the correct `role` — if the user replied to the bot's message, the seed is `role: assistant`; if they replied to another user's message, the seed is `role: user`
- Only seed on thread creation (empty history), not every message — once the thread has its own history, it's self-contained

Takeaways:

- **Platform threading models ≠ conversational threading models.** Telegram creates threads on reply; your bot's conversation history uses thread_id for isolation. These two models interact in non-obvious ways. Before using any platform's thread/reply mechanism for history isolation, map out exactly when new threads are created and what context carries over (answer: nothing, by default).
- **Test with reply chains, not just top-level messages.** "User sends message → bot replies" works. "User replies to bot's reply" breaks. The second pattern is how real group chats actually work — it should be in your test suite.
- **Empty context + LLM = hallucination.** When the model gets a message like "gmap 链接" with no history, it will guess. It doesn't say "I don't have context" — it confabulates from prior conversations or training data. The fix is never letting the model see a contextless follow-up.
- **This bug is invisible in 1:1 chats.** Thread isolation only matters in group chats where Telegram creates threads on reply. If you only tested in DMs, you'd never see it. Always test group chat flows separately.
