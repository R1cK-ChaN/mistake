---
date: 2026-03-23
severity: medium
tags: [cost-and-latency, eval-and-testing, vibe-coding]
---

# Optimized for cost at MVP stage instead of validating with best model first

## What happened

Started building with a cheaper, smaller model to keep costs low during development. Spent days tuning prompts to get acceptable quality from the budget model. Finally got it "working" — outputs looked reasonable, basic test cases passed.

Later switched to a better model for comparison and discovered the feature itself was flawed — the business logic had gaps that the better model would have exposed immediately, but the weaker model had been producing plausible-looking garbage that passed superficial checks. The prompt engineering effort was wasted because it was optimizing the wrong thing.

## Root cause

Premature cost optimization. At MVP stage, the goal is to validate the product logic, not minimize the bill. A cheap model that produces mediocre output creates two problems:

1. **You can't distinguish "model limitation" from "product bug."** When output is wrong, is it because the model isn't capable enough, or because your logic/prompt/data is broken? With the best model, if the output is wrong, it's your fault — clear signal.

2. **You waste engineering time on the wrong problem.** Prompt tuning for a weak model is effort spent making a bad tool less bad, instead of effort spent validating whether the product works at all.

## Fix / Lesson

Use the best available model during development and MVP. Downgrade after validation:

```
# MVP stage:
- Use the most capable model (Opus, GPT-4o, etc.)
- Don't worry about cost — you're validating, not scaling
- If it doesn't work with the best model, the product is broken
- If it works with the best model, you have a validated baseline

# Post-validation:
- Swap in cheaper models, measure quality degradation
- Apply prompt engineering, caching, batching
- Find the minimum viable model for each task
- Now cost optimization is meaningful — you know what "correct" looks like
```

Takeaways:

- **Best model first, downgrade after validation.** You need the best model to establish what "correct" looks like. Without that baseline, you're optimizing blind.
- **If it doesn't work with the best model, it won't work with a cheaper one.** The best model is your upper bound. Fix the product first, then find the cheapest model that clears the bar.
- **Premature cost optimization wastes more than it saves.** Days of prompt tuning for a cheap model are worthless if the feature itself is wrong.
- **At MVP stage, validate fast — cost doesn't matter yet.** You're not serving millions of users. The bill for using Opus during development is trivial compared to the cost of building the wrong product.
