---
date: 2026-03-22
severity: high
tags: [tool-use, reliability]
---

# API returns low-quality duplicate as first result — agent uses it blindly

## What happened

User asked about Clarke Quay MRT station. Google Places returned 3 results:

| # | Result | Reviews | Quality |
|---|---|---|---|
| 1 | Some random listing | 16 reviews | Garbage — wrong map link |
| 2 | Another duplicate | ~50 reviews | Mediocre |
| 3 | The real Clarke Quay MRT | 175 reviews | Correct — right location, right link |

The agent took result #1 (first in the list) and confidently gave the user a Google Maps link pointing to the wrong place. The correct result was sitting right there at position #3, but the tool returned results in API default order, and the model had no reason to skip to #3.

## Root cause

Two failures compounding:

1. **Google Places API default ordering is not quality ordering.** The API returns results by relevance-to-query, not by trustworthiness. A low-review duplicate can rank above the canonical listing because it keyword-matches slightly better.

2. **No post-processing between API response and model input.** The tool passed raw API results straight to the model in API-default order. The model — reasonably — trusts the first result. It has no heuristic for "16 reviews = probably garbage, 175 reviews = probably real."

The agent engineer's job is to bridge this gap. The API doesn't know what "quality" means for your use case. The model doesn't know API ordering isn't quality ordering. **Someone has to sort/filter, and that someone is the tool implementation.**

## Fix / Lesson

Sort results by `userRatingCount` descending before passing to the model:

```python
def search_places(query: str) -> list[dict]:
    results = google_places_api.search(query)

    # API default order ≠ quality order
    # Higher review count = more established, more trustworthy listing
    results.sort(key=lambda r: r.get("userRatingCount", 0), reverse=True)

    return [format_place(r) for r in results]
```

For more complex cases, a composite score:

```python
def quality_score(place: dict) -> float:
    reviews = place.get("userRatingCount", 0)
    rating = place.get("rating", 0)
    # Review count matters more than rating for deduplication
    # A 4.0 with 500 reviews beats a 5.0 with 3 reviews
    return reviews * 0.7 + rating * 0.3
```

Takeaways:

- **Never pass API results to the model in API-default order.** The API's ranking criteria are not your ranking criteria. Sort by whatever metric represents quality for your use case — usually review count for places, relevance score for search, recency for news.
- **The model trusts the first result.** If your tool returns a list, the first item carries disproportionate weight. Make sure the first item is the best item.
- **Review count is a better quality signal than rating for deduplication.** A 4.8-star place with 16 reviews is likely a spam/duplicate listing. A 4.3-star place with 175 reviews is a real, established business. `userRatingCount` is the single best filter for junk results.
- **Tool implementations must include a data quality layer.** Raw API → model is a pipeline with no quality gate. You need at minimum: sort by quality, deduplicate, and filter out low-signal results before the model sees them.
