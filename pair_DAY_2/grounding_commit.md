# Grounding Commit — Day 2

**Asker:** Amir Ahmedin  
**Pair:** Ephrata Nebiyu  
**Topic:** Agent and tool-use internals  
**Repo:** `10Acweek10`  
**How to verify:** Open `agent/llm_client.py`, find `complete_json()`. The `response_format` parameter should be set before the `complete()` call. Run the agent on one task with a provider that supports JSON mode (Fireworks, Together) and confirm the response parses without triggering the fallback chain (no fence stripping, no think-block removal, no substring extraction).

---

## What changed in the Week 11 portfolio

Two edits in `10Acweek10/`, grounded in Ephrata's four-level taxonomy showing that my fallback chain is evidence of operating at Level 1 (unconstrained generation) when Level 2+ is available.

### 1. Added `response_format` to `complete_json()` in `agent/llm_client.py`

**Before:**

```python
def complete_json(self, messages: list[dict], **kwargs) -> dict:
    result = self.complete(messages, **kwargs)
    raw = result["content"].strip()
    # Three fallback layers: fence strip, think-block removal, substring extraction
    ...
```

The model generates unconstrained text. Any token is legal at any step. The fallback chain catches the resulting format violations (fences, think blocks, malformed JSON). This is Level 1 — "prompt and pray with repair."

**After:**

```python
def complete_json(self, messages: list[dict], schema: dict | None = None, **kwargs) -> dict:
    # Request structured output — provider uses constrained decoding if supported
    if schema:
        kwargs["response_format"] = {
            "type": "json_schema",
            "json_schema": {"name": "structured_output", "schema": schema}
        }
    else:
        kwargs.setdefault("response_format", {"type": "json_object"})

    result = self.complete(messages, **kwargs)
    raw = result["content"].strip()
    # Fallback chain preserved — handles providers at levels 1-3
    ...
```

Now requests the highest available level. On providers that support strict mode (Level 4), invalid JSON is structurally impossible — the fallback chain never triggers. On providers at Levels 1-3, the fallback chain still catches format violations. This adapts to a heterogeneous provider landscape without assuming any single level.

### 2. Annotated Probe 6.2 in `probes/failure_taxonomy.md`

**Before:**

> Probe 6.2: JSON parse failures. Trigger rate: 20% on baseline.

**After:**

> Probe 6.2: JSON parse failures. Trigger rate: 20% on baseline. **Provider-dependent:** architecturally impossible on strict-mode providers (Level 4 — constrained decoding masks illegal tokens). Possible on fine-tuning-only providers (Levels 1-3). The fallback chain in `complete_json()` handles Levels 1-3; at Level 4, the chain is dead code. Verify which level your specific OpenRouter route operates at before removing the fallback.

---

## What this changes about my Week 10 claim

Before this commit, my mechanism design (`eval/method.md`) described policy-aware prompting as the intervention — adding ~80 tokens to the system prompt. But the structured output problem (Probe 6.2: JSON parse failures at 20%) was treated as a separate failure mode requiring the fallback chain. Ephrata's taxonomy showed me these are the same problem viewed from different levels: at Level 1, you need both the prompt intervention AND the fallback chain. At Level 4, the prompt intervention still helps (better task completion) but the fallback chain is dead code (JSON validity is guaranteed by construction).

The engineering decision is now explicit: request Level 4 when available, keep the fallback for when it's not, and treat the fallback chain's trigger rate as a diagnostic for which level the provider is actually operating at. If the fallback never triggers across 100 tasks, the provider supports strict mode and the chain can be removed. If it triggers on 5% of tasks, the provider is at Level 2-3 and the chain is earning its keep.

## What I learned that I did not know before this commit

I was treating my repair code (fence stripping, think-block removal, substring extraction) as "defensive engineering" — good practice regardless of the provider. Ephrata reframed it: the repair code is *evidence* of the decoding mode. Its existence tells you the model had permission to emit tokens outside the intended object. At Level 4 (constrained decoding), those tokens are masked before sampling — they literally cannot appear in the output. So the repair code isn't just unnecessary at Level 4; it's *impossible to trigger*. That's a stronger statement than "unnecessary" — it means the code path is dead by construction, not just unlikely to fire.

The one-sentence version: "A prompt can encourage a model to output the right key. A decoder constraint can forbid it from outputting the wrong key. For workflow safety, forbidding beats encouraging."
