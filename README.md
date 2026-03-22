# Mistake Handbook

A personal engineering handbook for the vibe coding era. Every mistake here came from AI-generated code — I don't write a single line by hand. The role is agent engineer: designing, reviewing, and directing, not typing. The mistakes are about what happens when you trust the output without sufficient review.

## How to add an entry

1. Copy `templates/mistake.md` into the relevant category folder
2. Name it descriptively: `forgot-to-validate-tool-output.md`
3. Fill in the template
4. Commit

## Categories

| Folder | Covers |
|---|---|
| `prompt-engineering` | Prompt design, system prompts, few-shot, instruction tuning |
| `tool-use` | Tool calling, MCP, function schemas, output parsing |
| `agent-orchestration` | Multi-agent, routing, delegation, loops, planning |
| `context-management` | Context window, memory, RAG, retrieval, summarization |
| `eval-and-testing` | Evals, benchmarks, regression, golden sets |
| `reliability` | Retries, fallbacks, error handling, guardrails, hallucination |
| `cost-and-latency` | Token usage, caching, batching, model selection |
| `deployment` | Infra, API keys, rate limits, versioning, monitoring |

## Conventions

- One mistake per file
- Use frontmatter tags for cross-category references
- Severity: **high** = caused outage/data loss, **medium** = wasted significant time, **low** = minor gotcha
- Be honest about root cause — that's where the value is
