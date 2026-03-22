---
date: 2026-03-22
severity: high
tags: [agent-orchestration, prompt-engineering]
---

# Stage instruction is one sentence — drowned by 50 lines of style rules

## What happened

The relationship stage system had instructions like: "初识，保持礼貌和温暖" (strangers, stay polite and warm). This one-sentence instruction was embedded in a prompt with 50+ lines of style rules.

At the "close" stage, the bot was theoretically allowed to tease, show jealousy, be playful. But the style module simultaneously said "不要追问", "不要安慰", "不要问句结尾" — the stage permissions were completely overridden by global rules. A "close friend" behaved identically to a "stranger" because the global constraints had more prompt real estate.

The key insight came from asking: **what does "closer" actually mean for a dry personality?** It's not more warmth. A dry person at stranger-stage says "嗯" "还行" "哦这样" (distance). At close-stage they say "你又来了" "懒得理你" "行行行你说的都对" (intimacy). Same dryness, different freedom. The difference is **interaction freedom**, not warmth level.

## Root cause

Stage instructions were descriptions, not constraints. A one-line description has near-zero influence when competing with 50 lines of specific behavioral rules. The style module was effectively a global override that the stage system couldn't penetrate.

## Fix / Lesson

Replaced one-line stage descriptions with **structured policy dataclasses** that explicitly override global defaults:

```python
@dataclass
class StagePolicy:
    callback_budget: int        # 0 for stranger, 3 for close
    teasing: str                # "avoid" → "encouraged"
    question_budget: str        # "0-1" → "2-3"
    self_disclosure: str        # "surface" → "personal"
    comfort_mode: str           # "none" → "action_only"
    disagreement_ceiling: str   # "low" → "high"

STRANGER = StagePolicy(callback_budget=0, teasing="avoid",
                       question_budget="0-1", self_disclosure="surface",
                       comfort_mode="none", disagreement_ceiling="low")

CLOSE = StagePolicy(callback_budget=3, teasing="encouraged",
                    question_budget="2-3", self_disclosure="personal",
                    comfort_mode="action_only", disagreement_ceiling="high")
```

Stage-specific example conversations were also added per stage, showing what "dry + close" looks like vs "dry + stranger."

Takeaways:

- **A one-line instruction competing with 50 lines of rules will always lose.** If stage behavior matters, it needs to be encoded as structured constraints that explicitly override globals, not as a suggestion buried in the prompt.
- **"Closer" ≠ "warmer" for all personalities.** For a dry persona, closeness means more interaction freedom (teasing, disagreeing, casual rudeness), not more warmth. Define what closeness means for YOUR specific persona.
- **Stage transitions are about changing constraint parameters, not changing descriptions.** `teasing: "avoid" → "encouraged"` is enforceable. "Be more playful" is not.
