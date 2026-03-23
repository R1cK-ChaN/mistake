---
date: 2026-03-23
severity: high
tags: [reliability, agent-orchestration, eval-and-testing]
---

# Introduced LLM components before deterministic logic was solid — couldn't isolate bugs

## What happened

Built a system with both rule-based logic and LLM-powered components. Integrated the LLM parts early because they were the "interesting" part of the product. When the system produced wrong outputs, couldn't determine the source:

- Was the business logic wrong?
- Was the LLM hallucinating?
- Was the prompt missing context?
- Was the data pipeline feeding bad inputs?

Spent days debugging what turned out to be a simple off-by-one error in the deterministic code — but it was masked by the LLM layer, which sometimes compensated for the bug (making it intermittent) and sometimes amplified it (making it look like a model issue).

## Root cause

LLM components are non-deterministic by nature. Every LLM call introduces variability — same input can produce different outputs. When you mix non-deterministic components with untested deterministic code, you lose the ability to debug by controlling variables.

Deterministic bugs become intermittent because the LLM sometimes masks them. LLM issues look like logic bugs because the deterministic code feeds them bad inputs. The two failure modes become entangled, and the only way to untangle them is to rip them apart — which is what should have been done from the start.

## Fix / Lesson

Strict ordering: deterministic first, LLM last.

```
# Correct build order:
1. Build ALL rule-based, deterministic components
2. Test thoroughly — system works perfectly without any LLM
3. Introduce LLM components one at a time, with clear boundaries
4. Each LLM component gets its own eval and fallback path

# Debugging benefit:
- If deterministic layer is tested and solid:
  - Bug appears → must be in the LLM layer or the boundary
  - Narrows the search space immediately
- If both layers are untested:
  - Bug appears → could be anywhere
  - Debugging is combinatorial explosion
```

Takeaways:

- **LLM = uncontrollable. Don't mix instability into the project early.** Every LLM call is a source of non-determinism. Add it only to a stable foundation, never to shaky ground.
- **You debug deterministic code by controlling variables. You can't control an LLM the same way.** If the deterministic layer isn't solid, you've lost your primary debugging tool.
- **Intermittent bugs are the worst bugs.** An LLM masking a deterministic error makes the bug appear and disappear based on model randomness. You'll chase the wrong cause for days.
- **The LLM is not the product — it's a component of the product.** Build the product first, then add the LLM component. Not the other way around.
