# Thread — Why Centralized Orchestrators Beat Distributed Handlers in Agent Systems

**Platform:** LinkedIn / X
**Date:** Day 2, Week 12

---

**Post 1/5**

My partner asked: "Why is a centralized state machine safer than letting each tool handler decide locally?"

I thought the answer was obvious — "single source of truth." But when I tried to name the *specific invariants* that break without centralization, I realized most of us can't. Here's what I found 🧵

---

**Post 2/5**

A multi-tool agent has three invariants only a central orchestrator can enforce:

1️⃣ **Monotonic transitions** — state only moves forward. No handler can independently revert or skip steps.

2️⃣ **Idempotent processing** — duplicate events are harmless. The orchestrator rejects events that don't match current state.

3️⃣ **Global eligibility** — "is this tool allowed right now?" depends on FULL lifecycle state, not the local context any single handler sees.

---

**Post 3/5**

The concrete failure without centralization:

```
t=0  Email handler sends email
t=1  Duplicate reply webhook arrives (delivered twice)
t=2  SMS handler sees "reply" → sends SMS
t=3  Calendar handler sees same "reply" → sends booking link
t=4  Lead gets BOTH — contradictory actions from one event
```

With a centralized orchestrator: duplicate is rejected at t=2. Only one action fires. State stays consistent.

---

**Post 4/5**

The agent-systems insight: a centralized orchestrator does for tool INVOCATION what constrained decoding does for token GENERATION.

• Constrained decoding masks invalid tokens → guarantees valid structure
• Orchestrator masks invalid tools → guarantees valid actions

Both work by RESTRICTING the action space rather than hoping the actor makes the right choice.

---

**Post 5/5**

What can still break even with an orchestrator:
• Crash mid-transition (fix: outbox pattern)
• Provider confirms but doesn't deliver (fix: delivery webhooks)
• Stale reads under concurrency (fix: optimistic locking)

The orchestrator isn't a silver bullet — but it moves failures from "silently corrupt" to "explicitly detectable."

Paper: Garcia-Molina & Salem, "Sagas" (1987) — why centralized coordinators beat choreography under partial failures.

---
