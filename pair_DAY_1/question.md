# Day 1 Question — Inference-Time Mechanics

**Asker:** Amir Ahmedin
**Topic:** Inference-time mechanics
**Date:** Day 1, Week 12

## The Question

In my Week 10 sales agent, the policy-aware prompting mechanism reduced cost per task from $0.53 to $0.15 and total tokens from 2,113,578 to 1,179,597 — a 72% cost drop and 44% token reduction — by cutting the number of conversation turns the agent needs to complete a task. But I cannot explain what happens at the inference level across those turns:

1. **How does the KV cache grow turn-over-turn in a multi-turn agent conversation?** Each new turn re-sends the full conversation history. Is the provider re-computing attention for all prior tokens (full prefill), or does prefix caching reuse the KV cache from the previous turn's shared prefix?

2. **Are Qwen3's `<think>` reasoning-trace tokens — which caused 60% of my baseline's max-steps failures — included in the `completion_tokens` count returned by the API, do they occupy KV cache during generation, and am I paying for tokens I strip and throw away in my `llm_client.py`?**

3. **Given the above, is my memo's cost derivation ($0.15/task at $2/M input, $8/M output) actually correct, or is it missing the cost of re-prefilled context and discarded reasoning tokens?**

## Connection to Existing Work

Knowing this would let me revise:

- **`memo.md`** (Week 10) — The CFO memo's cost-per-task derivation ($0.15) and cost-per-qualified-lead ($0.07) assume token counts from the API are the full picture. If prefix caching or think-token billing changes the math, the memo's numbers need correction.
- **`eval/method.md`** (Week 10) — The cost analysis table comparing baseline vs mechanism ($0.53 vs $0.15) would gain a prefill-vs-decode breakdown instead of just total tokens.
- **`llm_client.py`** (Week 10) — Line 79 strips `<think>` blocks after generation. If think tokens are billed, I should explore suppressing them at inference time rather than post-processing.
- **`probes/target_failure_mode.md`** (Week 10) — The revenue impact calculation ($50K–$190K/mo from reasoning loops) should include the actual token cost of those wasted think blocks.

## Why This Matters Beyond My Work

Every FDE running multi-turn agents through API providers (OpenRouter, Together, etc.) faces the same invisible cost structure. The difference between "tokens used" and "tokens billed" — and whether prefix caching applies — determines whether a production agent's unit economics are viable or not. Most practitioners report `completion_tokens` without knowing what's actually in that number.
