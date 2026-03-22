---
date: 2026-03-22
severity: high
tags: [reliability, tool-use, post-processing]
---

# Post-processing pipeline destroys URLs in three different ways

Three independent bugs in the output post-processing pipeline, all silently breaking Google Maps URLs. Each one alone would corrupt the link. Together they made it impossible for any URL to survive from tool output to user-facing message.

---

## Bug 1: `_trim_overwritten_reply` treats `?` in URLs as sentence-ending punctuation

### What happened

```
Input:  https://maps.google.com/?cid=12345678
Output: https://maps.google.com/?
```

The `_trim_overwritten_reply` function uses a regex to split text at sentence boundaries:

```python
re.split(r"(?<=[？!?])")
```

This matches `?` as a sentence-ending punctuation mark. A Google Maps URL contains `?` as a query string delimiter — the regex splits at that `?`, and everything after it (`cid=12345678`) gets treated as a "second sentence" and trimmed.

### Root cause

The regex has no concept of URLs. It sees `?` and assumes end-of-sentence. This is a classic regex-on-natural-language failure — but worse, because it's applied to structured data (URLs) that happens to be embedded in text.

### Fix

Detect URLs in the text and skip the entire trim logic when present:

```python
def _trim_overwritten_reply(text: str) -> str:
    if re.search(r'https?://', text):
        return text  # URLs contain ? and other punctuation — don't split
    return _original_trim_logic(text)
```

---

## Bug 2: `_truncate_bubble` hard-cuts at 60 characters, doesn't spare URLs

### What happened

```
Input:  "推荐海底捞 https://maps.google.com/?cid=12345678"
Output: "推荐海底捞 https://maps.google.com/?cid=1234567"
                                                    ^ cut at 60 chars
```

The bubble length limiter counts all characters equally. A 50-character URL inside a 70-character message gets truncated at character 60 — producing a broken link that looks valid but goes nowhere.

### Root cause

The truncation logic treats URLs as regular text. It doesn't know that a half-URL is worse than no URL — a truncated link is actively misleading because it looks clickable but 404s or lands on the wrong page.

### Fix

Messages containing URLs skip truncation entirely:

```python
def _truncate_bubble(text: str, limit: int = 60) -> str:
    if re.search(r'https?://', text):
        return text  # never truncate URLs mid-string
    if len(text) <= limit:
        return text
    return text[:limit] + "..."
```

---

## Bug 3: Google Maps `g_mp=` tracking parameter inflates URL to 131 chars, model drops it

### What happened

```
# What Google Places API returns (131 characters)
https://maps.google.com/?cid=12345678&g_mp=AATQhUnK7d2xB3kF...base64...

# What the model actually outputs
https://maps.google.com/?
```

Google Maps URIs include a `g_mp=` parameter containing base64-encoded tracking data. This inflates the URL from ~50 to ~131 characters. The model — already dealing with long structured output — silently truncates or drops the URL entirely. It's too long to reliably reproduce character-by-character.

### Root cause

The tool passed through the raw Google Maps URI without cleaning it. The `g_mp=` parameter is a Google tracking/analytics param — it's not needed to open the map. The `cid=` parameter alone is sufficient to identify the place.

Even without the other two bugs, this one would still cause failures: 131-character URLs with base64 noise are at the edge of what LLMs reliably reproduce in their output.

### Fix

Strip `g_mp=` at the tool level before the model ever sees it:

```python
from urllib.parse import urlparse, parse_qs, urlencode, urlunparse

def clean_maps_url(url: str) -> str:
    """Strip tracking params from Google Maps URL. cid= is sufficient."""
    parsed = urlparse(url)
    params = parse_qs(parsed.query)
    params.pop("g_mp", None)  # tracking param, not needed
    cleaned_query = urlencode(params, doseq=True)
    return urlunparse(parsed._replace(query=cleaned_query))

# Before: https://maps.google.com/?cid=12345678&g_mp=AATQhUnK7d2x... (131 chars)
# After:  https://maps.google.com/?cid=12345678 (~50 chars)
```

---

## Cross-cutting lessons

These three bugs form a pattern: **every layer in the pipeline independently assumed URLs wouldn't be in the text, and each one broke them in a different way.**

| Layer | Assumption | Failure |
|---|---|---|
| Tool output | Raw URL is fine to pass through | `g_mp=` makes it too long for model to reproduce |
| Model output | Model will reproduce URL exactly | 131-char URL with base64 gets truncated |
| Post-processing (trim) | `?` means end-of-sentence | Query string `?` triggers split |
| Post-processing (truncate) | 60-char limit is safe | URL cut mid-string, broken link |

Takeaways:

- **URLs are structured data, not text. Every text-processing function in your pipeline must be URL-aware.** Grep your codebase for any function that splits, truncates, or transforms text and ask: "what happens if there's a URL in here?"
- **Clean API data at the tool boundary, not downstream.** Strip tracking params, normalize URLs, and shorten where possible BEFORE the model sees them. The model is a lossy channel — the shorter and cleaner the input, the more reliably it reproduces it.
- **Post-processing is the most dangerous layer for silent corruption.** It runs after the model, so there's no reasoning step to catch the damage. Test post-processing functions with URLs, code snippets, and other structured content — not just plain text.
- **LLMs cannot reliably reproduce long URLs character-by-character.** Keep URLs under ~80 characters. Strip tracking params, use short links, or pass URLs as separate structured fields that bypass text processing entirely.
