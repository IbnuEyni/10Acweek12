# Sign-off — Day 2

**Asker:** Amir Ahmedin  
**Pair:** Ephrata Nebiyu  
**Topic:** Agent and tool-use internals — constrained decoding and function-calling mechanics  
**Grounding commit:** `10Acweek10/agent/llm_client.py` — added `response_format` for structured output (full diff in [`grounding_commit.md`](grounding_commit.md))

## Status: CLOSED

Ephrata's explainer closed the gap completely.

## What I understand now that I did not before

I went into Day 2 asking what the model does differently at the token level during function-calling vs when I prompt it to emit JSON. I was thinking in binary: either it's constrained decoding (guaranteed valid) or it's prompt injection (hope for the best). Ephrata showed me there are **four distinct levels**, and my engineering decision depends on which level my provider operates at.

First, the four-level taxonomy:
1. **Plain prompted JSON** (my current approach) — full vocabulary at every step. Model can emit fences, think blocks, wrong keys. My fallback chain is correct and necessary.
2. **JSON mode** (`response_format: json_object`) — guarantees valid JSON *syntax* but not my specific schema. Wrong keys, wrong value types still possible.
3. **Tool calling without strict** — provider adds protocol structure + model is fine-tuned on tool-call patterns. Learned behavior, not proof-by-construction.
4. **Tool calling with strict** (`strict: true`) — constrained decoding at the token level. Illegal schema-breaking tokens are masked before sampling. My fallback chain is dead code.

Second, the key insight: "The fact that you need to strip fences, remove think blocks, and salvage substrings is itself the clue — those repairs only make sense if the model had permission to emit tokens outside the intended object." My repair code is *evidence* of the decoding mode I'm operating in, not just defensive engineering. It's diagnostic.

Third, the one-sentence version: **"A prompt can encourage a model to output the right key. A decoder constraint can forbid it from outputting the wrong key. For workflow safety, forbidding beats encouraging."**

Fourth, the OpenRouter relay question: since OpenRouter may route to any provider (Fireworks, Together, Lepton), each operating at a different level, I should treat deletion of the fallback chain as a **testable hypothesis** — run the eval set on both paths (with and without `response_format`), compare parse failures, and only remove the fallback if the strict path produces zero failures on my specific deployment route.

## What changed in my Week 11 portfolio

- Added `response_format` parameter to the `complete()` call inside `complete_json()` in `agent/llm_client.py`. When the provider supports strict mode, invalid JSON is structurally impossible. The fallback chain remains for providers at levels 1-3.

- Annotated Probe 6.2 in `probes/failure_taxonomy.md` (JSON parse failures) as **provider-dependent**: architecturally impossible on strict-mode providers (level 4), possible on fine-tuning-only providers (levels 1-3). The probe remains in the taxonomy but with a note explaining when it's eliminated by construction vs when it requires the fallback chain.

- Added a note to `eval/method.md` mechanism design section: "The policy-aware prompting approach operates at Level 1 (plain prompted JSON). A third ablation variant — structured output via tool-calling (Level 4) — would eliminate Probe 6.2 entirely but requires provider support verification on the specific OpenRouter route used in production."

- Marked Oracle Forge `agent/kb_injector.py` for review: if the DAB scaffold uses function-calling for `query_db` and `execute_python`, understanding how tool descriptions influence selection would let me write better tool descriptions rather than stuffing context into the system prompt.
