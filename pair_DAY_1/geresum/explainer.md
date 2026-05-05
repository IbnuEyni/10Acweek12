# Explainer — What Does Your 106 ms Actually Consist Of?

**Written by:** Amir Ahmedin
**For:** Geresum Asfaw
**Topic:** Inference-time mechanics — prefill/decode latency decomposition
**Date:** Day 1, Week 12

## The Question

Geresum's agent calls `gpt-4.1-mini` via OpenRouter with 400–600 input tokens and `max_completion_tokens=350`. His memo reports "106 ms latency per task" but he cannot explain what that number consists of — and he has never inspected `usage.completion_tokens` to know how many tokens the model actually generates. His question forces a choice between three engineering levers he cannot currently distinguish: shorten prompts (if prefill dominates), reduce max tokens (if decode dominates and tokens hit the cap), or do nothing (if 106 ms is mostly network and the claim is already fine).

## The Mechanism

Wall-clock latency from request-send to response-read contains four sequential components:

```
|― Network ―|― Queue ―|― Prefill ―|――――― Decode ―――――|
                                    ↑                  ↑
                              first token         last token
←――――――――――――――――― 106 ms total ―――――――――――――――――――→
```

**Network round-trip (~15–40 ms):** Request travels to OpenRouter, then to OpenAI's servers, response travels back. Fixed regardless of prompt or output length.

**Queue wait (0–50 ms):** Request waits for a GPU slot. Near zero under low load, can spike under high load.

**Prefill (~3–8 ms for 400–600 tokens on gpt-4.1-mini):** The GPU processes all input tokens in parallel, computing KV pairs. Scales with input length. For gpt-4.1-mini (a small, fast model) with 400–600 tokens, this is very fast — single-digit milliseconds.

**Decode (~15–60 ms depending on actual output tokens):** The GPU generates output tokens one at a time. Each token takes ~0.3–0.5 ms on gpt-4.1-mini. If the model generates 50 tokens: ~20 ms. If it generates 150 tokens: ~60 ms. If it generates 350 tokens (hitting the cap): ~140 ms.

## Which Phase Dominates Geresum's 106 ms?

For `gpt-4.1-mini` with 400–600 input tokens, prefill is fast (~5 ms). The question is whether decode or network dominates. This depends entirely on **how many tokens the model actually generates** — the number Geresum has never inspected.

**Scenario A: Model generates ~50–80 tokens (likely for structured JSON output)**
```
Network:   30 ms
Queue:     10 ms
Prefill:    5 ms
Decode:    25 ms  (70 tokens × 0.35 ms/token)
―――――――――――――――
Total:     70 ms  ← close to 106 ms with some queue variance
```
→ **Network-dominated.** Neither shortening prompts nor reducing max_tokens helps. The memo claim is fine but should state "dominated by network round-trip."

**Scenario B: Model generates ~200+ tokens (verbose responses hitting near the cap)**
```
Network:   30 ms
Queue:      5 ms
Prefill:    5 ms
Decode:    80 ms  (220 tokens × 0.35 ms/token)
―――――――――――――――
Total:    120 ms  ← close to 106 ms
```
→ **Decode-dominated.** Reducing `max_completion_tokens` or prompting for shorter responses would cut latency.

**The answer: instrument `usage.completion_tokens` first.** Until Geresum knows the actual output token count, he cannot choose between the three levers.

## How to Instrument It

Geresum's `OpenRouterChatResult` already captures `usage` but never inspects it. The fix is one line of logging:

```python
# In openrouter.py, after parsing the response:
result = OpenRouterChatResult(
    content=...,
    latency_ms=latency_ms,
    usage=response_json.get("usage"),
    ...
)

# Add this logging:
if result.usage:
    print(f"[LLM] {latency_ms}ms | "
          f"prompt={result.usage['prompt_tokens']} | "
          f"completion={result.usage['completion_tokens']} | "
          f"decode_estimate={result.usage['completion_tokens'] * 0.35:.0f}ms")
```

After running 10 tasks with this logging, Geresum will see:
- If `completion_tokens` is consistently 50–80 → Scenario A (network-dominated)
- If `completion_tokens` is consistently 200+ → Scenario B (decode-dominated)

This single measurement resolves which lever to pull.

## The max_completion_tokens Misconception

`max_completion_tokens=350` is a **ceiling**, not a request. The model stops when it hits a stop token (end of response) OR the max — whichever comes first. If the model naturally produces 70 tokens of JSON, setting max to 350 or 150 makes zero difference to latency. The model already stopped at 70.

Reducing `max_completion_tokens` only cuts latency if:
1. The model is actually hitting the cap (generating exactly 350 tokens, getting cut off)
2. AND you're okay with truncated responses

If `usage.completion_tokens` shows 350 consistently, the model IS hitting the cap — and reducing it would cut decode time. If it shows 70, the cap is irrelevant.

## Adjacent Concepts

**1. Prefill is compute-bound, decode is memory-bound.** Prefill processes all tokens in parallel (GPU compute is the bottleneck). Decode generates one token at a time (reading KV cache from GPU memory is the bottleneck). This is why they scale differently and why "faster GPU" helps prefill but "more memory bandwidth" helps decode.

**2. The 17 ms delta between baseline (106 ms) and trained component (123 ms) is likely real.** If the trained component adds ~50 extra input tokens (adapter context or additional instructions), that's ~2 ms more prefill. The remaining 15 ms is likely more output tokens (the trained component generates longer responses). Instrumenting `completion_tokens` for both paths would confirm this.

**3. Streaming reveals the decomposition directly.** Using `stream=True` in the API call, the time from request to first token chunk = network + queue + prefill (TTFT). The time from first chunk to last chunk = pure decode. This gives exact decomposition without estimation.

## Sources & Pointers

1. Pope et al., "Efficiently Scaling Transformer Inference," MLSys 2023. https://arxiv.org/abs/2211.05102 — Analyzes the prefill/decode split in transformer inference. Shows prefill is compute-bound (parallel, scales with input) and decode is memory-bandwidth-bound (sequential, scales with output). The theoretical framework for understanding why the two phases respond to different engineering levers.

2. Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP 2023. https://arxiv.org/abs/2309.06180 — vLLM paper. Section 2 explains KV cache memory management during decode and why decode latency per token grows slightly with sequence length (attention over all cached KV pairs).

3. **Hands-on:** Instrument `usage.completion_tokens` in `OpenRouterChatResult` and run 10 tasks. Compare actual output token count against the 350 ceiling. Use streaming (`stream=True`) on one call to measure TTFT directly. These two measurements — actual token count and TTFT — resolve which of the three engineering levers applies.
