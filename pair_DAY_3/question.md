# Day 3 Question — Training and Post-Training Mechanics

**Asker:** Amir Ahmedin  
**Topic:** Training and post-training mechanics  
**Date:** Day 3, Week 12  
**Pair:** Yonas Eshete

## The Question (as submitted for voting)

In my Week 11 Tenacious-Bench judge training (`training/train_simpo.py`), I trained a SimPO adapter on Qwen2.5-1.5B-Instruct with LoRA rank=16, γ=1.5, and 599 preference pairs. Training converged to 100% reward accuracy and a reward margin of 1.461 on the eval split — yet on the sealed held-out partition (50 tasks), the trained judge (avg score 0.568, pass rate 30%) is outperformed by a zero-shot prompted base model (avg score 0.647, pass rate 58%). Delta B is -0.079.

**My 599 chosen outputs collapse to 111 unique strings generated from a 16-template library in `get_chosen()`, my eval split is a random 10% draw from the same template pool as training (`train_test_split(test_size=0.1, seed=42)`), and my trained model (pass rate 30%) loses to a pure regex heuristic baseline (34%). Given this, what does preference optimization actually learn when the chosen-side signal is template-collapsed — and what is the minimum structural change to my data pipeline that would make any preference-training recipe capable of generalizing to a held-out scorer that checks factual constraints (bench honesty, timezone fabrication, banned phrases) rather than template-surface patterns?**

A satisfying answer would tell me: (1) why template-collapsed chosen data causes the trained model to learn surface markers rather than rubric-satisfying behavior, (2) whether this is a fundamental limitation of preference optimization on low-diversity data or a fixable pipeline problem, and (3) what the minimum data-pipeline change is — scorer-gated chosen generation, diverse chosen sources beyond templates, or a structurally different eval split — that would make the next training run capable of beating the prompted base on held-out.

## Sharpening Responses (from Yonas's interrogation)

**Q: Your 599 chosen outputs collapse to 111 unique strings. Each chosen string appears in ~5.4 pairs on average. Have you looked at the per-template pair count? If 30 templates account for 80% of the chosen pairs, you trained the model to recognize those 30 surface patterns, and almost everything else is noise.**

I verified this. 111 unique chosen, 273 unique rejected — confirmed. The top 30 chosen strings account for 60.3% of all pairs (not 80%, but still dominant). Only 3 chosen strings are singletons. The top 2 strings each appear 34 times — both are the `workflow_correctness` default template ("Hi {name}, thanks for getting back to me...") with different names interpolated. So yes: my "599 preference pairs" are really "111 surface patterns ranked above 273 rejected patterns," and the model learned to recognize those patterns rather than the underlying rubric constraints.

**Q: Your eval split and your train split come from the same template pool (`train_test_split(test_size=0.1, seed=42)`). So your monotonic eval-loss curve is measuring how well the model learned the templates, not how well it generalizes. Are you sure your real question isn't "why did I trust eval loss as a generalization signal when my split was contaminated?"**

You're right — my eval loss is structurally incapable of detecting the held-out gap. The eval pairs share templates with the train pairs, so a model that memorizes templates will show perfect eval loss _and_ fail on held-out tasks that use different surface patterns. I was treating the clean curve (1.99 → 0.96) as evidence of convergence when it's actually evidence of template memorization. The real generalization signal would require an eval split drawn from _different_ templates or _different_ generation sources than the training set.

**Q: Your gamma sweep and your headline run disagree by 3× on reward margin (1.461 vs 4.021 at the same γ=1.5). Which one did you actually deploy? Which one is in your model card?**

The headline run (reward_margin=1.461, eval_loss=0.96) is the 599-pair run on the combined train+dev data. The gamma sweep (reward_margin=4.021, eval_loss=0.376) was the earlier 370-pair run. They're different datasets — the sweep was run first on 370 pairs, then I expanded to 599 pairs and retrained with γ=1.5 selected from the sweep. The model card reports the 599-pair run. But you're right that `methodology_rationale.md` says "γ=1.5 selected on lowest eval loss" without clarifying this was from the 370-pair sweep, not the 599-pair run. That's a documentation gap I need to fix.

**Q: Your trained model (pass rate 0.30) loses to a pure regex heuristic (0.34) by 4 points. Doesn't this suggest the failure isn't in SimPO's gradient at all, but in the fact that you trained the model to produce template-A surface markers that the scorer doesn't reward? Can any preference-optimization recipe close this gap given your current data?**

This is the sharpest version of the question. If the trained model is _worse_ than regex, it's not that SimPO failed to learn — it's that SimPO learned something the scorer doesn't value. The model learned to prefer outputs matching my 111 chosen templates, but those templates don't satisfy the held-out scorer's factual checks (bench honesty, timezone, banned phrases). The question becomes: is this a data problem (fixable by aligning chosen outputs with the scorer) or a fundamental limitation (preference optimization on template-collapsed data cannot generalize regardless of algorithm)?

**Q: Your part (a) asks about gradient mechanics in saturation — but your run isn't in saturation. Reward accuracy hits 1.0 at step 100, then margin keeps growing (0.514 → 1.304 → 1.461). The interesting question for YOUR run is: what is the gradient still doing on pairs that are already correctly ranked but whose margin keeps growing — and is that growth helpful or harmful for held-out generation?**

You're right — I mislabeled the regime. My model isn't saturated (loss would be ~0); it's in a regime where all pairs are correctly ranked but the loss is still pushing margins wider. The gradient is still active, still updating weights, still moving the output distribution — but toward what? If wider margins on template-matched pairs means the model is _more_ committed to template surface patterns and _less_ capable of general instruction-following, then the continued training after step 100 is actively harmful. But this is downstream of the data problem — even if I stopped at step 100, the model would still have learned template patterns rather than rubric constraints.

**Q: You're asking three questions but only one matters. Given the diagnostics above (chosen-string collapse, eval-split contamination, sweep vs headline mismatch, trained-vs-heuristic floor), my read is that the gradient questions are downstream of a data-pipeline failure. Tell me which question you want me to actually answer — the gradient one or the data-objective one.**

The data-objective one. You've convinced me. The gradient mechanics are moot if my 111 chosen templates don't satisfy the held-out scorer. The question I actually need answered is: **given that my chosen-side collapses to 111 template strings from a 16-template library, and my eval split is drawn from the same template pool as training, what does this imply for any preference-optimization recipe I run on this data — and what is the minimum structural change to my data pipeline that would make preference training capable of generalizing to the held-out scorer?**

## Connection to Existing Work

Knowing this would let me revise:

- **`training/prepare_training_data.py`** (Week 11) — The `get_chosen()` function uses 16 base templates producing only 111 unique chosen strings across 599 pairs. Fix: replace template generation with scorer-gated LLM generation — generate chosen outputs via DeepSeek/GPT-4, validate each against `scoring_evaluator.py`, reject failures, retry up to 3 times. No pair enters training unless its chosen side passes the same scorer that evaluates held-out.

- **`training/train_simpo.py`** (Week 11, Cell 5) — The `train_test_split(test_size=0.1, seed=42)` draws eval pairs from the same template pool as training, making eval loss structurally incapable of detecting generalization failure. Fix: replace with a held-out partition drawn from different generation sources (hand-authored adversarial tasks, not programmatic templates).

- **`methodology.md`** (Week 11) — The training configuration rationale assumes the preference signal is clean and diverse. Needs a "Data Diversity Requirements" subsection naming the minimum chosen-side diversity for preference training to generalize — not just pair count, but unique-chosen-to-total-pair ratio.

- **`model_card.md`** (Week 11) — The Limitations section says "599 pairs is below SimPO's tested minimum." Replace with the actual mechanism: 599 pairs collapsing to 111 unique chosen strings means the model learns template-surface recognition, not rubric-satisfying behavior. The limitation isn't pair _count_ — it's chosen-side _diversity_.

- **`ablations/ablation_results.json`** (Week 11) — Delta B (-0.079) is currently reported as "prompting matches or beats training" without diagnosis. Gets a root-cause annotation: trained model learned to prefer template-surface patterns that the held-out scorer doesn't reward.

## Why This Matters Beyond My Work

Every FDE training preference-optimized judges on small domain-specific datasets faces the same failure mode: training metrics look perfect, held-out performance degrades, and the instinct is to reach for gradient-level explanations — "switch to DPO," "raise LoRA rank," "add KL regularization." But if the chosen-side data is template-collapsed and the eval split shares the same templates, _no_ preference-optimization algorithm can generalize — the training signal itself doesn't contain the information the held-out scorer rewards. The pattern is: check chosen-side diversity and eval-split contamination _before_ touching the trainer. Most negative Delta B results in small-data preference training are data-pipeline failures disguised as algorithm failures.
