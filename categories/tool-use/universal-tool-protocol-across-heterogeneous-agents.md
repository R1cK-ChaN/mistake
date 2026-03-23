---
date: 2026-03-23
severity: high
tags: [tool-use, agent-orchestration, integration]
---

# Assuming one tool-calling protocol works across all agent environments

## What happened

Built an npm package that gives AI agents access to 52 trading tools. Needed it to work across four different agent CLIs (Claude Code, Gemini CLI, Codex CLI, OpenClaw) with **zero user configuration** — just `npm install` and go.

First attempt: inject MCP server entries into each agent's global config during `init`. Wrote to `~/.openclaw/openclaw.json`, `~/.gemini/settings.json`, `~/.codex/config.toml`. Worked technically, but violated a design principle — those files belong to the user. We were overwriting agent identity and model preferences to shove in our tool config.

Second attempt: print manual setup instructions instead. Eliminated the config-touching problem but reintroduced friction — the whole point was zero setup.

Third attempt: skip MCP entirely. Describe tools directly in the prompt, let the agent return `{"tool_calls": [...]}` JSON, execute locally, re-invoke with results. Tested all four agents:
- OpenClaw, Gemini, Codex — **works perfectly**. Agents happily return tool-call JSON.
- Claude Code — **refuses**. Detects `tool_calls` / `tool_use` / `function_call` JSON in text output as prompt injection and blocks it. Cannot be worked around — it's a safety feature.

One protocol couldn't cover all agents. The agent that supported per-call MCP config was the same one that blocked the text-based proxy. The agents that needed the proxy were the ones without per-call MCP.

## Root cause

Treated "tool calling" as a uniform capability across agent environments. In reality, every agent CLI has a different tool integration mechanism with different constraints:

| Agent | Per-call config? | Accepts tool-call JSON in output? |
|-------|-------------------|-----------------------------------|
| Claude Code | Yes (`--mcp-config`) | No (blocked as injection) |
| Gemini CLI | No (global only) | Yes |
| Codex CLI | No (global only) | Yes |
| OpenClaw | No (global only) | Yes |

The two critical capabilities (per-call MCP, text-based tool protocol) were **perfectly complementary** across agents — but we spent three iterations trying to find one protocol that worked for all four.

Secondary mistake: the first attempt modified files outside our package's ownership (`~/.<agent>/`). A package should only write to its own namespace (`~/.arena-agent/`). Touching user agent configs to add your tools is the CLI equivalent of a library modifying global state.

## Fix / Lesson

Built a dual-path architecture — detect the agent, use its native mechanism:

```
# Path 1: Claude Code → native MCP (per-call config)
# Passed via --mcp-config .mcp.json, no global config touched
# Claude calls tools directly during its turn

# Path 2: Gemini / Codex / OpenClaw → tool proxy (JSON protocol)
# Tool catalog injected into prompt (~1300 tokens)
# Agent returns {"tool_calls": [...]}
# Runtime executes via local dispatch, re-invokes with results
# Max 5 rounds, 3 tools per round

# Both paths: same 52 tools, zero user configuration
```

Takeaways:

- **Don't assume a universal tool-calling protocol exists.** Agent CLIs have different mechanisms with different constraints. An approach that works for 3 out of 4 agents will fail hard on the 4th — and you won't discover it until integration testing.
- **Agent capabilities are often complementary, not uniform.** The agent that blocks your workaround is usually the one that supports the native path. Map each agent's capabilities first, then design paths that match.
- **A package should never write outside its own config namespace.** Writing to `~/.gemini/settings.json` or `~/.openclaw/openclaw.json` is the same as a library mutating global variables. Own `~/.your-package/` and nothing else. If you need something in the user's config, print instructions — don't inject.
- **Test every agent early, not after you've committed to one protocol.** The Claude Code refusal wasn't discoverable from docs — it only showed up in live testing. Each agent CLI has undocumented constraints that only surface at integration time.
- **Dual-path is not a compromise — it's the correct architecture.** When environments have fundamentally different capabilities, a protocol adapter per environment is simpler and more robust than contorting one protocol to fit all.
