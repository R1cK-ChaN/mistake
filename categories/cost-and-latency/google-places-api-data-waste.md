---
date: 2026-03-22
severity: medium
tags: [cost-and-latency, tool-use]
---

# Paying for Advanced tier API but only using 5 fields

## What happened

Integrated Google Places API (New) so the agent could recommend restaurants/shops with real details — open hours, ratings, location. But the vibe-coded implementation only requested 5 basic fields in the field mask, missing critical ones like price range, review count, website, and Google Maps link.

Was paying Advanced tier pricing but getting Basic tier value. The agent gave shallow recommendations ("4.5 stars, open now") instead of rich ones ("4.5 stars from 2,300 reviews, ¥80-120/person, closes at 22:00, here's the map link").

## Root cause

Vibe coding (letting LLM generate the integration) produced a **minimally functional** implementation. It got the API call working — correct endpoint, valid response — but never explored what fields were actually available. The generated code used a narrow field mask because that's enough to "work".

The engineer (me) didn't review the API response schema or cross-check the field mask against the billing tier. Trusted "it works" without asking "is it complete?"

**Core lesson: vibe coding optimizes for "runs without error", not for "extracts full value".**

## Fix / Lesson

Expanded field mask to include the fields we were already paying for:

```
# Before — 5 fields, wasting Advanced tier cost
field_mask = "places.displayName,places.formattedAddress,places.rating,places.currentOpeningHours,places.types"

# After — full value from Advanced tier
field_mask = "places.displayName,places.formattedAddress,places.rating,places.currentOpeningHours,places.types,places.priceRange,places.priceLevel,places.userRatingCount,places.websiteUri,places.googleMapsUri"
```

Also changed the tool output from dumping raw JSON dict to structured text for the agent:

```
# Before — raw JSON, agent has to parse and often ignores fields
return json.dumps(place_data)

# After — structured text, agent uses everything
return f"""
{place['displayName']}
地址: {place['formattedAddress']}
评分: {place['rating']} ({place['userRatingCount']}条评价)
人均: {place.get('priceRange', 'N/A')}
营业时间: {place['currentOpeningHours']}
网站: {place.get('websiteUri', 'N/A')}
地图: {place['googleMapsUri']}
"""
```

Takeaways:

- **After vibe coding any API integration, open the API docs and diff the available fields against what was implemented.** The LLM will get you to "working", not to "complete".
- **Match your field mask to your billing tier** — if you're paying for Advanced, use Advanced fields. Audit this.
- **Structure tool output for the agent** — raw JSON wastes tokens and agents skip fields. Formatted text means every field gets used in the response.
- **Vibe coding needs a "completeness review" step** — add it to your workflow. "Does this use everything the API offers that's relevant to the use case?"
