# Grounding Commit — Day 3

**Asker:** Amir Ahmedin  
**Date:** Day 3, Week 12

## Artifact Edited

**File:** `tenacious-sales-bench/training/prepare_training_data.py`  
**Change:** Added scorer-gating to the pair generation loop — no pair enters `training_data.jsonl` unless its chosen side passes `scoring_evaluator.py`.

## What Changed and Why

Before this edit, `get_chosen()` returned one of 16 hand-authored templates with name/company/language interpolation, producing only 111 unique chosen strings across 599 pairs. The model learned to separate chosen from rejected by template-surface markers ("to be upfront," "I want to be honest") rather than by the factual constraints the held-out scorer checks (bench honesty, timezone correctness, banned phrases). This is why the trained model (pass rate 30%) lost to both the prompted base (58%) and a pure regex heuristic (34%).

After learning from Yonas's explainer that my chosen-side data IS my implicit reward function — and a 16-template reward function cannot generalize — I added two changes:

1. **Scorer-gating:** After `get_chosen()` generates a candidate, it's validated against `scoring_evaluator.py`. If `passed_all_checks` is false, the pair is dropped. This aligns the training reward with the held-out reward at the data level.

2. **TODO marker for LLM generation:** Added a comment block marking `get_chosen()` for replacement with LLM-generated outputs (DeepSeek V3 or GPT-4o) in the next training run. The scorer gate is necessary but not sufficient — template-based generation with gating still produces low diversity. The full fix requires both LLM generation (diversity) and scorer gating (alignment).

The edit also adds a diversity check at the end of `main()`: prints the unique-chosen-to-total-pair ratio and warns if it falls below 0.5 (i.e., if more than half the pairs share a chosen string with another pair).

```python
# Added after pair generation, before saving:
unique_chosen = len(set(p["chosen"] for p in all_pairs))
ratio = unique_chosen / len(all_pairs)
print(f"\nChosen diversity: {unique_chosen} unique / {len(all_pairs)} total = {ratio:.2f}")
if ratio < 0.5:
    logger.warning(
        "DIVERSITY WARNING: chosen-side ratio %.2f < 0.5 — "
        "model will likely learn template-surface markers rather than rubric constraints. "
        "Consider replacing get_chosen() with LLM generation + scorer gating.",
        ratio,
    )
```

Current ratio: 111/599 = 0.19. The warning would fire immediately, making the problem visible to anyone running the pipeline.
