# Tweet Thread — What Does Your LLM Latency Number Actually Consist Of?

**Author:** Amir Ahmedin
**Date:** Day 1, Week 12

## Tweet 1

Your agent reports "106 ms latency per task."

You put this in your memo. Your model card makes speed claims based on it.

But what IS that number? Can you defend it if someone asks "what would cut it in half?"

Here's what wall-clock latency actually decomposes into 🧵

## Tweet 2

Your "106 ms" contains FOUR components, not one:

1. Network round-trip (~10-30 ms)
2. Queue wait (0-100+ ms, variable)
3. Prefill — process input tokens in parallel (~5-20 ms)
4. Decode — generate output tokens one-by-one (~30-80 ms)

Only decode scales with output length. Prefill scales with INPUT length.

## Tweet 3

"Would reducing max_completion_tokens from 350 to 150 cut latency in half?"

Almost certainly NO.

max_completion_tokens is a ceiling, not a target. If your model naturally stops at 80 tokens, setting the max to 350 or 150 makes zero difference.

You only save time if the model was actually hitting the ceiling.

## Tweet 4

To decompose your latency, measure Time-To-First-Token (TTFT):

```python
# Use streaming to reveal TTFT
stream = client.chat.completions.create(..., stream=True)
# Time from request → first token = network + queue + prefill
# Time from first token → last token = pure decode
```

If TTFT is 70ms and decode is 36ms → prefill-dominated. Fewer output tokens won't help.

## Tweet 5

The deeper insight: prefill is compute-bound (parallel), decode is memory-bound (sequential).

This is why they scale differently:
- Longer prompt → longer prefill
- More output tokens → longer decode
- Faster GPU compute → faster prefill
- Faster memory bandwidth → faster decode

A single "latency" number hides which bottleneck you're hitting.

## Tweet 6

If your model card makes inference-speed claims, report:
- TTFT (time to first token)
- Tokens/second (decode speed)
- What dominates YOUR specific workload

Not just "106 ms per task" — that's indefensible without decomposition.

Full explainer with code: [blog link]
