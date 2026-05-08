# Day 4 Question — Evaluation and Statistics

**Asker:** Amir Ahmedin  
**Topic:** Evaluation and statistics  
**Date:** Day 4, Week 12  
**Pair:** Birkity Yishak

## The Question (as submitted for voting)

In my Week 11 Tenacious-Bench inter-rater agreement protocol (`inter_rater_agreement.md`), I report 91.7% raw agreement (76/83 dimension-task pairs) across two labeling passes on 30 tasks. The challenge spec requires >80% agreement per dimension, and all four dimensions pass: signal_accuracy 100%, tone_adherence 95.2%, resource_honesty 88.2%, workflow_correctness 84.6%. I present this as evidence that my rubric is well-specified.

**But raw agreement doesn't account for chance agreement. With a high base rate (92.7% of labels are "correct" in pass 1, 90.2% in pass 2), chance agreement is already 84.4%. My Cohen's kappa is 0.461 — "moderate" agreement, not the "almost perfect" my 91.7% headline implies. The 91.7% is only 7.3 percentage points above what I'd get by labeling everything "correct" without reading the tasks. How should I interpret and report inter-rater agreement when the base rate is this skewed — and does a kappa of 0.46 on my rubric actually undermine the benchmark's credibility, or is raw agreement the correct metric for this specifpaic protocol (intra-rater consistency on a binary label with known high base rate)?**

A satisfying answer would tell me: (1) when kappa is the right metric vs when raw agreement is defensible (and which applies to my specific protocol — same rater, 24h gap, binary label), (2) whether 0.46 kappa on workflow_correctness specifically (the dimension with lowest agreement and the one that requires LLM judge) means my rubric is genuinely ambiguous or just that the task is inherently harder, and (3) what I should actually report in the inter-rater agreement section — raw agreement alone, kappa alone, both with interpretation, or a different metric entirely (e.g., prevalence-adjusted bias-adjusted kappa, PABAK).

## Connection to Existing Work

Knowing this would let me revise:

- **`inter_rater_agreement.md`** (Week 11) — Currently reports only raw agreement with no chance-correction. If kappa is the right metric, the section needs to report both raw agreement AND kappa, with interpretation of what "moderate" means for benchmark credibility. If PABAK is more appropriate for high-prevalence binary labels, that changes the reported number entirely.

- **`methodology.md`** (Week 11, "Quality Assurance" section) — States "Require >80% agreement on each rubric dimension" as the threshold. But 80% raw agreement with 84.4% chance agreement means the threshold is _below_ chance — any rubric would pass regardless of quality. The threshold needs to be stated in kappa terms, or the raw-agreement threshold needs to account for base rate.

- **`model_card.md`** (Week 11) — The "Calibration Note" says "91.7% overall agreement exceeds the 80% threshold." If kappa is 0.46, this sentence is misleading — it implies strong agreement when the chance-corrected agreement is only moderate.

- **`evaluation/scoring_evaluator.py`** (Week 11) — The heuristic fallback for workflow_correctness returns 0.5 ("uncertain") because "this dimension requires multi-step semantic reasoning." The low kappa on workflow_correctness (84.6% raw, likely kappa ~0.3-0.4) confirms this is the hardest dimension to specify — which validates the design decision to use LLM judge for it, but also means the rubric itself may be underspecified for training purposes.

- **`tenacious-sales-bench/datasheet.md`** (Week 11) — The datasheet should report inter-rater reliability with the appropriate metric. If I'm reporting raw agreement without kappa, a reviewer familiar with measurement theory will flag this as a gap.

## Why This Matters Beyond My Work

Every FDE building evaluation benchmarks faces the same reporting problem: raw agreement looks high when base rates are skewed, and most challenge specs (including ours) set thresholds in raw-agreement terms without accounting for chance. The result is that benchmarks pass quality checks that are structurally incapable of detecting rubric ambiguity. Understanding when kappa matters, when it doesn't (e.g., when prevalence is so extreme that kappa becomes paradoxically low even with genuine agreement), and what to report instead is a measurement-theory question that determines whether an evaluation benchmark's credibility claims are defensible under scrutiny.
