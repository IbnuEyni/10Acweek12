# Explainer — Does Your System Prompt Get Re-Computed Every Call?

**Written by:** Amir Ahmedin
**For:** Mistire Daniel
**Topic:** Inference-time mechanics — KV cache and prefix caching
**Date:** Day 1, Week 12

## The Question

Mistire's outreach agent sends the same ~83-line system prompt on every LLM call. The agent is multi-turn — each conversation turn re-sends the full message history including this fixed prefix. His question: is the LLM re-computing the KV cache for those system prompt tokens on every single call, or does prefix caching mean he only pays that prefill cost once? And what exactly breaks the cache?

This matters because if prefix caching isn't active, every turn in every multi-turn task is paying full prefill cost for tokens that never change — a hidden cost component in his $0.53 baseline that he cannot currently explain or defend.

## The Mechanism

LLM inference has two phases: **prefill** and **decode**.

During **prefill**, the model processes all input tokens in parallel. For each token at each transformer layer, it computes a Key vector and a Value vector. These KV pairs are stored in GPU memory — this is the KV cache. Prefill is the expensive part: it scales with input length and is compute-bound.

During **decode**, the model generates output tokens one at a time. Each new token computes its own Query vector and attends against the cached Keys and Values. It never recomputes KV for previous tokens — it just reads from the cache and appends its own KV pair.

**Prefix caching** exploits a simple observation: if two requests share the same token prefix (e.g., the same system prompt), their KV cache entries for that prefix are identical. The inference engine hashes the prefix tokens, checks if those KV blocks already exist in GPU memory from a recent request, and if so, skips prefill for the cached portion. Only the new tokens after the prefix need computation.

For Mistire's 83-line system prompt: if prefix caching is active and the cache is warm, the LLM skips prefill for those tokens on the second call onward. It only computes KV for the new user message and conversation history appended after the system prompt.

## What Causes a Cache Miss

Five conditions force a full recompute:

1. **Any token change in the prefix.** The cache key is the exact token sequence. One added space, one reordered word, one different timestamp injected into the system prompt — the cache invalidates from that point forward. Mistire's system prompt must be byte-identical across calls.

2. **Different GPU/worker.** The KV cache lives in a specific GPU's memory. If the inference provider load-balances the request to a different GPU than the one that served the previous call, there's no cache to hit. This is the most common miss in production.

3. **Time expiry / memory pressure.** Cached KV blocks are evicted when the GPU needs memory for other requests. TTL is typically seconds to minutes under load, not hours. A burst of traffic from other users can evict Mistire's cached prefix.

4. **Provider doesn't support it.** Not all serving engines implement prefix caching. vLLM (automatic prefix caching), SGLang (RadixAttention), and TensorRT-LLM support it. A provider using a basic HuggingFace serving setup may not.

5. **Different model.** Each model has its own KV cache. Switching from Qwen3 to GPT-4o means a completely different cache space.

## Demonstration

A concrete example of how prefix caching affects Mistire's agent:

```
Call 1: [system_prompt (83 lines) + user_msg_1]
  → Full prefill: compute KV for ALL tokens (system + user)
  → KV cache stored on GPU-A

Call 2: [system_prompt (83 lines) + user_msg_1 + assistant_reply + user_msg_2]
  → Cache HIT on GPU-A: skip prefill for system_prompt tokens
  → Only compute KV for: user_msg_1 + assistant_reply + user_msg_2
  → Savings: ~83 lines of prefill skipped

Call 2 (cache MISS — routed to GPU-B):
  → Full prefill: recompute KV for ALL tokens from scratch
  → No savings
```

The billing implication: providers that support prefix caching (like Fireworks, Together via vLLM) often charge cached input tokens at 50% or even 0% of the normal input rate. If Mistire's calls hit the cache, his effective input cost for the system prompt drops to near zero after the first call. If they miss, he pays full input rate every time.

To verify whether caching is active, check the API response for `cache_read_input_tokens` or similar fields — some providers report cache hits explicitly.

## Adjacent Concepts

**1. Prompt structure determines cacheability.** The system prompt must come first and remain unchanged. If Mistire injects dynamic content (timestamps, user IDs) into the system prompt, he breaks the prefix. Dynamic content should go in the user message, after the fixed prefix.

**2. Multi-turn history grows the non-cached portion.** Even with perfect prefix caching on the system prompt, the conversation history after it grows every turn and is never cached (it changes each turn). This is why policy-aware prompting — which reduces turn count — saves cost even when prefix caching is active.

**3. Provider routing is the hidden variable.** When accessing models through OpenRouter, the request may be routed to different underlying providers (Together, Fireworks, Lepton, etc.) across calls. Each provider has its own GPU fleet and cache state. Consistent routing (pinning to one provider) increases cache hit rate.

## Sources & Pointers

1. Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP 2023. https://arxiv.org/abs/2309.06180 — Introduces paged KV cache in vLLM, the foundation that makes prefix caching memory-efficient by storing KV blocks in non-contiguous pages.

2. Zheng et al., "SGLang: Efficient Execution of Structured Language Model Programs," 2024. https://arxiv.org/abs/2312.07104 — Introduces RadixAttention, which uses a radix tree to automatically detect and reuse shared prefixes across requests without explicit user configuration.

3. **Hands-on verification:** Call the same model through OpenRouter twice with an identical system prompt + identical user message. Compare `prompt_tokens` and time-to-first-token between calls. If the provider supports prefix caching and the request hits the same GPU, the second call's time-to-first-token should be measurably lower (prefill skipped for the cached prefix).
