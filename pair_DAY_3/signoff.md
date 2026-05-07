# Sign-off — Day 3

**Asker:** Amir Ahmedin  
**Pair:** Yonas Eshete  
**Topic:** Training and post-training mechanics — chosen-side diversity as implicit reward function  
**Grounding commit:** `prepare_training_data.py` scorer-gating + diversity warning (full diff and rationale in [`grounding_commit.md`](grounding_commit.md))

## Status: CLOSED

Yonas's explainer plus the verified data diagnostics together closed the gap.

## What I understand now that I did not before

I went into Day 3 asking what gradient-level mechanism causes SimPO to achieve perfect training metrics while degrading held-out performance — and whether the fix was a KL penalty, higher LoRA rank, or switching to DPO. Yonas's interrogation reframed the question entirely before the explainer was even written. Three things landed.

First, Yonas ran diagnostics against my actual `training_data.jsonl` and showed me the collapse: 599 pairs, 111 unique chosen strings, 273 unique rejected. The top 30 chosen strings account for 60.3% of all pairs. The top 2 strings each appear 34 times — both are the `workflow_correctness` default template with different names interpolated. My "599 preference pairs" are really "111 surface patterns ranked above 273 rejected patterns." The gradient found the lowest-entropy separator between the two clusters — template-surface markers ("Hi {name}, to be upfront" vs "Guaranteed!") — and optimized for that. Not for bench honesty. Not for timezone correctness. Not for the rubric. Template markers.

Second, the framing "your chosen-side data IS your implicit reward function" reframed the entire failure. I was thinking of SimPO's loss as the reward signal. But the loss only has access to "this string is chosen, that string is rejected." Whatever separates the two clusters becomes the gradient direction. With 16 templates producing 111 unique strings, the strongest separator is the template surface pattern itself. This is reward overoptimization (Gao et al. 2023) in a different form — a narrow reward (16 templates) produces a narrow policy (template-approximate outputs that fail the held-out scorer). No preference-optimization algorithm survives this setup. DPO, ORPO, KTO — all would converge to the same template-surface manifold because the data gives them no other signal.

Third, the diversity threshold gave me a concrete engineering target. Yonas's comparison: his pipeline (DeepSeek V3 chosen-rewrites, scorer-gated) yields ratio 1.0 with no regression. Mine yields ratio 0.185 with catastrophic regression. Target: ≥ 0.9. Danger floor: 0.5. This is the number I was missing — I can now write a quality gate that fires before training starts, rather than discovering the problem after 30 minutes of GPU time and a negative Delta B.

The combined understanding: my negative Delta B (-0.079) is not a training-mechanics failure. It's a data-pipeline failure that no algorithm can overcome. The eval split (`train_test_split(test_size=0.1)` from the same template pool) made the failure invisible during training — eval loss dropped monotonically because the model was memorizing templates, and the eval set was made of the same templates. The held-out scorer checks factual constraints (bench honesty, timezone, banned phrases) that my templates don't reliably satisfy. The fix is two changes in `prepare_training_data.py` (LLM generation + scorer gating), not anything in the trainer.

I was asking about SimPO gradients, KL penalties, and whether DPO's reference model is the regularizer I was missing. All of that is irrelevant. The trainer is fine. The data is the constraint.

## What changed in my Week 11 portfolio

- Added scorer-gating to the pair generation loop in [`training/prepare_training_data.py`](../../tenacious-sales-bench/training/prepare_training_data.py) — no pair enters `training_data.jsonl` unless its chosen side passes `scoring_evaluator.py`. This aligns the training reward with the held-out reward at the data level.

- Added a diversity check at the end of `main()` that prints the unique-chosen-to-total-pair ratio and warns if it falls below 0.5. Current ratio: 111/599 = 0.19. The warning fires immediately, making the problem visible to anyone running the pipeline before they waste GPU time.

- Added a TODO marker on `get_chosen()` for replacement with LLM-generated outputs (DeepSeek V3 or GPT-4o) in the next training run. The scorer gate is necessary but not sufficient — template-based generation with gating still produces low diversity. The full fix requires both LLM generation (diversity ≥ 0.9 ratio) and scorer gating (alignment with held-out).

- Annotated Delta B in [`ablations/ablation_results.json`](../../tenacious-sales-bench/ablations/ablation_results.json) with root cause: "trained model learned to prefer template-surface patterns (111 unique chosen from 16 templates) that the held-out scorer doesn't reward. Data-pipeline failure, not algorithm failure."

- Marked [`model_card.md`](../../tenacious-sales-bench/model_card.md) Limitations section for revision: replace "599 pairs is below SimPO's tested minimum" with the actual mechanism — the limitation isn't pair count, it's chosen-side diversity (ratio 0.19 vs minimum viable 0.9).

Full grounding-commit document: [`grounding_commit.md`](grounding_commit.md).
