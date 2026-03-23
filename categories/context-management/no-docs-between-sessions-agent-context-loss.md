---
date: 2026-03-23
severity: medium
tags: [context-management, vibe-coding, agent-orchestration]
---

# No docs updated between sessions — coding agent lost context, produced contradictory code

## What happened

Built a feature across multiple coding sessions. Didn't update any documentation between sessions. Each new session started with the coding agent re-reading the codebase to reconstruct context — but it missed design decisions, understood components differently, and made changes that contradicted earlier work.

In one case, session 2 refactored a data structure that session 1 had deliberately designed for a specific access pattern. The refactor looked "cleaner" but broke the performance assumption. Without docs explaining *why* the original structure was chosen, the agent had no way to know it was intentional.

## Root cause

Coding agents have no persistent memory across sessions. When context is lost (new session, context window truncation, conversation reset), the agent reconstructs understanding from code alone. But code shows *what*, not *why*. Design decisions, tradeoffs, and deliberate constraints are invisible in the code itself.

Without docs, every new session is a fresh agent that's smart enough to read code but has zero awareness of the intent behind it. It will "improve" things that were deliberately designed that way.

## Fix / Lesson

Adopted doc-oriented programming — update docs after every feature or component:

```
# After completing each unit of work:
1. Update the relevant doc (architecture.md, API spec, data model doc)
2. Record the WHY, not just the WHAT — design decisions, tradeoffs, constraints
3. Commit docs alongside code — they're part of the deliverable
4. Next session: point the agent at the docs first before any code changes
```

Takeaways:

- **Docs are not for humans — they're for the next coding session.** In vibe coding, documentation is the persistence layer for agent context. Treat it as critical infrastructure, not an afterthought.
- **Code shows what, docs show why.** The agent can read your code perfectly. It cannot infer why you chose this approach over alternatives. That's what docs preserve.
- **Update docs after every feature, not at the end.** If you wait, you'll forget the reasoning. And the next session — which needs it most — won't have it.
- **Context loss is not a bug, it's the default.** Every coding agent session starts from zero. Plan for it by leaving a trail the next session can follow.
