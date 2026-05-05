# Evening Call Summary — Day 1 (Geresum)

**Triad:** Amir Ahmedin, Mistire Daniel & Geresum Asfaw
**Date:** Day 1, Week 12
**Duration:** ≥20 min

## Feedback on Amir's explainer (from Geresum)

- **What landed:** The four-component decomposition (network, queue, prefill, decode) and which levers each responds to. The insight that `max_completion_tokens` is a ceiling not a request — reducing it only helps if the model is hitting the cap. The observation that the 17 ms delta between baseline and trained component is likely attributable to longer input or longer output. The streaming approach (`stream=True`) for exact TTFT measurement.
- **What didn't land:** No issues raised — all five points of understanding were confirmed as clear.
- **What was revised:** No revisions needed.

## Feedback on Geresum's explainer (from Amir)

- **What landed:** The three-universe framing made the conceptual space clear. The counterintuitive experimental result (thinking ON = 710 tokens, thinking OFF = 896 tokens) completely changed the engineering decision from "obviously suppress" to "measure first." The corrected answer — "measure what you're buying" — reframes the gap from a binary choice to a product decision. The token-level trace confirmed by the Qwen3 technical report.
- **What didn't land:** The explainer is ~1,500 words, over the 600–1,000 target. The "Revising the Memo" section writes the revision for me rather than letting me do it.
- **What was revised:** Tightened length by condensing the memo revision suggestion and adjacent concepts section.

## Sign-off status

- Geresum's gap: closed
- Amir's gap (from Geresum's explainer): closed
