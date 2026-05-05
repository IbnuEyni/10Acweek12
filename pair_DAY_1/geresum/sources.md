# Sources — Day 1 (Geresum)

**Explainer:** Amir Ahmedin
**Date:** Day 1, Week 12

## Canonical Sources

1. **"Efficiently Scaling Transformer Inference"** — Reiner Pope, Sholto Douglas, Aakanksha Chowdhery, Jacob Devlin, James Bradbury, Jonathan Heek, Kefan Xiao, Shivani Agrawal, Jeff Dean. MLSys 2023. https://arxiv.org/abs/2211.05102 — Analyzes the prefill/decode split in transformer inference. Shows prefill is compute-bound (scales with input length, parallelizable) and decode is memory-bandwidth-bound (scales with output length, sequential). Provides the theoretical framework for understanding why reducing output tokens only helps if decode dominates.

2. **"Efficient Memory Management for Large Language Model Serving with PagedAttention"** — Woosuk Kwon et al. SOSP 2023. https://arxiv.org/abs/2309.06180 — vLLM paper. Section 2 explains KV cache memory management during decode, and why each decode step's latency grows with sequence length (attention over all cached KV pairs). Directly relevant to understanding why decode speed is not constant per token.

## Tool / Pattern Used (Hands-On)

**Streaming API call to measure TTFT (Time-To-First-Token).**

Used OpenAI-compatible streaming endpoint to separate wall-clock latency into TTFT (network + queue + prefill) and decode time (first token to last token). By dividing decode time by token count, derived per-token decode speed. This decomposition reveals whether a given latency measurement is prefill-dominated or decode-dominated — the key to answering whether reducing output tokens would reduce total latency.
