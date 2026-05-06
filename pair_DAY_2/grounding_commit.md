# Grounding Commit — Day 2

## Amir's grounding commit (based on Ephrata's explainer)

**Asker:** Amir Ahmedin
**Artifact edited:** `10Acweek10/agent/llm_client.py` — `complete_json()` method

### What changed

Added `response_format` parameter to the `complete()` call inside `complete_json()`, requesting strict structured output when available. The fallback chain is preserved for providers that don't support constrained decoding.

**Before:** `complete_json()` always generates unconstrained text and applies post-hoc parsing with three fallback layers (fence stripping, think-block removal, substring extraction).

**After:** `complete_json()` passes `response_format: {"type": "json_object"}` to the API call. When the provider supports strict mode, invalid JSON is structurally impossible at the token level. The fallback chain remains as defensive engineering for providers operating at levels 1–3 of Ephrata's taxonomy.

### Why this changed

Ephrata's four-level taxonomy made the engineering decision concrete:
- Level 1 (my current code): full vocabulary, fallback chain necessary
- Level 4 (strict structured outputs): constrained decoding, fallback chain is dead code

Since I use OpenRouter (which may route to any level), the correct approach is: request the highest level available (`response_format`) but keep the fallback chain for when the upstream provider doesn't support it. This is not "belt and suspenders" — it's adapting to a heterogeneous provider landscape.

Additionally, Probe 6.2 in `probes/failure_taxonomy.md` (JSON parse failures) is now annotated as **provider-dependent**: impossible on strict-mode providers, possible on fine-tuning-only providers. The probe remains in the taxonomy but with a note explaining when it's architecturally eliminated.

### The edit

```python
def complete_json(self, messages: list[dict], schema: dict | None = None, **kwargs) -> dict:
    """Complete and parse response as JSON. Uses constrained decoding when available."""
    # Request structured output — provider will use constrained decoding if supported
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
    if raw.startswith("```"):
        lines = raw.split("\n")
        lines = [l for l in lines if not l.strip().startswith("```")]
        raw = "\n".join(lines).strip()

    if "<think>" in raw:
        raw = re.sub(r"<think>.*?</think>", "", raw, flags=re.DOTALL).strip()

    try:
        result["parsed"] = json.loads(raw)
    except json.JSONDecodeError:
        start = raw.find("{")
        end = raw.rfind("}")
        if start != -1 and end != -1:
            result["parsed"] = json.loads(raw[start:end + 1])
        else:
            raise ValueError(f"Could not parse JSON from LLM response: {raw[:200]}")

    return result
```

---

## Ephrata's grounding commit (based on Amir's explainer)

**Asker:** Ephrata Nebiyu
**Artifact edited:** `docs/system_architecture.md` ([/home/rata/Documents/Ephrata/work/10Acadamy/training/SignalForge/docs/system_architecture.md])

### What changed

Updated the architecture write-up to make the channel orchestrator's defensive role explicit instead of describing only the happy-path flow. The new section names the invariants Ephrata can now defend after reading Amir's explainer: authoritative shared state, monotonic transitions, duplicate suppression when `external_id` is available, centralized channel eligibility, and the separation between provider adapters and policy.

### Why this changed

This changed the document from "here is the sequence of channels" to "here is the mechanism that prevents invalid tool actions," which is the concrete portfolio improvement produced by closing this gap.
