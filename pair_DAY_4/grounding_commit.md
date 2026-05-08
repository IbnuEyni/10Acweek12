# Grounding Commit — Day 4

**Asker:** Amir Ahmedin  
**Pair:** Birkity Yishak  
**Topic:** Evaluation and statistics  
**Repo:** `tenacious-sales-bench` (changes staged locally)  
**How to verify:** Read `inter_rater_agreement.md` (renamed header + statistical context section), `methodology.md` (threshold note), `model_card.md` (calibration note revision).

---

## What changed in the Week 11 portfolio

Three edits, all grounded in Birkity's explainer showing that raw agreement at high base rates is misleading without chance correction.

### 1. Renamed protocol and added statistical context in `inter_rater_agreement.md`

**Before (header):**

> # Inter-Rater Agreement

**After:**

> # Intra-Rater Test-Retest Reliability
>
> *Note: This protocol measures how consistently one evaluator applies the rubric over two passes (24h gap). It does not measure inter-rater agreement across independent raters. A rubric can be consistent for its author but unclear to another evaluator.*

**Before (conclusion):**

> The 91.7% overall agreement exceeds the 80% threshold required by the challenge specification.

**After:**

> **Statistical Context:**
>
> | Metric | Value | Interpretation |
> |---|---|---|
> | Observed agreement | 91.7% (76/83) | Actual match rate across passes |
> | Base rate (pass 1) | 92.7% "correct" | Most labels are "correct" — high prevalence |
> | Base rate (pass 2) | 90.2% "correct" | Consistent skew across passes |
> | Expected chance agreement | 84.4% | Agreement from marginal rates alone |
> | Cohen's kappa | 0.461 | "Moderate" — 46% of possible above-chance agreement |
> | PABAK | 0.834 | Prevalence-adjusted: `2 × 0.917 - 1` |
>
> The 91.7% raw agreement exceeds the challenge spec's 80% threshold, but chance agreement is already 84.4% at this base rate. Cohen's kappa of 0.461 reveals that most of the raw agreement is explained by label prevalence. The rubric is consistent for its author but the statistical evidence for rubric quality is weaker than the headline suggests.

### 2. Added threshold note to `methodology.md` Quality Assurance section

**Added after "Require >80% agreement on each rubric dimension":**

> *Note (Day 4, Week 12): The >80% raw agreement threshold is below expected chance agreement (84.4%) at this base rate — any rubric would pass regardless of quality. Future iterations should set thresholds in kappa terms (e.g., κ > 0.6 for "substantial" agreement) or use prevalence-adjusted metrics (PABAK > 0.7).*

### 3. Revised calibration note in `model_card.md`

**Before:**

> The 91.7% overall agreement exceeds the 80% threshold required by the challenge specification.

**After:**

> Intra-rater test-retest reliability was 91.7% observed agreement (κ=0.461, PABAK=0.834). The high base rate (>90% "correct" labels) inflates raw agreement; chance agreement alone is 84.4%. See `inter_rater_agreement.md` for full statistical context including per-dimension contingency tables.

---

## What this changes about my Week 11 claim

Before this commit, my benchmark's quality assurance section claimed "91.7% agreement proves the rubric is well-specified." This was overclaiming in two ways:

1. **Wrong protocol name:** Intra-rater consistency ≠ inter-rater agreement. My protocol shows one person is consistent, not that the rubric is clear to others.

2. **Raw agreement without chance correction:** At 84.4% chance agreement, my 91.7% is only 7.3 points above the floor. The kappa of 0.461 ("moderate") is the honest characterization — not "almost perfect" as the raw number implies.

The benchmark is not invalidated — the rubric IS consistent for its author, and the three disagreements were all resolved with rubric clarifications. But the statistical claim is now narrower and defensible rather than broad and misleading.

## What I learned that I did not know before this commit

I was treating "raw agreement > 80%" as a quality stamp without understanding that the threshold is meaningless at high base rates. A rubric where every task is labeled "correct" would achieve 80%+ agreement by construction — the threshold cannot distinguish a good rubric from a trivial one when prevalence is extreme. The fix is not just "report kappa" — it's understanding that the threshold itself needs to be stated in chance-corrected terms. My methodology.md now says this explicitly, so the next iteration of the benchmark uses a defensible threshold (κ > 0.6) rather than a structurally vacuous one (raw > 80%).
