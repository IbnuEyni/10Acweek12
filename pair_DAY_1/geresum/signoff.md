# Sign-Off — Day 1 (Geresum)

**Asker:** Geresum Asfaw
**Explainer:** Amir Ahmedin
**Date:** Day 1, Week 12

## Gap Closure Status

**Status:** closed

## What I understand now that I didn't before

My 106 ms wall-clock latency contains four sequential components — network round-trip, queue wait, prefill, and decode — each responding to different engineering levers. For my specific call shape (gpt-4.1-mini, 400–600 input tokens), prefill is fast (~5 ms) and the dominant component is either network or decode depending on how many tokens the model actually generates. I have never inspected `usage.completion_tokens` despite capturing it in my `OpenRouterChatResult` dataclass. `max_completion_tokens=350` is a ceiling not a request — it only affects latency if the model is hitting the cap. The immediate fix is instrumenting `completion_tokens` to determine which of the three engineering levers (shorten prompts, reduce max tokens, or accept the number as network-dominated) actually applies. Using `stream=True` gives exact TTFT decomposition without estimation.
