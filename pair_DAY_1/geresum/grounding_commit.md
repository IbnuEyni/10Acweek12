# Grounding Commit — Day 1 (Geresum)

**Asker:** Geresum Asfaw
**Date:** Day 1, Week 12

## Artifact Edited

**File:** `agent/llm/openrouter.py`
**Change:** Added logging of `usage.completion_tokens` after each API call to decompose the wall-clock latency into its components.

## What Changed and Why

Before this edit, `openrouter.py` captured `usage` in the `OpenRouterChatResult` dataclass but never inspected or logged it. The 106 ms latency reported in `memo.md` was an opaque wall-clock number with no decomposition. After learning that prefill scales with input length, decode scales with actual output tokens, and `max_completion_tokens` is a ceiling that only matters if the model hits it, I added instrumentation to log `completion_tokens` per call. This reveals whether the 106 ms is network-dominated (output tokens are short), decode-dominated (output tokens are near the 350 cap), or prefill-dominated (unlikely for 400–600 tokens on gpt-4.1-mini). The memo's latency claim can now be defended with data showing which component dominates.
