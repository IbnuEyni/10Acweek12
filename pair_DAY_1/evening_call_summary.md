# Evening Call Summary — Day 1

**Pair:** Amir Ahmedin & Mistire Daniel
**Date:** Day 1, Week 12
**Duration:** ≥20 min

## Feedback on Amir's explainer (from Mistire)

- **What landed:** The five cache miss conditions were specific and actionable. Provider routing as the hidden variable on OpenRouter was the key insight — his system prompt is already cache-friendly, the problem is whether OpenRouter pins him to the same GPU. The prefill/decode split was at the right level. The billing implication (cached tokens at 50% or 0%) connects directly to his baseline cost numbers.
- **What didn't land:** The hands-on verification step ("call the same model twice, compare TTFT") is described but no actual output is shown — he doesn't know what to look for in the response object. A sample response showing `cache_read_input_tokens` or equivalent would make it concrete. The explainer doesn't resolve which of OpenRouter's underlying providers actually support prefix caching for his specific setup.
- **What was revised:** Added a note clarifying that the `provider` field in OpenRouter responses identifies which backend served the request, and that Fireworks and Together (vLLM-based) are the most likely to support prefix caching.

## Feedback on Mistire's explainer (from Amir)

- **What landed:** The "costs you twice" framing (billed + KV cache inflation) clicked immediately. The `include_reasoning=True` code example is directly usable as a diagnostic. The fix (`enable_thinking=False` via `extra_body`) is actionable — a direct code change to `llm_client.py`. The vLLM issue #1826 caveat flagged a production trap I wouldn't have known about.
- **What didn't land:** No worked cost example using my actual numbers. I know I'm paying for think tokens, but I don't know what proportion of my $0.15/task is waste.
- **What was revised:** Added a paragraph estimating think-token waste using Amir's actual numbers ($0.15/task, ~1.18M tokens, $8/M output rate).

## Sign-off status

- Amir's gap: closed
- Mistire's gap: closed
