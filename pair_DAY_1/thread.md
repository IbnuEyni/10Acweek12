# Tweet Thread — Does Your System Prompt Get Re-Computed Every Call?

**Author:** Amir Ahmedin
**Date:** Day 1, Week 12

## Tweet 1

Your multi-turn agent sends the same system prompt on every API call.

Is the LLM recomputing attention for those tokens every single time?

Or does prefix caching mean you only pay once?

Here's what actually happens at the GPU level 🧵

## Tweet 2

LLM inference = two phases:

**Prefill**: compute Key + Value vectors for ALL input tokens (expensive, parallel)
**Decode**: generate output tokens one-by-one using the cached KVs

Prefix caching: if two requests share the same token prefix, reuse the KV cache from the first request. Skip prefill for the shared part.

## Tweet 3

For a fixed system prompt, this means:

```
Call 1: full prefill (system + user) → KV stored on GPU
Call 2: cache HIT → skip system prompt prefill, only compute new tokens
```

Some providers charge cached tokens at 50% or 0% of input rate.

But only if the cache is warm and you hit the same GPU.

## Tweet 4

5 things that break the cache and force full recompute:

1. Any token change in the prefix (even one space)
2. Request hits a different GPU at the provider's data center (their load balancer picks — you have no control)
3. Cache evicted under memory pressure (seconds-minutes TTL)
4. Provider doesn't implement prefix caching
5. Different model = different cache

#2 is the most common miss in production. The KV cache lives on the provider's GPUs, not your machine.

## Tweet 5

The engineering implication: your system prompt must be byte-identical and come FIRST in every request.

Never inject timestamps, user IDs, or dynamic content into the system prompt. Put dynamic content in the user message — after the cacheable prefix.

This one structural choice determines whether you get cache hits or pay full prefill every call.

## Tweet 6

Sources:
- vLLM PagedAttention paper (Kwon et al., 2023): makes prefix caching memory-efficient
- SGLang RadixAttention (Zheng et al., 2024): automatic prefix detection via radix tree

Full explainer with worked examples: [blog link]

If you're running multi-turn agents through API providers, check your response headers for cache hit indicators. You might be paying 2x what you need to.
