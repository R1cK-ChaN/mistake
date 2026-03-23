---
date: 2026-03-23
severity: high
tags: [agent-orchestration, prompt-engineering, vibe-coding]
---

# Skipped research phase, went straight to coding agent — built wrong architecture

## What happened

Had an idea for a product feature. Jumped straight into Claude Code / Codex and started building. The coding agent was fast and confident — scaffolded an architecture, set up files, wired components. It felt productive.

Weeks later, discovered the chosen approach had known limitations that a 10-minute search would have revealed. The architecture needed to be ripped out and rebuilt. The time "saved" by skipping research was paid back 5x in rework.

## Root cause

Treated the coding agent as a thinking tool when it's an execution tool. Coding agents (Claude Code, Codex, etc.) operate on the code in front of them — they don't survey the landscape, compare approaches, or research what others have tried. They'll build whatever you ask, confidently, even if the approach is fundamentally flawed.

Web-grounded chatbots (ChatGPT, Claude.ai, Gemini) have internet access and deep research capabilities that coding agents don't. They can survey existing solutions, identify pitfalls, and synthesize analysis. Different tool, different job.

## Fix / Lesson

Established a mandatory research phase before any coding:

1. **Brainstorm with web-grounded chatbots first.** Use ChatGPT / Claude.ai / Gemini for ideation — they can ground ideas against real-world information.
2. **Produce a PRD before opening the coding agent.** The PRD covers: what, why, technical approach, implementation sequence, external dependencies, success criteria.
3. **The PRD lives in the repo.** It becomes the reference document the coding agent points back to when context is lost.

Takeaways:

- **Coding agents are execution tools, not thinking tools.** They'll build the wrong thing just as fast as the right thing. The thinking happens before the coding session starts.
- **Use the right AI for the right phase.** Web chatbots for research and design, coding agents for implementation. Mixing them up wastes the strengths of both.
- **10 minutes of research prevents weeks of rework.** The cost of skipping research is invisible until you're deep enough in that rebuilding is painful.
