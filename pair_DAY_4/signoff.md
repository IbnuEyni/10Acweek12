# Sign-off — Day 4

**Asker:** Amir Ahmedin  
**Pair:** Birkity Yishak  
**Topic:** Evaluation and statistics — inter-rater agreement under skewed base rates  
**Grounding commit:** `inter_rater_agreement.md` protocol rename + kappa reporting (full diff in [`grounding_commit.md`](grounding_commit.md))

## Status: PARTIALLY CLOSED

The core gap (why 91.7% raw agreement is misleading and what to report instead) is closed. The secondary gap (PABAK/AC1 mechanism, threshold for "too ambiguous") remains open.

## What I understand now that I did not before

My `inter_rater_agreement.md` reports 91.7% raw agreement as evidence that the rubric is well-specified. Birkity's explainer showed me three things that undermine that headline:

First, the protocol name is wrong. I have one rater labeling twice with a 24-hour gap — that's **intra-rater test-retest reliability**, not inter-rater agreement. The claim it supports is narrower: "one person applies the rubric consistently over time." It does NOT support "the rubric is clear to other evaluators." This is a real limitation I was papering over with the wrong terminology.

Second, chance agreement at my base rate (92.7% "correct" in pass 1, 90.2% in pass 2) is already 84.4%. My 91.7% is only 7.3 percentage points above what I'd get by labeling everything "correct" without reading the tasks. Cohen's kappa of 0.461 reveals this: only 46% of the possible agreement above chance is captured. The "moderate" interpretation is honest — my rubric consistency is real but not as impressive as 91.7% sounds.

Third, the kappa paradox means 0.46 doesn't automatically destroy credibility — it's a known artifact of extreme prevalence. But it does mean I cannot report raw agreement alone. The defensible report is the bundle: observed agreement, base rates, kappa, and a prevalence-aware companion (PABAK = 0.834).

**What remains open:** I can compute PABAK (`2p - 1 = 0.834`) but cannot explain *why* this formula corrects for prevalence or defend it against "why not just use kappa?" I also don't have a threshold for when kappa on a specific dimension (workflow_correctness at ~0.3-0.4) means "rubric is too ambiguous for training." These are follow-up gaps, not blockers for the grounding commit.

## What changed in my Week 11 portfolio

- Renamed `inter_rater_agreement.md` header from "Inter-Rater Agreement" to "Intra-Rater Test-Retest Reliability" with a note explaining the distinction.

- Added a "Statistical Context" section reporting: observed agreement (91.7%), base rates (92.7%/90.2%), chance agreement (84.4%), Cohen's kappa (0.461, "moderate"), and PABAK (0.834). Replaced the "91.7% proves the rubric is well-specified" claim with the narrower: "One rater was mostly consistent; the label distribution was highly skewed; kappa shows the chance-agreement problem."

- Added a note to `methodology.md` Quality Assurance section: "The >80% raw agreement threshold is below expected chance agreement (84.4%) at this base rate. Future iterations should set thresholds in kappa terms (e.g., κ > 0.6) or use prevalence-adjusted metrics."

- Annotated `model_card.md` Calibration Note: replaced "91.7% overall agreement exceeds the 80% threshold" with "Intra-rater test-retest reliability was 91.7% (κ=0.461). High base rate (>90% 'correct' labels) inflates raw agreement; see inter_rater_agreement.md for full statistical context."
