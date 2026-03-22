---
date: 2026-03-22
severity: medium
tags: [cost-and-latency, tool-use]
---

# Paying for Advanced tier API but only using 5 fields

## What happened

Integrated Google Places API (New) so the agent could recommend restaurants/shops with real details. But the vibe-coded implementation only requested 5 fields in the field mask:

| Requested (5 fields) | Missing (never requested) |
|---|---|
| `displayName` | `priceRange` |
| `formattedAddress` | `priceLevel` |
| `rating` | `userRatingCount` |
| `currentOpeningHours` | `websiteUri` |
| `editorialSummary` | `googleMapsUri` |

Was paying **Advanced tier** pricing ($25–30 per 1K requests for fields like `priceRange`) but only using **Basic/Essentials tier** fields. The agent gave thin recommendations with no way for users to act on them:

```
# What the agent said (before)
"I recommend Hai Di Lao at 313 Somerset. It's rated 4.5 and currently open."

# What it could have said (after)
"I recommend Hai Di Lao at 313 Somerset. Rated 4.5 from 3,200 reviews,
 ~$30-50/person, open until 06:00. Website: haidilao.com
 📍 Google Maps: https://maps.google.com/?cid=..."
```

No price info → user can't judge if it fits budget. No review count → 4.5 from 3 reviews vs 3,200 is meaningless. No map link → user has to manually search again. The recommendation was technically correct but practically useless.

## Root cause

The AI generated a field mask that was enough to get a valid API response. It picked the most obvious fields (`displayName`, `rating`, `formattedAddress`) and stopped there. The code compiled, the API returned data, the agent produced an answer — all green signals.

Two failures on my side:

1. **Didn't review the field mask against the API pricing page.** Google Places (New) bills per-field-tier. If you're hitting Advanced tier pricing (because of `currentOpeningHours`), you should be requesting ALL Advanced fields — they're included in the cost. Leaving them out is literally paying for data you don't use.

2. **Didn't test the agent output against what a real user needs.** A recommendation without price, review count, or a map link is a dead end. The user has to leave the agent and search manually — which defeats the purpose of having the tool at all.

**Core lesson: AI-generated code optimizes for "runs without error", not for "extracts full value". It will never request fields it wasn't told about, even if they're free at your billing tier.**

## Fix / Lesson

### 1. Expanded field mask to match billing tier

```python
# Before — 5 fields, paying Advanced tier but using Basic tier value
field_mask = (
    "places.displayName,"
    "places.formattedAddress,"
    "places.rating,"
    "places.currentOpeningHours,"
    "places.editorialSummary"
)

# After — full Advanced tier value
field_mask = (
    "places.displayName,"
    "places.formattedAddress,"
    "places.rating,"
    "places.currentOpeningHours,"
    "places.editorialSummary,"
    # --- these were missing, all included in Advanced tier cost ---
    "places.priceRange,"
    "places.priceLevel,"
    "places.userRatingCount,"
    "places.websiteUri,"
    "places.googleMapsUri"
)
```

### 2. Structured tool output instead of raw JSON

```python
# Before — raw JSON dict, agent ignores most fields
return json.dumps(place_data)

# After — structured text, every field used in response
def format_place(place: dict) -> str:
    price = place.get('priceRange') or place.get('priceLevel', 'N/A')
    return (
        f"{place['displayName']}\n"
        f"地址: {place['formattedAddress']}\n"
        f"评分: {place['rating']} ({place.get('userRatingCount', '?')}条评价)\n"
        f"人均: {price}\n"
        f"营业时间: {format_hours(place.get('currentOpeningHours'))}\n"
        f"网站: {place.get('websiteUri', 'N/A')}\n"
        f"地图: {place.get('googleMapsUri', 'N/A')}"
    )
```

### Takeaways

- **After any API integration, open the pricing page and diff available fields against your field mask.** If you're paying for a tier, use every field in that tier. This is a 5-minute audit that prevents ongoing waste.
- **Test agent output from the user's perspective, not the developer's.** "Does this response let the user take action without leaving the chat?" If not, you're missing fields.
- **Structure tool output as formatted text, not raw JSON.** Raw JSON wastes context tokens and agents routinely skip fields. Pre-formatted text ensures every field shows up in the response.
- **AI-generated API integrations need a "completeness review".** The LLM stops at "it works". Your job is to check "does it use everything available and relevant?"
