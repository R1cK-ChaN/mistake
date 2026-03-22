---
date: 2026-03-22
severity: high
tags: [reliability, eval-and-testing]
---

# Post-processing regex fixes give false sense of progress — model intent unchanged

## What happened

Built a `normalize_companion_reply()` function that used regex and replacement dicts to strip bad patterns from bot output: removing "确实", cleaning up 文艺腔 phrases, trimming summary endings. The metrics improved — "确实" frequency dropped to zero, literary phrases decreased.

But the model's underlying behavior was completely unchanged. It was still generating "确实" — we were just deleting it after the fact. Worse: the metric improvements made the team believe the problem was getting solved, delaying the real structural work (changing the prompt and generation approach) by weeks.

## Root cause

Post-processing operates on **surface tokens**, not on **generation intent**. The model's optimization target was still "be warm, be supportive, wrap things up nicely." Deleting the output tokens doesn't change what the model is trying to do — it just hides the evidence.

The false progress was the real damage. When your dashboard shows "确实 usage: 0%", nobody feels urgency to fix the underlying prompt. The post-processing became a coping mechanism that prevented the team from confronting the real problem.

## Fix / Lesson

Reduced post-processing to **format-only operations** (stripping tool call leaks, fixing punctuation). All behavior shaping moved to the generation layer (prompt, examples, candidate selection).

```
# What post-processing should do
- Strip leaked tool call syntax
- Fix broken punctuation/encoding
- Remove validation starters ("好的，")

# What post-processing should NOT do
- Delete banned words/phrases (move to prompt examples)
- Rewrite sentence endings (move to reranker)
- Trim "advice-like" content (move to generation constraints)
```

Takeaways:

- **Post-processing that removes behavioral patterns creates false metrics.** The behavior frequency drops to zero in output, but the model is still generating it. You're measuring the filter, not the model.
- **If you're regex-fixing a behavior, you haven't fixed it.** Regex is a band-aid. The fix belongs in the generation layer — prompt, examples, or candidate selection.
- **Post-processing should be format-only, never behavior-shaping.** The moment post-processing starts making content decisions, it becomes a coping mechanism that delays real fixes.
