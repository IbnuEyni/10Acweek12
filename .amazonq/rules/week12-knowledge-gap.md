# Week 12 — Knowledge Gap Formulation for Compounding
# Project Rules for AI-Assisted Development

## Project Context

This repository contains paired daily research deliverables for TRP1 Week 12.
Each day produces: a sharpened question, an explainer (600-1000 words), a tweet
thread (4-6 posts), call summaries, a signoff, a grounding commit, and sources.
The work is grounded in Weeks 10-11 portfolio artifacts (10Acweek10/ and
tenacious-sales-bench/) and must produce concrete edits to those artifacts.

## Repository Structure

```
10Acweek12/
├── pair_DAY_N/           # One folder per day (N = 1-4)
│   ├── question.md       # Sharpened question with artifact connections
│   ├── morning_call_summary.md
│   ├── explainer.md      # 600-1000 word blog post answering partner's question
│   ├── thread.md         # 4-6 post LinkedIn/X thread
│   ├── evening_call_summary.md
│   ├── signoff.md        # Gap closure judgment + portfolio edits
│   ├── grounding_commit.md  # Before/after on specific Week 10/11 artifacts
│   └── sources.md        # 2 canonical papers + hands-on tool/pattern
└── TRP1 Challenge Week 12_.md  # Challenge specification
```

## Writing Standards

### Questions (question.md)
- Must name a specific gap in the asker's own Week 10/11 work
- Must be grounded: "Knowing this would let me revise [specific file]"
- Must include sharpening responses from the partner's interrogation
- Must pass four properties: diagnostic, grounded, generalizable, resolvable
- Bold the core question. Provide "a satisfying answer would tell me" criteria.

### Explainers (explainer.md)
- Open with the partner's question in your own words
- Name the load-bearing mechanism in plain language
- Show it: code, diagram, or concrete worked example with real numbers
- Connect 2-3 adjacent concepts that make the gap worth closing
- Cite 2 canonical sources (original papers, not summaries)
- Include at least one hands-on demonstration (runnable code or verified output)
- Label illustrative examples explicitly — never present constructed values as empirical
- All numbers must trace to actual data in the partner's repo or be marked as estimates

### Signoffs (signoff.md)
- Status: closed / partially closed / not closed
- "What I understand now that I did not before" — full narrative arc
- "What changed in my Week 11 portfolio" — specific file paths and edits
- Match the depth of the partner's signoff (reciprocal quality)

### Grounding Commits (grounding_commit.md)
- Verification instructions (how to confirm the edit)
- Before/after on specific code or text (not paraphrased — actual content)
- "What this changes about my claim" — what was wrong, what's fixed
- "What I learned that I did not know" — deeper insight beyond the fix

### Threads (thread.md)
- 5 posts, each standalone (reader who doesn't click through still gets value)
- Post 1: hook with the question/paradox
- Posts 2-3: core mechanism with code or concrete numbers
- Post 4: adjacent insight or engineering recommendation
- Post 5: sources + link to full explainer

## Technical Standards

### Code in Explainers
- Must be runnable or clearly labeled as pseudocode
- Use the partner's actual file paths, variable names, and data structures
- Adapt placeholder field names to the partner's real JSON schema when possible
- Include runtime estimates for diagnostic scripts

### Statistical Claims
- Report confidence intervals alongside point estimates
- Name the test used and why it's appropriate
- When reporting agreement metrics, include both raw agreement AND chance-corrected (kappa)
- When reporting accuracy, state the denominator explicitly (n=X tasks)
- Never report a conditional estimate (accuracy-on-answered) without noting the coverage rate

### Citations
- Original papers when they exist (not blog posts or summaries)
- Include arXiv/DOI links
- Do not claim specific section/equation numbers unless verified against the paper
- Name the key insight used from each source (not just "this paper exists")

## Grounding Rules

### Every claim must trace to evidence
- Numeric claims → specific JSON field in a repo file
- Mechanism claims → derivation or cited paper
- Engineering recommendations → runnable diagnostic or before/after demonstration
- "I verified this" → name the command or script that produced the verification

### Partner's code takes precedence over assumptions
- When writing an explainer, read the partner's actual code before speculating
- If you cannot access the partner's repo, state assumptions explicitly
- Correct factual errors in revision (evening call), don't let them ship

### Feedback must be specific and actionable
- "What landed" — name the specific paragraph or formula
- "What didn't land" — name what's missing or wrong, not just "could be better"
- Revisions must address every point in "what didn't land"

## Quality Gates

Before submitting any day's deliverables:
1. All numbers in the explainer are verified against actual repo data
2. Code snippets are syntactically correct and use real field names
3. The signoff names specific file paths that changed
4. The grounding commit has before/after (not just a description of the change)
5. The thread's posts each stand alone without the blog
6. Sources include DOI/arXiv links and name the key insight used

## Week 10/11 Artifact References

### Week 10 (10Acweek10/)
- `agent/llm_client.py` — LLM client with complete() and complete_json()
- `eval/method.md` — Policy-aware prompting mechanism design
- `eval/harness.py` — τ²-Bench evaluation harness
- `memo.md` — CFO decision memo ($0.15/task, $0.07/qualified lead)
- `probes/failure_taxonomy.md` — 33 probes across 7 failure categories

### Week 11 (tenacious-sales-bench/)
- `training/train_simpo.py` — SimPO training script (Qwen2.5-1.5B, LoRA r=16)
- `training/prepare_training_data.py` — Preference pair generator (16 templates → 111 unique chosen)
- `evaluation/scoring_evaluator.py` — Hybrid scorer (regex + LLM judge)
- `ablations/ablation_results.json` — Delta A/B/C results
- `model_card.md` — Judge model card
- `methodology.md` — Dataset construction and training path
- `inter_rater_agreement.md` — Intra-rater test-retest reliability (κ=0.461)

## Topics Covered

| Day | Topic | Key Insight |
|-----|-------|-------------|
| 1 | Inference-time mechanics | Think tokens are billed + occupy KV cache; suppress don't strip |
| 2 | Agent and tool-use internals | Four levels of structured output; repair code = evidence of decoding mode |
| 3 | Training and post-training | Chosen-side diversity (ratio 0.19) is the binding constraint, not algorithm choice |
| 4 | Evaluation and statistics | Raw agreement at high base rates is misleading; report kappa + coverage × accuracy |
