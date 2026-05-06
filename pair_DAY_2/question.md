# Day 2 Question — Agent and Tool-Use Internals

**Asker:** Amir Ahmedin  
**Topic:** Agent and tool-use internals  
**Date:** Day 2, Week 12  
**Pair:** Ephrata Nebiyu

## The Question (as submitted for voting)

In my Week 10 Conversion Engine (`agent/llm_client.py`, `complete_json()` method), I prompt the model to output raw JSON with structured fields like `reply_class`, `should_book_call`, and `needs_human_handoff`. I then strip markdown fences, remove `<think>` blocks, and attempt `json.loads()` — falling back to substring extraction between the first `{` and last `}` when parsing fails. This is essentially a poor man's function call.

**What is the model doing differently at the token level when it "chooses" a tool via the function-calling API versus when I prompt it to emit JSON? Specifically: does function-calling constrain the decoding vocabulary at each token step (like constrained decoding / grammar-guided generation), or is it purely a prompt-engineering trick where the tool schema is injected into the system prompt and the model generates free-form tokens that happen to match?**

## Sharpening Responses (from Ephrata's interrogation)

**Q: Do you want to focus on the next-token decoding mechanism, or on the whole API stack around the model?**

Both, but the decoding mechanism is the core. I want to know what happens at the logit level during generation — whether tokens are masked or not. The API stack (how OpenRouter relays constraints to upstream providers) is the adjacent context that makes the answer actionable for my specific deployment.

**Q: Is your real confusion about tool choice (how the model decides "call a tool now") or about structured argument generation once the tool has already been selected?**

Both are gaps, but they're different mechanisms. My `complete_json()` problem is about **structured argument generation** — I need valid JSON back, and I want to know if the API guarantees that structurally. The tool *choice* question is secondary and connects to Oracle Forge (Week 8-9) where the agent picks the wrong tool — but that's adjacent, not core.

**Q: Are you comparing your current setup to plain function calling, or specifically to function calling with strict schema enforcement / constrained decoding?**

Specifically to **strict schema enforcement / constrained decoding**. I know plain function-calling exists — what I cannot explain is whether it applies token-level constraints (logit masking via FSM/grammar) or whether it's just fine-tuning that makes the model *likely* to produce valid JSON without *guaranteeing* it. The distinction determines whether my fallback parsing chain is dead code or correct engineering.

**Q: Is the gap mainly about reliability (why function calling breaks less often) or about mechanism (what changes mathematically during generation)?**

**Mechanism first, reliability as consequence.** I want to understand what changes procedurally/mathematically during generation — because once I understand the mechanism, the reliability difference becomes obvious. If it's constrained decoding: reliability is 100% by construction. If it's fine-tuning: reliability is high but not guaranteed, and my fallback chain is correct.

## Connection to Existing Work

Knowing this would let me revise:

- **`agent/llm_client.py`** (Week 10, lines 82–100) — The `complete_json()` method uses regex stripping and substring extraction as fallbacks for malformed JSON. If native function-calling applies constrained decoding, I should refactor to use the `tools` parameter, eliminating the entire fallback chain and the failure mode it handles.

- **`probes/failure_taxonomy.md`** (Week 10, Probe 6.2) — Documents JSON parse failures as a failure category with a measured trigger rate. If constrained decoding guarantees valid JSON, this entire probe category becomes architecturally impossible — I can remove it from the taxonomy and replace it with "tool-selection accuracy" as the relevant failure mode.

- **`eval/method.md`** (Week 10) — The mechanism design section describes the policy-aware prompting approach. If tool-calling provides structural guarantees that prompting cannot, the mechanism comparison should include a "structured output via tool-calling" condition as a third ablation variant.

- **Oracle Forge `agent/kb_injector.py`** (Week 8-9) — The KB injection monkey-patches the DataAgent's `__init__` to inject context into `db_description`. If the DAB scaffold uses function-calling for `query_db` and `execute_python`, understanding how tool descriptions influence selection would let me write better tool descriptions rather than stuffing everything into the system prompt.

## Why This Matters Beyond My Work

Every FDE building production agents faces the "structured output" problem: how do you guarantee the model returns valid, schema-compliant responses? The three approaches — (1) prompt-and-pray with fallback parsing, (2) native function-calling API, (3) constrained decoding libraries like Outlines or guidance — have fundamentally different failure modes and cost profiles. But most practitioners cannot explain *why* they differ at the token level. The answer determines whether JSON parse failures are an engineering problem (fixable by switching APIs) or a fundamental model limitation (requiring retry logic regardless).
