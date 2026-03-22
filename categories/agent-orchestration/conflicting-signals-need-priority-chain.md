---
date: 2026-03-22
severity: high
tags: [agent-orchestration, prompt-engineering]
---

# Multiple behavior control signals conflict with no priority order — model picks randomly

## What happened

Three systems simultaneously influenced the bot's reply style:
- **Relationship stage**: "you're close friends, teasing is allowed"
- **Engagement modulator**: "user seems disengaged, lower energy"
- **User emotion detection**: "user seems upset, be gentle"

When all three fired at once (close friend + disengaged + upset), the model got contradictory signals: tease (stage says OK) vs. be gentle (emotion says required) vs. lower energy (engagement says pull back). The result was unpredictable — sometimes it teased an upset user, sometimes it went silent on a close friend who just needed a joke.

## Root cause

Each control module was designed independently. None knew about the others. There was no defined priority chain, so when signals conflicted, the model resolved the contradiction arbitrarily — whichever instruction happened to have more weight in the prompt that turn.

This is the multi-agent coordination problem applied to a single agent with multiple behavioral modules: without explicit priority, modules compete and the output is non-deterministic.

## Fix / Lesson

Defined an explicit priority chain:

```
Priority (highest to lowest):
1. User emotion     — if user is upset, override everything, be appropriate
2. Engagement policy — if user is disengaged, adjust energy/topic
3. Relationship stage — default behavioral permissions

# In practice:
if user_emotion in [SAD, ANGRY, STRESSED]:
    style = emotion_appropriate_style(user_emotion)
elif user_disengaged:
    style = low_energy_topic_shift()
else:
    style = stage_default_style(relationship_stage)
```

Takeaways:

- **When multiple modules influence generation, define an explicit priority chain.** Without one, the model resolves conflicts randomly. Document: "emotion > engagement > stage" or whatever your hierarchy is.
- **Design modules with awareness of each other.** Each module should know what signals can override it. "Stage allows teasing" needs a caveat: "unless emotion module says user is upset."
- **Test with scenarios where all modules fire simultaneously.** The happy path (one module active) always works. The failure is in the intersection. Create test cases: "close friend + disengaged + upset" and verify the output matches your priority chain.
