# Sources — Day 4

**Topic:** Evaluation and statistics — non-response bias and systematic abstention  
**Explainer:** Amir Ahmedin (answering Birkity's question)  
**Date:** Day 4, Week 12

## Canonical Sources

### Source 1: Rubin's Missing Data Taxonomy
**Rubin, D.B.** "Inference and Missing Data." *Biometrika*, 63(3), 581–592, 1976. https://doi.org/10.1093/biomet/63.3.581

Defines the MCAR/MAR/MNAR taxonomy — the foundational framework for understanding when dropping missing data biases your estimate. Every evaluation paper that reports "accuracy on tasks the model answered" is implicitly assuming MCAR (missingness is independent of the outcome). Rubin's framework provides the formal conditions under which this assumption holds and what happens when it doesn't.

**Key insight used:** Birkity's 51% abstention rate with clustering on hard tasks is textbook MNAR — the missingness mechanism (unparseable output from model uncertainty) is correlated with the outcome (being wrong). Under MNAR, accuracy-on-answered is an upper bound on true accuracy, and no unbiased correction exists without strong assumptions.

### Source 2: Little and Rubin Textbook
**Little, R.J.A. and Rubin, D.B.** *Statistical Analysis with Missing Data*, 2nd edition. Wiley, 2002.

The textbook treatment of missing data theory. Chapter 1 provides the intuition for the three regimes. Chapter 4 covers formal tests for MCAR (Little's MCAR test). The chi-squared test for equal abstention rates across subgroups used in the explainer is a simplified version of Little's test for categorical data.

**Key insight used:** The detection procedure (Check 1: abstention rate by subgroup) is a practical implementation of testing MCAR by checking whether missingness rates are independent of observed covariates. If they're not independent, MCAR is rejected and the conditional accuracy estimate is biased.

## Tool / Pattern Used

### Hands-on: Three-check detection procedure

Implemented a three-step diagnostic for Birkity's abstention pattern:

1. **Abstention rate by subgroup** — groups tasks by source_mode/difficulty, computes abstention rate per group. If rates vary (0% on synthetic, 100% on TB-PROG-* hard), MCAR is rejected by inspection. Runtime: 2 minutes.

2. **Accuracy by subgroup** — within answered tasks, computes accuracy per source_mode. If accuracy varies by the same features that predict abstention (high on easy subgroups, low on hard subgroups that partially answered), the MAR→MNAR signal is present. Runtime: 2 minutes.

3. **Manual inspection of 5 UNKNOWN outputs** — classifies each as "verdict present but unparseable" (parser fix) vs "model genuinely confused" (capability gap). This qualitative check determines whether the 51% abstention rate is a measurement artifact or a real capability boundary. Runtime: 10 minutes.

**Key finding:** The three checks together determine (a) whether the abstention is systematic (Check 1), (b) whether it correlates with capability (Check 2), and (c) whether the fix is in the parser or the model (Check 3). For Birkity's data, Checks 1-2 are conclusive from the clustering pattern alone; Check 3 determines the engineering response.

The coverage × accuracy identity (`accuracy_on_all = accuracy_on_answered × coverage_rate`) provides the single honest metric: 59.4% × 49% = 29.1%. This is mathematically equivalent to accuracy-on-all and should be reported as the primary number when abstention rate exceeds 20%.
