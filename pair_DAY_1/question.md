# Day 1 Question — Inference-Time Mechanics

**Asker:** Amir Ahmedin
**Topic:** Inference-time mechanics
**Date:** Day 1, Week 12

## The Question

In my Week 10 sales agent (`llm_client.py`, line 79), I strip Qwen3's `<think>` reasoning-trace tokens after generation — the model produces them, I throw them away before passing the response to the next agent step. These think tokens caused 60% of my baseline's max-steps failures by inflating turn counts.

**Are Qwen3's `<think>` reasoning-trace tokens included in the `completion_tokens` count returned by the API, do they occupy KV cache slots during generation, and am I paying for tokens I strip and discard?**

If they are billed as output tokens at $8/M, then my cost-per-task ($0.15) is correct but I'm wasting money on tokens I never use. If they also occupy KV cache during generation, they inflate the context window for subsequent tokens and slow down decode. Either way, the engineering response changes: suppress at inference time (e.g., `think_mode=off` or logit bias) rather than post-process.

## Context — Why This Requires Understanding KV Cache Mechanics

This question sits inside a larger inference-time cost structure I cannot fully explain. My policy-aware prompting mechanism reduced cost from $0.53 to $0.15 per task and tokens from 2,113,578 to 1,179,597 — a 72% cost drop — by cutting conversation turns. Each turn re-sends the full conversation history, which means the KV cache grows turn-over-turn. Whether the provider re-computes attention for all prior tokens (full prefill) or reuses the KV cache via prefix caching determines the true cost of each turn. The think-token question is the sharpest entry point into this cost structure because it's the one place where I'm making a concrete engineering choice (strip vs suppress) without understanding the billing and cache implications.

## Connection to Existing Work

Knowing this would let me revise:

- **`llm_client.py`** (Week 10) — Line 79 strips `<think>` blocks after generation. If think tokens are billed and occupy KV cache, I should suppress them at inference time rather than post-processing — a direct code change.
- **`memo.md`** (Week 10) — The CFO memo's cost-per-task derivation ($0.15) and cost-per-qualified-lead ($0.07) assume the API's token counts are the full picture. If discarded think tokens are billed, the memo needs a "hidden cost" line item.
- **`eval/method.md`** (Week 10) — The cost analysis table comparing baseline vs mechanism ($0.53 vs $0.15) would gain a think-token waste breakdown showing how much of each cost is reasoning tokens that never reach the user.
- **`probes/target_failure_mode.md`** (Week 10) — The revenue impact calculation ($50K–$190K/mo from reasoning loops) should include the actual token cost of those wasted think blocks, not just the turn-count cost.

## Why This Matters Beyond My Work

Every FDE running multi-turn agents through API providers (OpenRouter, Together, etc.) faces the same invisible cost structure. Most practitioners report `completion_tokens` without knowing whether reasoning-trace tokens are inside that number. The difference between "tokens used" and "tokens billed" — and whether those hidden tokens also consume KV cache capacity — determines whether a production agent's unit economics are viable. This is especially acute for reasoning models (Qwen3, DeepSeek-R1, o1) where think tokens can be 2–5x the visible output.
