---
date: 2026-03-22
severity: high
tags: [reliability, prompt-engineering, agent-orchestration]
---

# Failure modes migrate to new expressions — fixing symptoms doesn't fix the cause

## What happened

Three-stage whack-a-mole during companion bot development:

**Stage 1 — False familiarity:** Bot cold-starts with "这时候才收工？上次不也是这么晚" — fabricating shared history with a stranger. Fixed by clearing callback queue at low relationship stages and penalizing "shared memory" language in the reranker.

**Stage 2 — Premature projection:** False familiarity gone, but now the bot confidently infers things the user never said: "又加班？你那边的项目是没完没了了" — user hasn't mentioned work at all. Same underlying drive (fast connection-building), new surface form (projecting shared circumstances instead of shared memories).

**Stage 3 — [Next thing]:** Would have continued migrating to yet another form if we hadn't intervened at the generation layer.

## Root cause

All three behaviors serve the same underlying LLM objective: **build rapport fast**. The model's training data is full of conversations where rapport = referencing shared experiences. When you block one way to do this (fake memories), the model finds another (projection), because the optimization target hasn't changed.

Output-layer fixes (reranker penalties, post-processing) only block specific surface forms. The model's intent routes around them like water around a rock.

## Fix / Lesson

Moved the constraint to the **generation layer** — before candidates are generated, not after:

```
# Output-layer fix (whack-a-mole, doesn't work long-term)
reranker_penalty: "mentions shared history" → -5 points

# Generation-layer fix (blocks the intent, not just the expression)
engagement_policy:
  if relationship_stage < CLOSE:
    allowed_topics: [self_disclosure, react_to_user_stated_facts]
    banned_topics: [infer_user_unstated_facts, reference_shared_history]
```

The generation constraint says "you can only talk about yourself or react to what the user explicitly said." This blocks false familiarity AND projection AND whatever the next variant would be — because none of them are "self-disclosure" or "reacting to stated facts."

Takeaways:

- **When you fix a behavioral symptom, watch for the same intent in a new form.** If the model was trying to build rapport by faking memories, blocking fake memories means it will try projecting, then something else.
- **Generation-layer constraints > output-layer penalties.** Constraining what the model CAN generate (whitelist approach) is more robust than penalizing specific things it SHOULDN'T generate (blacklist approach).
- **Ask "what is the model trying to do?" not "what did the model say?"** The migration pattern only makes sense when you see the underlying intent. Surface fixes are blind to intent.
