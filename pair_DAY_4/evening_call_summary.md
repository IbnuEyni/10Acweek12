# Evening Call Summary — Day 4

**Pair:** Amir Ahmedin & Birkity Yishak  
**Date:** Day 4, Week 12  
**Duration:** Async (network constraints continued — feedback exchanged via text)

## Feedback on Birkity's explainer (from Amir — for Amir's question)

**What landed:**

1. "Fix the protocol name" — intra-rater test-retest reliability, not inter-rater agreement. I was calling it "inter-rater" in my `inter_rater_agreement.md` when it's one person labeling twice. This narrows the claim correctly: consistent for its author ≠ clear to another evaluator.

2. The chance agreement arithmetic laid out explicitly: `(0.927 × 0.902) + (0.073 × 0.098) ≈ 0.844`. My 91.7% is only 7.3 points above what you'd get by labeling everything "correct." Seeing the calculation makes the inflation visceral.

3. The kappa paradox named: "Under extreme prevalence, kappa can look surprisingly low even when observed disagreement is small." Kappa isn't wrong at 0.46 — it's revealing that most of my agreement is explained by base rate. But it also penalizes me for having a rubric where most tasks genuinely ARE correct.

4. The reporting bundle — not "pick one metric" but report observed agreement + base rates + kappa + PABAK + contingency table. This is the concrete answer to "what should I write."

5. The revised writeup template — exact paragraph I can put in `inter_rater_agreement.md` that names the protocol correctly and doesn't overclaim.

**What didn't land (partially addressed):**

1. PABAK is introduced (`2 × observed agreement - 1 = 0.834`) but not explained. Why does this formula correct for prevalence? What assumption does it make that kappa doesn't? I can compute it but can't defend it if asked.

2. Gwet's AC1 is mentioned three times but never computed or explained. If I'm supposed to report it, I need to know how to compute it on my data and what number it produces.

3. The workflow_correctness section doesn't give a threshold. At what kappa does a dimension become "too ambiguous to use for training"? She says "the statistic alone cannot decide" but doesn't give a decision rule.

4. No code or runnable demonstration. The challenge spec requires hands-on verification — I'd have to implement kappa/PABAK/AC1 computation myself.

5. Doesn't address the methodology.md threshold problem: "Require >80% agreement" is below chance agreement (84.4%). Any rubric passes this threshold regardless of quality.

## Feedback on Amir's explainer (from Birkity — for Birkity's question)

**What landed:**

1. The survivorship bias framing — the school analogy makes the core intuition land immediately. "95% of students pass the final exam, but 40% dropped out before the exam" is the one-sentence version of my problem. I can explain this to anyone now.

2. The conditional probability mechanism: `P(correct | answered) = P(correct)` only if `P(answered | correct) = P(answered | wrong)`. This is the formal statement of why dropping abstentions inflates accuracy. The 30-point bias isn't a fluke — it's structural.

3. The two causal paths at the model level (uncertainty→verbosity→parse failure, complexity→reasoning overflow→structural breakdown) explain WHY my judge produces unparseable output specifically on hard tasks. The abstention isn't a choice — it's a symptom of the same difficulty that would cause wrong answers.

4. The MCAR/MAR/MNAR taxonomy with my data mapped to each regime. I'm clearly in MNAR territory — and the explainer makes it obvious why without me needing to run a formal test.

5. The coverage × accuracy identity: `59.4% × 49% = 29.1%`. This single formula collapses the two numbers into one honest metric. "Reporting accuracy without coverage is like reporting precision without recall" — that's the framing I'll use in my ablation results.

**What didn't land:**

1. The code snippets use placeholder field names (`results["per_task"]`, `task["verdict"]`, `task["correct"]`) that don't match my actual JSON structure. I need to adapt them to my `ablation_results.json` schema. The logic is right but not copy-paste-runnable.

2. The "Check 3: manually inspect 5 UNKNOWN outputs" section tells me to classify as "parser fix" vs "model fix" but doesn't give me a decision rule for what to do AFTER I classify them. If 3/5 are parser problems and 2/5 are genuine confusion, what's my next step? Fix the parser and re-run? Report both numbers regardless?

3. The reporting rule (abstention <5% → report on-answered; >20% → report on-all) is useful but the thresholds feel arbitrary. Why 5% and 20%? Is there a statistical basis or is this a convention?

## Sign-off status

- Amir's gap: partially closed (protocol naming + kappa interpretation landed; PABAK/AC1 mechanism and threshold question remain)
- Birkity's gap: closed (MCAR/MAR/MNAR framework + detection procedure + reporting rule)
