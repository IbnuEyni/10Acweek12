# Explainer — Your 59.4% Accuracy Is Survivorship Bias With a Statistics Name

**Written by:** Amir Ahmedin  
**For:** Birkity Yishak  
**Topic:** Evaluation and statistics — non-response bias and systematic abstention in LLM evaluation  
**Date:** Day 4, Week 12

## The Question

Birkity's SimPO judge returns UNKNOWN on 33 of 65 held-out tasks (51%). Reported accuracy is 59.4% (19/32 answered correctly). If abstentions count as wrong, true accuracy is 29.2% (19/65). The abstentions aren't random — they cluster entirely on TB-PROG-\* hard tasks and TB-TRACE-021–025, while zero synthetic edge cases abstain. Her question: how do you detect whether abstentions are systematic, and how does that change which accuracy number to trust?

## The Core Intuition: Survivorship Bias

Imagine a school that reports "95% of our students pass the final exam" — but 40% of students dropped out before the exam. The 95% is true _conditional on surviving to the exam_. It tells you nothing about the school's actual teaching quality. The dropouts were the students who would have failed. By disappearing from the denominator, they made the survivors look better.

Birkity's judge is doing exactly this. It "drops out" on hard tasks (TB-PROG-\* hard, TB-TRACE-021–025) and "survives" on easy tasks (synthetic semantic edge cases). The 59.4% accuracy is the school's "95% pass rate" — true for the survivors, meaningless as a measure of the judge's actual capability.

This has a formal name in statistics: **non-response bias**. And the framework for reasoning about it comes from Rubin (1976), who classified missing data into three regimes based on _why_ the data is missing.

## The Mechanism: Why Dropping Abstentions Inflates Accuracy

The `_accuracy` function computes:

```
accuracy_reported = correct / answered = 19 / 32 = 59.4%
```

But the quantity you actually want is:

```
accuracy_true = correct / total = 19 / 65 = 29.2%
```

The relationship between them:

```
accuracy_true = accuracy_reported × coverage
             = 59.4% × (32/65)
             = 59.4% × 49.2%
             = 29.2%
```

When you drop abstentions from the denominator, you're computing a **conditional probability**: P(correct | answered). This equals P(correct) — the unconditional accuracy you want — **only if** answering is independent of being correct. Written formally:

```
P(correct | answered) = P(correct)   ← only true if P(answered | correct) = P(answered | wrong)
```

In words: the reported accuracy equals true accuracy only if the judge is equally likely to abstain on tasks it would get right vs tasks it would get wrong. If the judge preferentially abstains on tasks it would get wrong (which is what Birkity's clustering suggests), then:

```
P(answered | correct) > P(answered | wrong)
```

and the conditional estimate is **biased upward**. The bias magnitude is:

```
bias = P(correct | answered) - P(correct)
     = 59.4% - 29.2%
     = 30.2 percentage points
```

This 30-point gap is not a statistical fluke — it's a structural consequence of the abstention pattern. Every task that abstains and would have been wrong inflates the numerator's share of the denominator. The more wrong-tasks abstain, the better the survivors look.

**The mechanism at the model level:** Why does the judge produce unparseable output specifically on hard tasks? Two causal paths:

1. **Uncertainty → verbosity → parse failure.** When the model is uncertain about the verdict, it hedges: "On one hand... but on the other hand... it's difficult to say definitively..." This produces long, equivocating outputs that don't contain a clean PASS/REJECT token. The parser fails not because the model refused to answer, but because uncertainty manifests as format violations.

2. **Task complexity → reasoning overflow → structural breakdown.** Hard tasks (TB-PROG-\* with ambiguous product claims, TB-TRACE-021–025 with multi-step reasoning) require longer chains of thought. The model's output exceeds the expected structure, interleaves reasoning with verdict, or produces partial JSON that the parser can't resolve. The structural breakdown is caused by the same task complexity that would cause a wrong answer.

Both paths produce the same statistical signature: abstention correlates with being wrong. The model doesn't "choose" to abstain — its failure mode (uncertainty, complexity overflow) happens to produce outputs the parser can't resolve. The abstention is a _symptom_ of the same underlying cause (task difficulty) that would produce wrong answers.

## Three Regimes of Missingness

### MCAR — Missing Completely At Random

The probability of abstention is the same for every task, regardless of task properties or the true answer. Like a network glitch that randomly drops 10% of API calls — it hits easy and hard tasks equally.

**If MCAR holds:** accuracy-on-answered is an unbiased estimate of true accuracy. You can safely drop the missing data. The 59.4% would be trustworthy.

**How to know you're in MCAR:** The abstention rate should be roughly equal across all task subgroups (source_mode, difficulty, category). If you split tasks by any feature and the abstention rate is ~51% in every subgroup, MCAR is plausible.

**Birkity's data:** Not MCAR. Abstention rate is 100% on TB-PROG-\* hard tasks, 100% on TB-TRACE-021–025, and 0% on synthetic edge cases. The rates vary wildly by subgroup. MCAR is rejected by inspection alone — you don't even need a statistical test.

### MAR — Missing At Random

The probability of abstention depends on _observed_ task features (difficulty, source_mode, category) but not on the _true answer_ itself. Like a student who drops out because the course is hard (observed feature), not because they specifically would have failed the final (unobserved outcome).

**If MAR holds:** accuracy-on-answered is biased, but _correctable_. You can reweight: compute accuracy within each subgroup that has both answered and abstained tasks, then combine using the full population's subgroup proportions. This gives you an estimate of what accuracy would have been if the judge had answered everything.

**How to know you're in MAR:** Within each subgroup that has some answered tasks, the accuracy should not depend on how close the task was to the abstention boundary. MAR says "once I know the task is hard, whether the judge abstains or answers is random." If the hardest _answered_ tasks within a subgroup have the same accuracy as the easiest answered tasks in that subgroup, MAR is plausible.

**Birkity's data:** Possibly MAR for some subgroups (if the few TB-PROG tasks that did answer have similar accuracy to synthetic tasks). But the 100% abstention rate on entire subgroups means you have no answered tasks to estimate accuracy for those subgroups — MAR correction is impossible without at least some answered tasks in every subgroup.

### MNAR — Missing Not At Random

The probability of abstention depends on the _true answer itself_ — the judge abstains _because_ it would get it wrong. Like a student who drops out specifically because they know they'll fail the final, not just because the course is hard.

**If MNAR holds:** accuracy-on-answered is an **upper bound** on true accuracy. There is no unbiased correction without strong assumptions. The judge is selectively showing you its successes and hiding its failures. The only honest report is the range: [accuracy-on-all, accuracy-on-answered] = [29.2%, 59.4%], with a note that the true value is somewhere in between but you cannot determine where without observing the abstained verdicts.

**How to know you're in MNAR:** You can't directly test it — that's the fundamental problem. MNAR means the missingness depends on something you don't observe (the true answer). But you can make it _plausible_ or _implausible_ by reasoning about the mechanism. In Birkity's case: the judge abstains on hard tasks. Hard tasks are the ones it's most likely to get wrong. The abstention mechanism (unparseable output) is plausibly triggered by the model being uncertain — uncertain models produce hedged, malformed, or rambling outputs that parsers can't resolve. This is the signature of MNAR: the model's uncertainty (which correlates with being wrong) causes the abstention.

## The Detection Procedure

Three checks, in order. Each builds on the previous.

**Check 1: Abstention rate by subgroup — kill MCAR in 2 minutes.**

Group tasks by source_mode, difficulty, category. If abstention rates vary dramatically across groups (0% here, 100% there), MCAR is dead.

```python
import json
from collections import defaultdict

results = json.load(open("ablations/ablation_results.json"))

# Group by source_mode (adapt field names to your JSON structure)
groups = defaultdict(lambda: {"answered": 0, "abstained": 0})
for task in results["per_task"]:  # adapt to your actual structure
    mode = task.get("source_mode", "unknown")
    if task["verdict"] == "UNKNOWN":
        groups[mode]["abstained"] += 1
    else:
        groups[mode]["answered"] += 1

print(f"{'Source Mode':<25} {'Answered':>8} {'Abstained':>9} {'Abstention %':>12}")
print("-" * 58)
for mode, counts in sorted(groups.items()):
    total = counts["answered"] + counts["abstained"]
    rate = counts["abstained"] / total if total else 0
    print(f"{mode:<25} {counts['answered']:>8} {counts['abstained']:>9} {rate:>11.1%}")

# If rates vary wildly (0% vs 100%), MCAR is rejected by inspection.
```

For Birkity's data, this will show ~100% abstention on TB-PROG-\* hard and TB-TRACE-021–025, and ~0% on synthetic. MCAR is dead on arrival.

**Check 2: Accuracy by subgroup — detect the MAR→MNAR signal.**

Within answered tasks, check if accuracy varies by the same features that predict abstention. If the judge scores high on easy subgroups and low on the few hard tasks it answered, abstention correlates with capability.

```python
# For each source_mode, compute accuracy only on answered tasks
print(f"\n{'Source Mode':<25} {'Accuracy':>8} {'n_answered':>10}")
print("-" * 47)
for mode, tasks in grouped_by_mode.items():  # group your per_task list
    answered = [t for t in tasks if t["verdict"] != "UNKNOWN"]
    if answered:
        correct = sum(1 for t in answered if t["correct"])
        acc = correct / len(answered)
        print(f"{mode:<25} {acc:>7.1%} {len(answered):>10}")
    else:
        print(f"{mode:<25} {'N/A (all abstained)':>20}")
```

If synthetic tasks score 90% and the few programmatic tasks that answered score 30%, the judge abstains where it's weakest. This is the MAR→MNAR signal: abstention is correlated with being wrong, not just with task difficulty.

**Check 3: Manually inspect 5 UNKNOWN outputs — parser vs capability.**

Read 5 UNKNOWN outputs. Two possible findings:

- **Verdict is present but unparseable:** "I think this is a PASS because the agent acknowledged the bench gap..." but no clean PASS token the parser can extract. This is a **parser problem** — fixable in 30 minutes with a regex fallback or structured output mode. The 51% abstention rate is a measurement artifact, not a capability boundary.

- **Model is genuinely confused:** Hedged, rambling, contradictory output with no clear verdict. This is **MNAR** — the model's uncertainty causes both the wrong answer and the unparseable output. The abstention IS the failure, not a separate problem from the failure.

```python
# Quick inspection of UNKNOWN outputs
unknowns = [t for t in results["per_task"] if t["verdict"] == "UNKNOWN"]
for t in unknowns[:5]:
    print(f"\n{'='*60}")
    print(f"Task: {t['task_id']} | Source: {t.get('source_mode')} | Difficulty: {t.get('difficulty')}")
    print(f"Raw output (first 300 chars):")
    print(t.get("raw_output", "[no output logged]")[:300])
    print(f"\n→ Verdict present but unparseable? Or genuinely confused?")
```

This qualitative check determines whether the fix is a better parser (measurement problem) or a better model (capability problem).

## What This Means for Reporting

The distinction between regimes determines which number is credible:

| Regime                                                                                 | What to report                        | Why                                                              |
| -------------------------------------------------------------------------------------- | ------------------------------------- | ---------------------------------------------------------------- |
| MCAR (abstention rate <5%, uniform across subgroups)                                   | Accuracy-on-answered                  | Unbiased estimate; abstentions are noise                         |
| MAR (abstention clusters on features, but some answered tasks exist in every subgroup) | Reweighted accuracy + abstention rate | Correctable bias; report the correction method                   |
| MNAR (entire subgroups abstain, or abstention correlates with difficulty)              | **Both numbers** + the range          | No unbiased correction possible; report the uncertainty honestly |

Birkity is in MNAR territory. The honest report is:

> "Judge accuracy: 29.2%–59.4%. The lower bound assumes all abstentions are failures; the upper bound assumes abstentions are random. Abstentions cluster on TB-PROG-\* hard tasks and TB-TRACE-021–025 (100% abstention rate), suggesting the true value is closer to the lower bound. Abstention rate: 51%."

The 59.4% alone is not a credible performance number at 51% abstention with systematic clustering. It is survivorship bias with a statistics name.

## The Deeper Point: Coverage × Accuracy

A judge that answers 49% of tasks at 59.4% accuracy has an **effective accuracy** of 59.4% × 0.49 = 29.1% — which is mathematically identical to accuracy-on-all (29.2%, the rounding difference is noise). This isn't a coincidence. It's the identity:

```
accuracy_on_all = accuracy_on_answered × coverage_rate
```

This means: reporting accuracy-on-answered without coverage is like reporting precision without recall. Both numbers together tell the full story. Either number alone is misleading. A judge with 95% accuracy on 10% of tasks is worse than a judge with 60% accuracy on 100% of tasks — but the first one's "accuracy" headline looks better.

The product (accuracy × coverage) is the single metric that captures both. For Birkity: 59.4% × 49% = 29.1%. For a judge that answers everything at 50% accuracy: 50% × 100% = 50%. The second judge is more useful despite lower conditional accuracy.

## Sources & Pointers

1. **Rubin, D.B., "Inference and Missing Data," Biometrika, 1976.** https://doi.org/10.1093/biomet/63.3.581 — Defines the MCAR/MAR/MNAR taxonomy. The foundational framework for understanding when dropping missing data biases your estimate. Every evaluation paper that reports "accuracy on tasks the model answered" is implicitly assuming MCAR — Rubin's framework is the tool for checking whether that assumption holds.

2. **Little, R.J.A. and Rubin, D.B., _Statistical Analysis with Missing Data_, Wiley, 2002 (2nd ed).** — The textbook treatment. Chapter 1 gives the intuition; Chapter 4 covers tests for MCAR. The chi-squared test for equal abstention rates across subgroups is a simplified version of Little's MCAR test for categorical data.

3. **Hands-on diagnostic:** Inspect 5 UNKNOWN outputs from Birkity's held-out run. Classify each as "verdict present but unparseable" (parser fix) vs "model genuinely confused" (capability gap). This qualitative check determines whether the 51% abstention rate is a measurement artifact (fixable in 30 minutes with a regex fallback) or a real capability boundary (requires model improvement). The statistical framework tells you what the numbers mean; the manual inspection tells you what to do about it.
