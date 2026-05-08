# Thread — Your LLM Judge Reports 59% Accuracy. The Real Number Is 29%.

**Platform:** LinkedIn / X  
**Date:** Day 4, Week 12

---

**Post 1/5**

Your LLM judge scored 59.4% accuracy on held-out tasks.

But it only answered 32 of 65 tasks. The other 33 returned UNKNOWN — unparseable output.

If you count abstentions as failures: 19/65 = 29.2%.

A 30-percentage-point gap between "accuracy on tasks it answered" and "accuracy on all tasks." Which number do you trust? 🧵

---

**Post 2/5**

The mechanism is simple conditional probability:

```
accuracy_reported = correct / answered = 19/32 = 59.4%
accuracy_true     = correct / total   = 19/65 = 29.2%
```

These are equal ONLY if the judge is equally likely to abstain on tasks it would get right vs wrong.

If it preferentially abstains on tasks it would get wrong (which is what happens when hard tasks produce unparseable output), the conditional estimate is biased upward. By construction.

---

**Post 3/5**

Why does the judge abstain specifically on hard tasks? Two causal paths:

1️⃣ **Uncertainty → verbosity → parse failure.** Model hedges: "On one hand... but on the other..." No clean PASS/REJECT token. Parser fails.

2️⃣ **Complexity → reasoning overflow → structural breakdown.** Hard tasks need longer chains of thought. Output exceeds expected structure. Parser can't resolve.

Both paths: abstention correlates with being wrong. The model doesn't "choose" to abstain — its failure mode just happens to produce unparseable output.

---

**Post 4/5**

The detection is 3 checks:

✅ **Check 1:** Group tasks by difficulty/source. If abstention rate is 0% on easy tasks and 100% on hard tasks → not random. (2 minutes)

✅ **Check 2:** Compare accuracy across subgroups that DID answer. If easy subgroups score 90% and hard subgroups score 30% → abstention correlates with capability.

✅ **Check 3:** Read 5 UNKNOWN outputs. Is the verdict in there but unparseable (parser fix) or genuinely absent (model fix)?

---

**Post 5/5**

The reporting rule:

- Abstention < 5%: report accuracy-on-answered. Fine.
- Abstention 5-20%: report BOTH numbers side by side.
- Abstention > 20%: accuracy-on-answered is not credible. Report accuracy-on-all as primary.

The single metric that captures both: **accuracy × coverage.**

59.4% × 49% = 29.1%. That's your real number.

Reporting accuracy without coverage is like reporting precision without recall. Both together or neither alone.

Sources: Rubin (1976) MCAR/MAR/MNAR taxonomy. Full explainer: [blog link]

---
