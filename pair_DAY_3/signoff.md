# Sign-Off — Day 3

**Asker:** Amir Ahmedin  
**Explainer:** Yonas Eshete  
**Date:** Day 3, Week 12

## Gap Closure Status

**Status:** closed

## What I understand now that I didn't before

My negative Delta B (-0.079) is not a training-mechanics failure — it's a data-pipeline failure that no preference-optimization algorithm can overcome. My 599 preference pairs collapse to 111 unique chosen strings from a 16-template library in `get_chosen()`. The strongest separator between chosen and rejected in my data is the template-surface pattern ("Hi {name}, to be upfront" vs "Guaranteed! Consider it done!"), not the factual constraints my held-out scorer checks (bench honesty, timezone correctness, banned phrases). SimPO's gradient found that surface separator early (reward accuracy 1.0 at step 100) and optimized for it. The held-out scorer doesn't reward template markers — it checks factual content. So the trained model learned to produce template-approximate outputs that fail the scorer, while the prompted base model (which never saw my templates) generates diverse outputs that happen to satisfy the factual checks more often.

The fix is not in the trainer. It's in `prepare_training_data.py`: replace template-based `get_chosen()` with LLM-generated chosen outputs (for diversity) gated by `scoring_evaluator.py` (for alignment). Both changes are required together — diversity without gating produces misaligned data, gating without diversity produces the same template-collapse I have now. The eval split also needs replacement: `train_test_split(test_size=0.1)` from the same template pool makes eval loss structurally incapable of detecting generalization failure.

I was asking about SimPO gradients, KL penalties, and whether to switch to DPO. All of that is irrelevant. The trainer is fine. The data is the constraint.
