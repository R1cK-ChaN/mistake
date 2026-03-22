---
date: 2026-03-22
severity: medium
tags: [tool-use, agent-orchestration]
---

# Tool returns formatted text string instead of structured dict — model can't extract fields

## What happened

The `search_places` tool returned a pre-formatted multi-line text string:

```python
def _format_place(place: dict) -> str:
    return (
        f"{place['displayName']}\n"
        f"地址: {place['formattedAddress']}\n"
        f"评分: {place['rating']} ({place['userRatingCount']}条评价)\n"
        f"人均: {place.get('priceRange', 'N/A')}\n"
        f"营业时间: {format_hours(place['currentOpeningHours'])}\n"
        f"网站: {place.get('websiteUri', 'N/A')}\n"
        f"地图: {place.get('googleMapsUri', 'N/A')}"
    )
```

After JSON serialization this became a single escaped string with `\n` literals. URLs, prices, and ratings were all buried in one blob of text. The model had to regex-parse its own tool output to extract a Google Maps link or a website URL.

Result: the model sometimes dropped URLs from its response, mangled them by truncating mid-string, or mixed up fields from adjacent lines. Structured data went in, got flattened to text, then the model had to reverse-engineer the structure — a round-trip that loses information at every step.

## Root cause

Earlier I fixed the raw-JSON-dump problem (model ignoring fields) by switching to formatted text. That fixed one problem but created another: **formatted text is easy to read but hard to extract from.**

The mistake was treating tool output as a display format instead of a data format. The tool's job is to return structured data; the model's job is to format it for the user. When the tool pre-formats, it takes away the model's ability to selectively use fields.

```
Structured dict → model picks what to show → good response
Formatted text  → model parses text to get fields back → lossy, fragile
```

## Fix / Lesson

Return a structured dict. Let the model decide how to present it:

```python
# Before — formatted text string, URLs buried in blob
def _format_place(place: dict) -> str:
    return f"{place['displayName']}\n地址: {place['formattedAddress']}\n..."

# After — structured dict, every field independently accessible
def _format_place(place: dict) -> dict:
    return {
        "name": place["displayName"],
        "address": place["formattedAddress"],
        "rating": place.get("rating"),
        "review_count": place.get("userRatingCount"),
        "price_range": place.get("priceRange"),
        "price_level": place.get("priceLevel"),
        "opening_hours": format_hours(place.get("currentOpeningHours")),
        "website": place.get("websiteUri"),
        "maps_url": place.get("googleMapsUri"),
        "summary": place.get("editorialSummary"),
    }
```

The model can now directly access `maps_url` as a standalone field instead of pattern-matching it out of a text blob.

Takeaways:

- **Tool output should be structured data (dict/list), not pre-formatted text.** The model is better at formatting for users than you are at formatting for models.
- **URLs must be standalone fields, never embedded in text.** An LLM extracting a URL from a multi-line string will truncate it, hallucinate parts, or skip it entirely.
- **Don't pre-format what the model will re-format anyway.** If you return a text blob, the model will try to rephrase it (it's a language model, that's what it does). It can't rephrase a text blob without risking data loss. But it can compose a perfect response from clean structured fields.
- **This contradicts the earlier "use structured text instead of raw JSON" fix** — and that's the real lesson. The first fix was wrong in the other direction. The correct answer was always: **structured dict with clear keys, serialized with `ensure_ascii=False`**.
