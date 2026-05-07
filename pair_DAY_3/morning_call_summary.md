# Morning Call Summary — Day 3

**Pair:** Amir Ahmedin & Yonas Eshete  
**Date:** Day 3, Week 12  
**Duration:** ≥20 min

## What was ambiguous in the original drafts

**Amir's draft:**
- Yonas ran diagnostics against Amir's actual `training_data.jsonl` and found the chosen-side collapses to 111 unique strings from a 16-template library (5.4x average repetition). He challenged whether the question should be about SimPO gradient mechanics at all, or about a data-pipeline failure upstream of any algorithm choice.
- Yonas flagged that Amir's eval split (`train_test_split(test_size=0.1, seed=42)`) draws from the same template pool as training, making eval loss structurally incapable of detecting generalization failure — the clean loss curve (1.99 → 0.96) is evidence of template memorization, not convergence.
- Yonas pointed out the trained model (pass rate 30%) loses to a pure regex heuristic (34%), suggesting the model learned template-surface markers the scorer doesn't reward — not that SimPO's gradient mechanics failed.
- Yonas forced a scope cut: "You're asking three questions but only one matters. The gradient one or the data-objective one?" Amir conceded: the data-objective one.

**Yonas's draft:**
- Amir flagged that Yonas's baseline pairwise accuracy is 98% and training moved it to 100% — only 2 pairs flipped — and asked whether Yonas knows which specific pairs flipped and whether they're adversarial (P007/P011/P027) or easy.
- Amir challenged whether the gradient question is even in the right regime: training loss of 0.016 means all pairs are saturated by epoch 2, so the question isn't "what happens in saturation" but "were the hard pairs ever unsaturated long enough to get real gradient before the easy ones dragged loss to zero."
- Amir asked how many of the 618 pairs have base-model margin below γ — if it's 5-10, the training signal is too thin for stable learning regardless of algorithm.
- Amir asked whether the real problem is data quality (chosen outputs that barely pass the evaluator), not gradient math. Yonas pushed back: his chosen side is scorer-gated (every chosen passes `scoring_evaluator.py`), unlike Amir's template-based pipeline. But conceded the log-prob signal can still be weak even with clean data.

## How each question was sharpened

**Amir's question:**
- Original: "What is the gradient-level mechanism by which SimPO achieves perfect training metrics while degrading held-out performance — reward hacking, catastrophic forgetting, or rank-constrained distortion?"
- Sharpened to: "Given that my chosen-side collapses to 111 template strings, my eval split is contaminated, and my trained model loses to regex — what does preference optimization learn on template-collapsed data, and what is the minimum data-pipeline change to make any preference recipe generalize to the held-out scorer?"
- The reframe moved the question from algorithm-level (SimPO gradient mechanics) to data-level (chosen-side diversity and objective alignment).

**Yonas's question:**
- Original: bundled three sub-questions (gradient decay math, hard-pair filtering principle, engineering decision between raising γ / hard-pair mining / expanding held-out).
- Sharpened to: "What fraction of my 618 pairs had base-model margin above γ before training, and for the pairs that started below γ, what was the per-step gradient trajectory before they saturated?" — with (b) and (c) collapsed into two consequence paragraphs at the end.
- Amir's reframe of the regime (not saturation, but "were hard pairs ever unsaturated long enough") was accepted and incorporated.

## Final confirmation

- [x] Amir confirms Yonas's question is unambiguous
- [x] Yonas confirms Amir's question is unambiguous
