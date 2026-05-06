# Sources — Day 2

**Topic:** Agent and tool-use internals — centralized orchestrators as tool-use policy layers
**Explainer:** Amir Ahmedin (answering Ephrata's question)
**Date:** Day 2, Week 12

## Canonical Sources

### Source 1: Enterprise Integration Patterns — Process Manager
**Hohpe, G. & Woolf, B.** *Enterprise Integration Patterns: Designing, Building, and Deploying Messaging Solutions.* Addison-Wesley, 2003. Chapter: Process Manager.
https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html

Defines the distinction between **orchestration** (centralized Process Manager that controls the sequence) and **choreography** (distributed participants that react to events independently). The Process Manager pattern maintains the authoritative state of a multi-step process and decides which participant to invoke next based on that state. The book explicitly identifies the failure mode of choreography: no single participant has enough context to make globally correct decisions under partial failures. This is the theoretical foundation for why Ephrata's centralized orchestrator is architecturally safer than distributed handlers.

**Key insight used:** The Process Manager's core responsibility is not "calling things in order" — it's maintaining the invariant that only valid transitions occur given the current global state. This reframes the orchestrator from "workflow coordinator" to "policy enforcement layer."

### Source 2: Sagas Paper
**Garcia-Molina, H. & Salem, K.** "Sagas." ACM SIGMOD Record, 1987.
https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf

Introduces the Saga pattern for long-lived transactions that span multiple services. A saga is a sequence of local transactions where each has a compensating action. The paper compares two coordination approaches: (1) a central coordinator that knows which steps completed and triggers compensation on failure, and (2) choreography where each participant must know the full protocol. The paper shows that choreography requires every participant to handle every possible failure combination — which becomes combinatorially intractable as the number of steps grows.

**Key insight used:** The saga coordinator's value is not just "knowing the order" — it's being the single place that can detect partial completion and trigger compensation. In Ephrata's system, if SMS sends but calendar booking fails, only the orchestrator knows both facts and can decide whether to retry, compensate, or escalate. A distributed SMS handler cannot know that the calendar step failed.

## Tool / Pattern Used

### Hands-on: Comparison with my own Week 10 orchestrator

Examined my own `agent/main.py` (Week 10 Conversion Engine) as a concrete implementation of the same pattern:

```python
# agent/main.py — orchestrator enforces state before invoking handlers:

@app.post("/prospects/{prospect_id}/outreach")
def send_outreach(prospect_id: str):
    prospect = prospects.get(prospect_id)
    # INVARIANT: monotonic transition — only ENRICHED or NEW can send outreach
    if prospect.state not in (ConversationState.ENRICHED, ConversationState.NEW):
        raise HTTPException(400, f"Prospect in state {prospect.state.value}, cannot send outreach")
    # Only after validation does the handler (composer) get invoked
    email = composer.compose(prospect)
    ...
```

This is the same pattern Ephrata uses: the orchestrator (`main.py`) checks global state before invoking any tool (email composer, SMS handler, booking engine). The handlers themselves never check whether they're "allowed" to run — that's the orchestrator's job. If I removed this check and let `email_composer.py` decide locally whether to compose, it would need access to the full prospect state — coupling it to concerns outside its domain.

The comparison confirmed that the pattern generalizes: my sales agent, Ephrata's SignalForge, and any multi-tool agent system all benefit from the same centralized policy layer that restricts which tools can fire based on global state.
