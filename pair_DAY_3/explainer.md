# Explainer — What SimPO's Gradient Actually Does When Your Base Model Is Already Right

**Written by:** Amir Ahmedin  
**For:** Yonas Eshete  
**Topic:** Training and post-training mechanics — SimPO gradient dynamics on thin hard sets  
**Date:** Day 3, Week 12

## The Question

Yonas trained a SimPO adapter on Qwen2.5-3B-Instruct (618 pairs, β=2.0, γ=1.5, LoRA rank=16). Training loss reached 0.016. Base pairwise accuracy was already 98% — training moved it to 100%, flipping a minimum of 2 pairs. On held-out (n=44), the lift is +2.27pp at p=0.372. He has 18 adversarial pairs (P007/P011/P027) in his dev eval window and cannot tell whether the flips landed on those or on easy pairs. His question: what fraction of the 618 pairs carried real gradient signal before training, and what was the per-step trajectory for those pairs before they saturated?

## The Mechanism: Per-Pair Gradient Decay

SimPO's loss for one pair:

```
L = -log σ(β·M - γ)
```

where `M` is the length-normalized log-prob margin (chosen minus rejected), β=2.0 scales it, γ=1.5 is the target. Define **margin slack** `s = β·M - γ`. The gradient magnitude:

```
|∂L/∂θ| ∝ σ(-s) = 1 / (1 + exp(s))
```

This is the entire story. The sigmoid gate determines how loud each pair is to the optimizer:

| Margin slack (s) | σ(-s) | Gradient strength | Meaning                                        |
| ---------------- | ----- | ----------------- | ---------------------------------------------- |
| -3.0             | 0.95  | 95% of max        | Pair far below target — screaming at optimizer |
| -1.0             | 0.73  | 73% of max        | Below target — strong signal                   |
| 0.0              | 0.50  | 50% of max        | Exactly at γ boundary                          |
| +1.0             | 0.27  | 27% of max        | Above target — fading                          |
| +3.0             | 0.05  | 5% of max         | Well above — nearly silent                     |
| +4.1             | 0.016 | 1.6% of max       | Your final average loss                        |
| +5.0             | 0.007 | 0.7% of max       | Functionally dead                              |

The decay is a cliff, not a slope. A pair goes from "full signal" to "silent" within ~4 units of slack. Once a pair crosses s=3, it will never meaningfully influence the model again.

## Illustrative Trajectory: One Hard Pair vs One Easy Pair

_The values below are constructed to show the gradient cliff dynamic. They are not computed from Yonas's actual per-pair margins. The real per-pair trajectory comes from running the diagnostic script in the next section._

**Pair A — adversarial (illustrative).** Suppose a P007 pair starts with M=0.3 (base model assigns similar log-prob to chosen and rejected). Then s = 2.0×0.3 - 1.5 = **-0.9**. Gradient strength: σ(0.9) = 0.71.

```
Start:    s=-0.9  gradient=0.71  ← real learning happening
After update 1: s≈+0.3  gradient=0.43  ← crossing γ, fading
After update 2: s≈+1.5  gradient=0.18  ← mostly silent
Final:    s≈+4.1  gradient=0.016 ← saturated
```

**Pair B — easy (illustrative).** Suppose a programmatic template pair starts with M=2.0. Then s = 2.0×2.0 - 1.5 = **+2.5**. Gradient strength: σ(-2.5) = 0.08.

```
Start:    s=+2.5  gradient=0.08  ← already silent
Final:    s≈+4.5  gradient=0.011 ← still silent, margin wider
```

Pair A got real gradient during its update events. Pair B was a passenger from step 0.

**The per-pair update count:** With 618 pairs, batch size 8, and 2 epochs, each pair appears in exactly 2 batches across the entire run (one appearance per epoch). Not 8, not 60 — **2 update events total per pair.** This is the binding constraint. The question is not "how many steps of signal did the hard pair absorb" but "was the pair still in the unsaturated regime (s < 0) at the moment it appeared in its 2 batches?" If yes, it got 2 full-strength gradient updates — enough to flip a pair from wrong to right, but not enough to build robust generalization. If it had already crossed γ by its second appearance, it got 1 real update and 1 no-op.

## What This Means for Yonas's 618 Pairs

With 98% base accuracy (2% wrong = ~12 pairs with negative margin on a similar distribution), the lower bound on the hard set is ~12 pairs with s < 0. The rest of the distribution — how many sit between 0 and +2 (moderate gradient) versus above +2 (silent) — is what the histogram will show. The diagnostic script below resolves the exact shape.

Each of those ~12 hard pairs got at most 2 gradient updates. Two updates at LR=5e-5 with full gradient signal (σ(-s) > 0.5) is mechanically sufficient to flip a pair from wrong to right — the LoRA weight delta is small but nonzero. Whether 2 updates produce _robust_ learning (generalization to held-out) or just a _fragile_ flip (correct on this exact pair, not on similar unseen pairs) depends on whether the gradient direction generalizes. With only ~12 hard pairs, the gradient direction is determined by a very thin slice of the data — making fragile flips more likely than robust learning.

**The critical unknown:** Were the 18 adversarial pairs (P007/P011/P027) in the ~12 hard-pair set, or in the passenger set? If the base model already separated adversarial chosen from adversarial rejected (just with smaller margin), they were passengers and training never touched them. If they had negative or near-zero margin, they got their 2 updates and may have flipped.

## The Diagnostic Script

This resolves the question in 15 minutes:

```python
# ablations/diagnose_per_pair_margins.py
import json, torch
from pathlib import Path
from transformers import AutoModelForCausalLM, AutoTokenizer

MODEL_ID = "unsloth/Qwen2.5-3B-Instruct"
BETA, GAMMA = 2.0, 1.5
ADVERSARIAL_PROBES = {"P007", "P011", "P027"}

model = AutoModelForCausalLM.from_pretrained(MODEL_ID, torch_dtype=torch.float16, device_map="auto")
tokenizer = AutoTokenizer.from_pretrained(MODEL_ID)

def length_norm_logprob(model, tokenizer, messages):
    """Compute length-normalized log-prob of the assistant response."""
    full_ids = tokenizer.apply_chat_template(messages, return_tensors="pt").to(model.device)
    prompt_ids = tokenizer.apply_chat_template(messages[:-1], add_generation_prompt=True, return_tensors="pt")
    prompt_len = prompt_ids.shape[1]
    with torch.no_grad():
        logits = model(full_ids).logits
    log_probs = torch.nn.functional.log_softmax(logits[:, :-1], dim=-1)
    token_lps = log_probs.gather(-1, full_ids[:, 1:].unsqueeze(-1)).squeeze(-1)
    resp_lps = token_lps[:, prompt_len-1:]
    return (resp_lps.sum() / resp_lps.shape[1]).item()

# Load pairs — adapt path to your repo layout
pairs = [json.loads(l) for l in open("training_data/train.jsonl")]

results = []
for i, pair in enumerate(pairs):
    lp_chosen = length_norm_logprob(model, tokenizer, pair["prompt"] + pair["chosen"])
    lp_rejected = length_norm_logprob(model, tokenizer, pair["prompt"] + pair["rejected"])
    margin = lp_chosen - lp_rejected
    slack = BETA * margin - GAMMA
    gradient_strength = 1 / (1 + torch.exp(torch.tensor(slack)).item())

    # Detect adversarial pairs via task_id in metadata
    task_id = pair.get("_meta", {}).get("task_id", "")
    is_adversarial = any(probe.lower() in task_id.lower() for probe in ADVERSARIAL_PROBES)

    results.append({
        "pair_idx": i, "task_id": task_id, "margin": margin, "slack": slack,
        "gradient_strength": gradient_strength, "is_adversarial": is_adversarial,
    })

# Summary
hard = [r for r in results if r["slack"] < 0]
adversarial = [r for r in results if r["is_adversarial"]]
adv_hard = [r for r in adversarial if r["slack"] < 0]

print(f"Total pairs: {len(results)}")
print(f"Hard pairs (s < 0): {len(hard)} ({len(hard)/len(results)*100:.1f}%)")
print(f"Adversarial pairs: {len(adversarial)}")
print(f"Adversarial AND hard: {len(adv_hard)}")
print(f"\nVerdict: {'Training had signal on adversarial cases' if adv_hard else 'Adversarial pairs were passengers — training never touched them'}")

# Save full results for histogram
with open("ablations/base_model_margins.jsonl", "w") as f:
    for r in results:
        f.write(json.dumps(r) + "\n")
print(f"Per-pair margins saved to ablations/base_model_margins.jsonl")
```

The output of this script is the answer. If `adv_hard > 0`, training had gradient signal on the adversarial cases. If `adv_hard == 0`, the +2.27pp came entirely from easy pairs and the adversarial set is exactly as the base model left it.

## Why Raising γ and Hard-Pair Mining Don't Help

**Raising γ** (e.g., γ=3.0) shifts the slack: pairs at old s=+1.5 become s=0, getting full gradient again. But these are pairs the model already ranks correctly — you're asking it to rank them _harder_. If the adversarial pairs were already at s=+2 under the old γ, they're now at s=+0.5 under the new γ — moderate gradient, but still not the "deep learning" regime (s < -1). You'd need γ high enough to push adversarial pairs into negative slack, which requires knowing their base margin first (→ run the histogram).

**Hard-pair mining** (train only on pairs with s < 0) is principled but collapses at Yonas's scale. If ~12 pairs qualify, you're training a rank-16 LoRA (18M parameters) on 12 examples. Below ~50 pairs, per-batch composition becomes unstable enough that gradient noise plausibly dominates signal — each batch is a different random subset of the hard pairs and the optimizer oscillates rather than converges.

**Expanding the adversarial held-out** is the only move that changes the _detectability_ problem. At n=44 with a baseline of 97.73% and a +2.27pp effect, the held-out is too small to detect a lift this size at any conventional significance threshold. The problem may not be that training failed — it may be that your measurement instrument is too small to see what training did. Source additional adversarial tasks from the same probe families (P007/P011/P027) using the multi-LLM synthesis pipeline, targeting n≥200 with ≥50 adversarial pairs.

## Sources & Pointers

1. **Meng, Xia, and Chen, "SimPO: Simple Preference Optimization with a Reference-Free Reward," NeurIPS 2024.** https://arxiv.org/abs/2405.14734 — The loss derivation and γ/β sensitivity analysis. The gradient decay `σ(-s)` follows directly from differentiating the SimPO loss. The 0.5-0.75 γ/β range Yonas uses is the paper's recommended region for instruction-tuned models.

2. **Rafailov et al., "Direct Preference Optimization," NeurIPS 2023.** https://arxiv.org/abs/2305.18290 — The reference-model term that SimPO removes. DPO's gradient includes a `π_ref(y)/π_θ(y)` importance weight that naturally decays updates on pairs where the policy has moved far from the reference. SimPO has no equivalent brake, which is why the margin-widening regime (after all pairs are correctly ranked) continues unchecked.

3. **Hands-on:** The diagnostic script above. Run on Colab T4, ~15 minutes for 618 pairs at max_length=512. The histogram + adversarial cross-reference is the single artifact that resolves whether training had signal on the cases Yonas built the bench for. Ship it as `ablations/diagnose_per_pair_margins.py` alongside any future preference-tuning run.
