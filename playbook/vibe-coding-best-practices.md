# Best Practices for the Vibe Coding Era

Hard-won rules for building stable products when AI writes every line of code. These aren't theoretical — each one comes from watching the opposite approach fail.

---

## Phase 1: Research & Design — Use the right AI for the right job

**Brainstorm with web-grounded chatbots, not coding agents.**

Before touching any code, use website chatbots (ChatGPT, Claude.ai, Gemini) for ideation and research. They have what coding agents don't: internet grounding and deep research capabilities. They can survey existing solutions, compare approaches, identify pitfalls, and synthesize a comprehensive analysis that no coding agent can match.

This phase must produce a **Product Requirements Document (PRD)**:
- What the product does and why
- Technical approach and method choices (with rationale from research)
- Step-by-step implementation sequence
- External dependencies and API interfaces
- Success criteria

The PRD becomes the reference document that lives in the repo. Every coding session points back to it. When the coding agent loses context (and it will), the PRD is the source of truth that gets it back on track.

**Why this matters:** Coding agents are execution tools, not thinking tools. If you start coding before you've shaped the idea, you'll vibe-code yourself into an architecture that's hard to change — and you won't realize it's wrong until you're deep in.

---

## Phase 2: Build bottom-up, one unit at a time

**Never ask a coding agent to implement the full architecture in one shot.**

Build from the foundation up, in this order:

1. **External interfaces first** — API contracts, third-party integrations, anything you don't control. Pin these down because they constrain everything above.
2. **Data layer** — Database schemas, data models. Get the foundation right.
3. **Core logic** — Business rules, deterministic processing. One component at a time.
4. **Integration** — Wire components together.
5. **Interface layer** — UI, CLI, API endpoints. Last, not first.

At each level:
- **Unit test and eval before moving up.** Don't stack untested layers — bugs compound and become impossible to trace.
- **Update docs after every feature/component.** This is doc-oriented programming. When the coding agent's context explodes or gets truncated, the docs are what keep the next session coherent.
- **Commit working state frequently.** Every completed unit is a checkpoint you can return to.

**Why this matters:** Coding agents are confident and fast. They'll happily scaffold an entire architecture that looks impressive and fails in subtle ways. Building unit by unit forces verification at each step. Slower to start, dramatically faster to finish.

---

## Phase 3: Classic software practices still apply

**Don't let vibe coding trick you into skipping fundamentals.**

- **Dev / Staging / Prod environments.** The same environment separation that worked before AI still works. Dev for experimentation, staging for integration validation, prod for users. Make sure you can roll back at any time.
- **Version control discipline.** Frequent commits, meaningful messages, branches for features. The fact that AI wrote the code doesn't change the need to track what changed and why.
- **Rollback capability at every stage.** Never deploy anything you can't undo. This is more important with vibe coding, not less — because you have less line-by-line understanding of what changed.

**Why this matters:** Vibe coding changes who writes the code, not the physics of software. Deployments still fail, integrations still break, users still find edge cases. The safety nets exist because the problems exist.

---

## Phase 4: Deterministic first, LLM last

**Get all rule-based logic complete and sound before introducing any LLM component.**

LLM = uncontrollable. Every LLM call introduces non-determinism, latency, cost, and a failure mode you can't fully predict. If you mix LLM components into the early build:
- You can't tell if a bug is in your logic or in the model's output
- You can't reproduce failures reliably
- Debugging requires reasoning about probabilistic behavior on top of deterministic bugs
- Complexity compounds in ways that are impossible to untangle

The correct order:
1. Build all rule-based, deterministic components first
2. Test them thoroughly — they should work perfectly without any LLM
3. *Then* introduce LLM components, one at a time, with clear boundaries
4. Each LLM component gets its own eval and fallback path

**Why this matters:** You can debug deterministic code by controlling variables. You cannot debug an LLM by controlling variables — you can only constrain its inputs and validate its outputs. If the deterministic foundation isn't solid, you'll never know whether failures come from your code or from the model.

---

## Phase 5: Validate first, optimize later

**Use the best model at MVP stage. Downgrade after validation.**

During development and MVP:
- Use the most capable model available (e.g., Opus, GPT-4o). Don't worry about cost.
- The goal is to **validate that the business logic works end-to-end** with the highest-quality LLM output.
- If it doesn't work with the best model, it won't work with a cheaper one — and you need to know that now, not after you've optimized for cost.

After validation:
- Swap in smaller/cheaper models and measure quality degradation
- Apply prompt engineering, caching, or other cost optimizations
- Find the minimum viable model for each task

**Why this matters:** Premature cost optimization wastes more than it saves. You'll spend days tuning prompts for a cheap model, only to discover the feature itself is wrong. Validate the product first, optimize the bill second.

---

## Phase 6: Log everything from day one

**Every layer should log. No exceptions.**

Implement comprehensive logging from the start, not after the first incident:
- **External API calls** — request, response, latency, errors
- **Data layer** — queries, mutations, what changed
- **Business logic** — decisions made, conditions evaluated, branches taken
- **LLM calls** — prompt (or hash), response, tokens used, model version
- **Inter-component boundaries** — what was passed, what was received

**Why this matters:** As the project grows, you will lose the ability to hold the full architecture in your head (or in the agent's context). The only way to debug a complex system is by controlling variables — and you can only control what you can observe. Without logs, a bug in a large architecture is a needle in a haystack where you can't even see the hay.

This is especially critical in vibe-coded projects where you may not have deep familiarity with every line of code. Logs are how you understand what the code you didn't manually write is actually doing.

---

## Summary: The vibe coding workflow

```
1. RESEARCH    Web chatbots (grounded, deep research) → PRD
2. BUILD       Bottom-up, unit by unit, test each layer
3. PRACTICE    Dev/staging/prod, version control, rollback ready
4. ORDER       Deterministic logic first → LLM components last
5. VALIDATE    Best model first → downgrade after it works
6. OBSERVE     Logging at every layer from day one
```

The common thread: **never be greedy for getting everything in the short term.** Vibe coding makes it feel like you can build fast without discipline. You can build fast *with* discipline — and that's the only version that produces something stable.
