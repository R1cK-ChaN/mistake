---
date: 2026-03-23
severity: high
tags: [eval-and-testing, agent-orchestration, vibe-coding]
---

# Asked coding agent to implement full architecture in one shot — untested layers compounded bugs

## What happened

Gave the coding agent a broad instruction: "build the API with auth, database, business logic, and endpoints." It delivered — a full scaffold that looked impressive. Files everywhere, clean structure, types wired up.

Started testing from the top (API endpoints) and hit failures. Couldn't tell if the bug was in the endpoint handler, the business logic, the database queries, or the auth middleware. Each layer had small issues that interacted in unexpected ways. Debugging one layer required understanding all layers simultaneously. A bug that would take 5 minutes to find in isolation took hours to trace through the full stack.

## Root cause

Stacked untested layers. Each layer was "probably fine" individually, but small errors in each layer compounded into system-level failures that were impossible to decompose. The coding agent builds confidently — it doesn't stop to say "we should test this before building on top of it."

This is the vibe coding trap: the agent's speed makes it feel like building everything at once is efficient. It's not — it's borrowing verification time from the future at a punishing interest rate.

## Fix / Lesson

Build bottom-up, one unit at a time. Test before moving up:

```
1. External interfaces — API contracts, third-party integrations (pin these first)
2. Data layer        — schemas, models (test: can I CRUD correctly?)
3. Core logic        — business rules (test: do rules produce correct outputs?)
4. Integration       — wire components (test: do layers talk correctly?)
5. Interface         — UI, CLI, endpoints (test: does the user flow work?)
```

Takeaways:

- **Never ask a coding agent to implement full architecture in one shot.** Break it into units. Test each unit before building the next.
- **Bugs compound across untested layers.** A small bug per layer means exponential debugging time at the top. Testing each layer in isolation keeps bugs linear.
- **The coding agent won't stop to verify — that's your job.** It will happily build 10 layers without testing any. You need to impose the discipline of test-then-proceed.
- **Bottom-up is slower to start, dramatically faster to finish.** The time spent testing each layer is repaid many times over by not debugging the entire stack at once.
