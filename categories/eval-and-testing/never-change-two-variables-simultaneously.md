---
date: 2026-03-22
severity: medium
tags: [eval-and-testing]
---

# Changed model and architecture simultaneously — can't attribute results

## What happened

Were about to swap the underlying LLM AND deploy a new behavioral architecture at the same time. If the output improved, no way to know if the new model was better, the new architecture was better, or both. If it degraded, no way to know which change caused it.

We caught this before deploying and ran the new architecture on the old model first. But the instinct to batch changes is strong — "we're already deploying, might as well swap the model too."

## Root cause

Time pressure and the desire to minimize deployment cycles. Changing two things at once feels more efficient. It's not — it's more efficient to deploy but less efficient to debug, and debugging is where the real time goes.

## Fix / Lesson

Hard rule: **one variable per experiment.**

```
# Wrong — can't attribute results
Deploy v2 architecture + switch to Claude Sonnet 4 simultaneously

# Right — isolatable
Week 1: Deploy v2 architecture on Claude Haiku (old model)
         → Measure: did architecture improve behavior?
Week 2: Switch to Claude Sonnet 4 on v2 architecture
         → Measure: did model improve behavior?
```

Takeaways:

- **One variable per experiment, no exceptions.** This is basic scientific method but extremely easy to violate under shipping pressure.
- **"We're already deploying" is not a reason to batch changes.** Each additional variable multiplies debugging complexity, it doesn't add linearly.
- **If you must change two things, at least have a rollback plan for each independently.** Feature flags that let you toggle architecture vs. model separately.
