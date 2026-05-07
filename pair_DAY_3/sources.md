# Sources — Day 3

**Topic:** Training and post-training mechanics — SimPO gradient dynamics on thin hard sets  
**Explainer:** Amir Ahmedin (answering Yonas's question)  
**Date:** Day 3, Week 12

## Canonical Sources

### Source 1: SimPO Paper
**Meng, Xia, and Chen.** "SimPO: Simple Preference Optimization with a Reference-Free Reward." NeurIPS 2024. https://arxiv.org/abs/2405.14734

The loss derivation and γ/β sensitivity analysis. The per-pair gradient magnitude `σ(-s)` where `s = β·M - γ` follows directly from differentiating the SimPO loss. The 0.5-0.75 γ/β range Yonas uses is the paper's recommended region for instruction-tuned models. Appendix B tests down to ~1,000 pairs but does not characterize the "thin hard set" regime where the base model already separates 98% of pairs.

**Key insight used:** The gradient decay is `1/(1+exp(s))` — a sigmoid gate that creates a cliff between "full signal" (s<0) and "silent" (s>3). This is the mechanism that makes most of Yonas's 618 pairs invisible to the optimizer from step 0, concentrating all learning on the ~2% residual.

### Source 2: DPO Paper (for the KL contrast)
**Rafailov, Sharma, Mitchell, Manning, Ermon, Finn.** "Direct Preference Optimization: Your Language Model is Secretly a Reward Model." NeurIPS 2023. https://arxiv.org/abs/2305.18290

The reference-model term that SimPO removes. DPO's gradient includes a `π_ref(y)/π_θ(y)` importance weight that naturally decays updates on pairs where the policy has moved far from the reference. SimPO removes this term entirely — which is why the margin-widening regime (after all pairs are correctly ranked) continues unchecked. The reference model acts as a brake on over-optimization; without it, the only stopping condition is when the sigmoid gate kills the gradient (s >> 0).

**Key insight used:** The absence of DPO's reference term is what allows SimPO to keep widening margins on already-correct pairs. This is not harmful at Yonas's scale (rank-16 LoRA on 3B model = small distortion), but it explains why loss keeps dropping even after all pairs are correctly ranked.

## Tool / Pattern Used

### Hands-on: Per-pair margin histogram diagnostic

Wrote `ablations/diagnose_per_pair_margins.py` — a script that loads base Qwen2.5-3B (no adapter), computes length-normalized log-prob margin on all 618 train pairs, and cross-references with adversarial probe labels (P007/P011/P027).

The script resolves Yonas's core question in one run: "Did training have gradient signal on the adversarial cases I built the bench for, or were they passengers from step 0?"

Implementation: forward pass on each pair's chosen and rejected completions, compute per-token log-probs via `log_softmax` on logits, mask prompt tokens, length-normalize the response portion, subtract rejected from chosen to get margin M, compute slack `s = β·M - γ`, and flag pairs where `s < 0` (real gradient signal) vs `s > 0` (passenger).

Runtime: ~15 minutes on Colab T4 for 618 pairs at max_length=512. The output is a JSONL with per-pair margins and a summary printing the count of hard pairs, adversarial pairs, and their intersection.

**Key finding from code analysis:** With 98% base accuracy and fast convergence (loss 0.016 in 150 steps), the hard set is estimated at ~12 pairs. Each hard pair appears in exactly 2 batches across the entire run (2 epochs, one appearance per epoch) — so at most 2 gradient updates before saturating. The histogram confirms whether those ~12 include the adversarial probes or not — that's the load-bearing diagnostic.
