# Sources — Day 1

**Explainer:** Amir Ahmedin
**Date:** Day 1, Week 12

## Canonical Sources

1. **"Efficient Memory Management for Large Language Model Serving with PagedAttention"** — Woosuk Kwon, Zhuohan Li, Siyuan Zhuang, Ying Sheng, Lianmin Zheng, Cody Hao Yu, Joseph E. Gonzalez, Hao Zhang, Ion Stoica. SOSP 2023. https://arxiv.org/abs/2309.06180 — Introduces paged KV cache in vLLM. The paging mechanism stores KV blocks in non-contiguous memory pages, enabling efficient prefix sharing across requests without memory duplication. This is the foundation that makes prefix caching practical at scale.

2. **"SGLang: Efficient Execution of Structured Language Model Programs"** — Lianmin Zheng, Liangsheng Yin, Zhiqiang Xie, Jeff Huang, Chuyue Sun, Cody Hao Yu, Shiyi Cao, Christos Kozyrakis, Ion Stoica, Joseph E. Gonzalez, Hao Zhang. 2024. https://arxiv.org/abs/2312.07104 — Introduces RadixAttention, which uses a radix tree data structure to automatically detect and reuse shared prefixes across requests. No explicit user configuration needed — the engine identifies common prefixes and serves them from cache. This is the mechanism most likely active when calling models through providers that use SGLang.

## Tool / Pattern Used (Hands-On)

**Experiment: Prefix cache hit verification via OpenRouter.**

Made two identical API calls to the same model (Qwen3) through OpenRouter with an identical system prompt and user message. Compared time-to-first-token between calls. On the second call, time-to-first-token was measurably lower when the request hit the same backend, indicating prefix cache reuse. Then modified the system prompt by one token — time-to-first-token returned to baseline, confirming cache invalidation on any prefix change.

Key observation: OpenRouter's response includes a `provider` field indicating which backend served the request. When consecutive calls hit the same provider, cache hits are likely. When they hit different providers, no caching benefit regardless of prompt structure.
