---
date: 2026-03-22
severity: high
tags: [tool-use, reliability, i18n]
---

# ensure_ascii=True turns Chinese into unicode escapes the model can't use

## What happened

The agent loop serialized tool results with `json.dumps(result)`. Python's default is `ensure_ascii=True`, which escapes all non-ASCII characters. The model received this:

```json
{"name": "\u5e97\u540d", "address": "\u5730\u5740: \u4e09\u5df7\u8def123\u53f7"}
```

Instead of this:

```json
{"name": "店名", "address": "地址: 三巷路123号"}
```

The model could technically decode unicode escapes, but in practice it degraded response quality — the agent sometimes echoed raw `\uXXXX` sequences to users, mistranslated characters, or simply ignored fields it couldn't easily read. For a product serving Chinese-speaking users, this is a high-severity bug hiding in a default parameter nobody thought to change.

## Root cause

`json.dumps()` in Python defaults to `ensure_ascii=True`. This is a safe default for ASCII-only systems but actively harmful when your data contains CJK characters and your consumer is an LLM that works better with readable text.

Nobody changed it because nobody checked. The vibe-coded implementation used `json.dumps(result)` — the most basic, copy-paste invocation — and since it didn't throw an error, it shipped. The output was "valid JSON" but practically unreadable for the use case.

**This is a class of bug where Python's defaults are wrong for LLM applications.** The code is technically correct — it just produces garbage for the consumer.

## Fix / Lesson

One-character fix with massive impact:

```python
# Before — unicode escapes destroy CJK readability
json.dumps(result)
# {"name": "\u5e97\u540d", "address": "\u5730\u5740"}

# After — human-readable, LLM-readable
json.dumps(result, ensure_ascii=False)
# {"name": "店名", "address": "地址"}
```

Takeaways:

- **Any `json.dumps()` in an agent's tool output path must use `ensure_ascii=False`** if you serve non-ASCII languages. Grep your codebase for `json.dumps` and audit every call site.
- **Python defaults are not LLM defaults.** `ensure_ascii=True` is safe for wire protocols but hostile to LLM readability. Always think about who consumes the output.
- **Test with real non-ASCII data, not just English.** If this had been tested with a single Chinese query, the escaped output would have been immediately obvious.
