# Grounding Commit — Day 3

**Asker:** Amir Ahmedin  
**Pair:** Yonas Eshete  
**Topic:** Training and post-training mechanics  
**Repo:** `tenacious-sales-bench` (code push pending — changes staged locally)  
**How to verify:** After push, read `training/prepare_training_data.py` (scorer-gating + diversity warning), `model_card.md` (Limitations revision), and `ablations/ablation_results.json` (Delta B annotation). The diagnostic that motivated these changes is reproducible: `python3 -c "import json; from collections import Counter; pairs=[json.loads(l) for l in open('training/training_data.jsonl')]; c=Counter(p['chosen'] for p in pairs); print(f'Unique chosen: {len(c)}/{len(pairs)} = {len(c)/len(pairs):.3f}')"` → outputs `111/599 = 0.185`.

---

## What changed in the Week 11 portfolio

Three concrete edits in [`tenacious-sales-bench/`](../../tenacious-sales-bench/), all directly grounded in Yonas's explainer showing that my chosen-side data IS my implicit reward function — and a 16-template reward function cannot generalize.

### 1. Added scorer-gating + diversity warning to `training/prepare_training_data.py`

**Before** (no quality gate, no diversity check):

The `generate_pairs()` function calls `get_chosen(task)` which returns one of 16 templates with name/company/language interpolation. The output goes directly into `training_data.jsonl` without validation against the held-out scorer. No check on whether the chosen-side diversity is sufficient for preference training to generalize.

**After** (scorer-gated + diversity warning):

```python
# Added to the pair generation loop:
chosen = get_chosen(task)
# Scorer-gate: reject chosen outputs that don't pass the same scorer used on held-out
from evaluation.scoring_evaluator import TenaciousBenchEvaluator
evaluator = TenaciousBenchEvaluator()
result = evaluator.score_task(task, parse_agent_output(chosen))
if not result.passed:
    logger.warning("Chosen output failed scorer for task %s — dropping pair", task["task_id"])
    continue

# Added at end of main(), before saving:
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

Current ratio: 111/599 = 0.19. The warning fires immediately. Target after fix: ≥ 0.9 (the natural output of any LLM-call-per-task pipeline with scorer gating).

### 2. Marked `get_chosen()` for replacement with LLM generation

**Before:** `get_chosen()` is a template-lookup function with 16 base templates (4 tone + 3 resource_honesty + 3 signal_grounding + 6 workflow). Every chosen string is one of these plus `{name}`/`{company}`/`{lang}` interpolation.

**After:** Added a TODO block at the top of `get_chosen()`:

```python
def get_chosen(task: dict) -> str:
    """Generate a passing output for the task.
    
    TODO(day3-grounding): Replace this entire function with LLM-generated
    chosen outputs (DeepSeek V3 or GPT-4o) gated by scoring_evaluator.py.
    
    Current implementation produces only 111 unique strings from 16 templates
    (ratio 0.19). This causes preference training to learn template-surface
    markers rather than rubric constraints. Target: ratio >= 0.9.
    
    The scorer gate alone is necessary but not sufficient — template-based
    generation with gating still produces low diversity. The full fix requires
    both LLM generation (diversity) and scorer gating (alignment).
    
    See: pair_DAY_3/explainer.md from Yonas Eshete, "Your chosen-side data
    IS your reward function, and a 16-template reward function cannot generalize."
    """
    ...
```

### 3. Annotated Delta B in `ablations/ablation_results.json`

**Before:**

```json
"interpretation": "Prompting matches or beats training — honest finding, report in blog"
```

**After:**

```json
"interpretation": "Prompting beats training — root cause: chosen-side collapses to 111 unique strings from 16 templates (ratio 0.19). Gradient learned template-surface markers ('to be upfront', 'I want to be honest') rather than rubric constraints (bench honesty, timezone, banned phrases). Data-pipeline failure, not algorithm failure. Fix: LLM-generated chosen (diversity >= 0.9) + scorer-gating (alignment). See Day 3 explainer from Yonas Eshete."
```

### 4. Marked `model_card.md` Limitations section for revision

**Before:**

> 1. **Below SimPO's tested minimum:** SimPO Appendix B tests down to ~1,000 pairs. We trained on 599. Statistical significance was not achieved.

**After (pending full rewrite):**

> 1. **Chosen-side diversity collapse:** 599 pairs collapse to 111 unique chosen strings from a 16-template library (unique-to-total ratio: 0.19). The model learns template-surface recognition rather than rubric-satisfying behavior. The limitation is not pair *count* — it is chosen-side *diversity*. Target for v0.2: ratio ≥ 0.9 via LLM-generated chosen outputs with scorer gating. SimPO's tested minimum (~1,000 pairs) assumes diverse chosen-side data; our 599 pairs with ratio 0.19 are effectively ~111 unique training signals.

---

## What this changes about my Week 11 claim

Before this commit, my methodology assumed the preference signal was clean and diverse — the "Why SimPO Over DPO" section argued that reference-free optimization is an advantage, and the model card attributed the negative Delta B to insufficient pair count. Both claims were wrong in a way that no amount of algorithm-level reasoning could have revealed.

The actual failure: my `get_chosen()` function is a 16-template lookup. With 111 unique chosen strings sharing surface markers ("to be upfront," "I want to be honest," "I want to be transparent") and 273 unique rejected strings sharing the opposite markers ("Guaranteed!", "Consider it done"), the strongest separator in my data is cluster identity — not the factual constraints my held-out scorer checks. SimPO's gradient found that separator at step 100 (reward accuracy 1.0) and spent the remaining training widening margins on template-surface features the scorer doesn't reward.

The fix is not in the trainer. DPO's KL term would not have helped — it prevents distribution drift from the reference, but if the reference model also assigns higher log-prob to template-surface markers (which it does, because the templates are grammatically clean), the KL anchor just keeps the model near a policy that already prefers templates. The fix is upstream: diverse chosen-side data that cannot be separated from rejected by surface markers alone.

## What I learned that I did not know before this commit

I had been treating "number of preference pairs" as the binding constraint on training quality. Yonas showed me that the binding constraint is **chosen-side diversity** — specifically, the unique-chosen-to-total-pair ratio. My 599 pairs at ratio 0.19 are worse than 100 pairs at ratio 1.0 would be, because 100 diverse pairs give the gradient 100 different directions to learn from, while 599 collapsed pairs give it 111 directions repeated 5.4x each — reinforcing surface patterns rather than teaching generalizable rubric behavior.

The deeper insight: my eval split (`train_test_split(test_size=0.1)` from the same template pool) made this failure structurally undetectable during training. Eval loss dropped monotonically because the model was memorizing templates, and the eval set was made of the same templates. I had a clean training curve and a catastrophic held-out result, and no in-loop signal could have warned me. The fix for *detection* is as important as the fix for *generation*: the eval split must come from a different source than training, or eval loss is measuring memorization, not generalization.

Full signoff document: [`signoff.md`](signoff.md).
