---
date: 2026-03-22
severity: high
tags: [eval-and-testing, reliability, tool-use]
---

# Tool audit only logs tool name, not input/output — can't diagnose failures

## What happened

Had tool audit logging in place, but it only recorded **which tool the agent called**. Never logged the tool's return value. When things went wrong, couldn't tell where the fault was:

1. **Agent called the wrong tool?** (agent routing problem)
2. **Tool returned bad/incomplete data?** (tool implementation problem)
3. **Agent misinterpreted correct tool output?** (agent reasoning problem)

All three look identical when you only have `tool_name: "google_places"` in your logs. Spent hours guessing which layer broke, every single time.

## Root cause

Treated tool audit as a usage tracker ("which tools are popular") instead of a debugging tool ("what exactly happened in this interaction"). The vibe-coded logging implementation did the minimum — log the call — and I didn't spec out what "audit" actually needs to cover.

Also conflated live/staging requirements. In production, minimal logging makes sense (cost, privacy, PII). But the same sparse logging was running in staging/testing where there's zero reason not to capture everything.

## Fix / Lesson

Different audit levels by environment:

```
# Production — lightweight, privacy-safe
audit_log:
  tool_name: ✓
  tool_input_hash: ✓        # for dedup/replay, not debugging
  latency_ms: ✓
  success/fail: ✓
  tool_output: ✗             # PII/cost concern

# Staging / Testing / Internal — full trace
audit_log:
  tool_name: ✓
  tool_input: ✓              # full input params
  tool_output: ✓             # full return value
  latency_ms: ✓
  success/fail: ✓
  agent_reasoning: ✓         # why the agent chose this tool (if available)
  output_used_in_response: ✓ # what the agent actually did with the result
```

With full logs, every failure becomes classifiable:

| Symptom | tool_input | tool_output | agent_response | Diagnosis |
|---|---|---|---|---|
| Wrong answer | correct | correct | wrong | Agent reasoning error |
| Wrong answer | correct | incomplete | wrong | Tool implementation gap |
| Wrong answer | wrong params | N/A | wrong | Agent tool-calling error |
| No tool called | N/A | N/A | hallucinated | Tool routing failure |

Takeaways:

- **Audit logging in staging must cover the full request lifecycle**: tool selection reason → tool input → tool output → how agent used the output
- **You can't debug agent systems with application-level logging** — you need agent-level observability (the reasoning chain, not just the function call)
- **Separate your logging config by environment** — production can be lean, staging/testing must be verbose. No reason to be blind where you don't have to be
- **Define your audit schema before implementation** — "add logging" is too vague for vibe coding. Spec exactly which fields to capture or the LLM will do the minimum
