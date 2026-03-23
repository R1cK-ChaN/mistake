---
date: 2026-03-23
severity: high
tags: [agent-orchestration, cost-and-latency, over-engineering]
---

# Two-layer agent architecture where one agent + rules suffices

## What happened

Built a two-layer agent system: a **setup agent** that configures trading strategy, and a **runtime agent** that monitors market prices every 30 seconds and makes decisions. Sounded elegant — separation of concerns, the setup agent handles the complex thinking, the runtime agent handles live execution.

Three problems killed it:

1. **Token costs exploded.** The runtime agent ran every 30 seconds, each invocation consuming tokens for context, reasoning, and tool calls. Even on a subscription plan with weekly limits, burned through the entire allowance in days. A polling agent is a token furnace — the cost scales with frequency, not with the number of useful decisions made.

2. **The two layers created a meaningless complexity tradeoff.** If the runtime agent had real autonomy (freedom to deviate from setup), the setup agent's configuration was pointless — why carefully configure a strategy if the runtime agent ignores it? If the runtime agent had no autonomy (strictly follows setup), it was just an expensive rule engine. There was no sweet spot: either the setup layer is meaningless or the runtime layer should be deterministic code.

3. **Silent failures across layers were impossible to debug.** Vibe-coded the system to a "works on demo" state, but inter-layer communication had subtle failures — missed handoffs, context loss, misinterpreted strategy parameters. Each layer worked in isolation; the failures only appeared in the interaction between them. With no comprehensive logging, diagnosing which layer caused a bad trade required reconstructing the full conversation chain manually.

## Root cause

Jumped to multi-agent architecture before doing the context engineering work that would make *any* LLM layer effective. The two-layer design wasn't just the wrong shape — it was premature complexity built on a weak foundation.

LLMs are genuinely capable of decision-making — but like human decision-makers, they need **reliable information sources and bounded scope** to decide well. Without curated inputs, an LLM doesn't fail by refusing to decide — it fails by confidently deciding on garbage, and you can't tell the difference from the outside. The runtime agent was making trading decisions on raw, unbounded market context with no information curation. Adding a second agent layer before solving the context problem just doubled the surface area for bad decisions.

The compounding mistakes:

1. **Premature architecture before context engineering.** Never got the information pipeline right (what data, how much, how reliable) before wiring up the agents. The runtime agent received noisy, unbounded context and made noisy, unreliable decisions. No amount of architectural elegance fixes bad inputs.

2. **The runtime layer's job was fundamentally deterministic.** Monitor price, check conditions, execute action — this requires `if price < threshold: sell`, not reasoning. Using an LLM for this adds cost and non-determinism to tasks that need neither.

3. **"More agent = more capable" is wrong.** A two-layer agent architecture is justified when both layers genuinely need reasoning over novel situations. When one layer's job is mechanical, that layer should be code, not an agent.

4. **Vibe coding masked the architectural problems.** The system reached a "running" state quickly, but without structured logging, the silent failures in inter-layer communication were invisible until they caused real damage.

## Fix / Lesson

Collapsed to a single setup agent that configures a **rule-based strategy**, executed by deterministic code:

```
# Before — two agents
setup_agent → configures strategy → runtime_agent (LLM, every 30s)
  - Runtime agent reads market, reasons about strategy, decides action
  - Tokens consumed every 30s whether or not anything interesting happens
  - Inter-agent communication fails silently

# After — one agent + rules
setup_agent → configures rules → rule_engine (code, every 30s)
  - Rule engine: if conditions met, execute action. Zero tokens.
  - Setup agent only invoked for initial config or user-requested changes
  - Added comprehensive logging at every decision point
```

Takeaways:

- **Don't use an agent for tasks that don't require reasoning.** Polling a price feed, checking thresholds, executing predefined actions — these are `if/else` problems. LLMs add cost and non-determinism to tasks that need neither.
- **High-frequency agent invocation is economically unsustainable.** An agent called every 30 seconds consumes tokens proportional to time, not to useful work. Most invocations will conclude "nothing to do" — paying full LLM cost for a null decision.
- **Two-layer agent has an autonomy paradox.** If the inner agent has real freedom, the outer agent's configuration is theater. If it doesn't, replace it with code. The architecture only makes sense when both layers genuinely need to reason about novel situations.
- **Vibe coding hides architectural mistakes.** Getting a multi-agent system to "run" is easy. Getting it to run *correctly* requires structured logging across every layer boundary. Add comprehensive logging before you add complexity — you'll need it to discover that the complexity was unnecessary.
- **Do context engineering before architecture engineering.** An LLM making decisions needs reliable info sources and bounded scope — same as a human. If you haven't solved what information the agent sees and how much, adding more agent layers just multiplies the garbage-in-garbage-out problem.
- **An LLM without curated inputs becomes the bottleneck, not the boost.** It won't refuse to decide — it'll decide confidently on bad data. That's worse than no agent at all, because it looks like it's working.
- **Start with the simplest architecture that could work, then add agent layers only when deterministic code demonstrably fails.** One agent + rules is easier to build, cheaper to run, and far easier to debug than two agents talking to each other.
