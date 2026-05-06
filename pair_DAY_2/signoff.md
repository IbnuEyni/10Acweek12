# Sign-Off — Day 2

## Amir's sign-off on Ephrata's explainer (for Amir's question)

**Asker:** Amir Ahmedin
**Explainer:** Ephrata Nebiyu
**Date:** Day 2, Week 12
**Judgment:** Closed ✓

### What I understand now that I did not before

Before this explainer, I treated "function-calling" as a single thing — either it's constrained decoding or it's prompt injection. I couldn't explain why my `complete_json()` fallback chain sometimes works perfectly and sometimes catches real parse failures.

Now I understand there are **four distinct levels**, not two:

1. **Plain prompted JSON** (my current approach) — full vocabulary available at every step. Model can emit fences, think blocks, wrong keys. My fallback chain is correct and necessary.

2. **JSON mode** (`response_format: json_object`) — guarantees valid JSON syntax but NOT my specific schema. Model can still return wrong keys or wrong value types. My fallback chain is partially needed (syntax is safe, schema is not).

3. **Tool calling without strict** — provider adds protocol structure + model is trained on tool-call patterns. Much better than prompting, but still learned behavior, not proof-by-construction. Fallback chain is mostly unnecessary but not dead code.

4. **Tool calling with strict structured outputs** (`strict: true`) — constrained decoding at the token level. Illegal schema-breaking tokens are masked before sampling. My fallback chain IS dead code for this mode.

The key insight Ephrata named: "The fact that you need to strip fences, remove think blocks, and salvage substrings is itself the clue — those repairs only make sense if the model had permission to emit tokens outside the intended object." My repair code is *evidence* of the decoding mode, not just defensive engineering.

The revised explainer also addressed the OpenRouter relay question honestly: I should treat deletion of the fallback chain as a **testable hypothesis**, not an assumption. The concrete test procedure (run eval set on both paths, compare parse/key/type failures) gives me an immediate next step.

The one-sentence version I'll carry forward: **"A prompt can encourage a model to output the right key. A decoder constraint can forbid it from outputting the wrong key. For workflow safety, forbidding beats encouraging."**

---

## Ephrata's sign-off on Amir's explainer (for Ephrata's question)

**Asker:** Ephrata Nebiyu
**Explainer:** Amir Ahmedin
**Date:** Day 2, Week 12
**Judgment:** Closed ✓

### What Ephrata understands now that he did not before

The real value of the centralized orchestrator is not just "keeping the flow organized," but enforcing system-wide invariants that distributed handlers cannot reliably maintain from local context alone. The most useful part: tool eligibility is a function of **authoritative global lifecycle state**, not of the isolated event a single handler happens to observe, and `_transition()` + `_is_duplicate_event()` + `allowed_next_channels()` together form the core policy surface.

Ephrata can now explain why retries, duplicate delivery, and partial failures become dangerous when policy is distributed: each handler sees only a projection of state, while the orchestrator can make one consistent decision against the shared state record.

### Why this is closed

The updated explainer gave the mechanism he was missing: the orchestrator is the policy layer that protects authoritative shared state, monotonic transitions, duplicate suppression, and centralized tool eligibility against the partial local views that distributed handlers would act on. Connected back to the real repo: `_transition()` enforces forward lifecycle movement, `_is_duplicate_event()` protects against replay when identifiable webhook deliveries repeat, and `allowed_next_channels()` keeps channel eligibility tied to global state rather than local handler context. That is enough to defend the architecture unaided.
