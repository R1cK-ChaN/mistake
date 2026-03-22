---
date: 2026-03-22
severity: medium
tags: [eval-and-testing, agent-orchestration]
---

# Heuristic reranker hits ceiling — can't detect subtle behavioral issues

## What happened

The multi-candidate reranker used heuristic scoring: response length, blacklist words, question-at-end detection, topic overlap penalties. It worked well for obvious issues — too long, banned phrases, interrogation patterns.

But it couldn't catch **subtle behavioral problems** like "this reply is technically fine but reads like the bot is managing the user's emotions." The heuristic had no way to score "隐性管理对方情绪" (covert emotional management) because it's not a word or a pattern — it's a pragmatic interpretation that requires understanding intent.

## Root cause

Heuristic rerankers operate on surface features: token presence, length, syntax patterns. They're fast and deterministic but fundamentally limited to what can be expressed as regex/rule. The gap between "surface features" and "behavioral quality" is where the ceiling hits.

Examples of things heuristics can't catch:
- "Technically on-topic but subtly steering the user's emotions"
- "Correct information but delivered in a condescending tone"
- "Not explicitly advice but structured as advice"

## Fix / Lesson

Kept heuristic reranker for fast surface checks but designed the architecture with a **migration path to LLM-as-judge**:

```
# Current: heuristic reranker (fast, cheap, surface-level)
candidates → heuristic_score() → best candidate

# Future path: LLM judge for subtle quality
candidates → heuristic_score()    → top 2 candidates
           → llm_judge(top_2)     → final selection

# LLM judge prompt
"Which reply sounds more like a real person texting and less like
an AI managing the user's emotions? Reply A or B. Just the letter."
```

Takeaways:

- **Heuristic rerankers have a ceiling at "surface features".** They catch banned words, length violations, and pattern repetition. They can't catch tone, intent, or pragmatic meaning.
- **Design for the migration path from day one.** Even if you start with heuristics (you should — they're fast and cheap), structure your pipeline so swapping in an LLM judge is a config change, not a rewrite.
- **Use heuristics as a filter, LLM as a judge.** Heuristics narrow from N candidates to top 2-3 (cheap). LLM judge picks the winner from 2-3 (expensive but only runs on finalists). This keeps cost manageable.
