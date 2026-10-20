# Route with a cascade — cheap deterministic checks before the LLM

> **Rule:** Make routing/classification a cascade, cheapest and most-deterministic layer first; the model
> only handles what the cheap layers can't decide. Order: priority keyword → rule keyword → LLM classifier
> → low-confidence fallback. The LLM is the **recall backstop for vocabulary gaps**, not just an ambiguity
> handler — but add a pre-filter only where the traffic actually hits it.

*(Terse: `.claude/rules/layer-cheap-checks-before-the-llm.md`.)*

## What it means

"Code vs LLM" is the wrong framing — it's code *then* LLM. Deterministic layers handle the bulk fast and
free; the model earns its cost on the remainder. The layers, in priority order:
1. **Priority route** — keyword scan for the highest-stakes intent (so "cancel and refund" → Retention, not
   Billing). Order encodes business priority.
2. **Rule route** — keyword scan for the common intents.
3. **LLM classifier** — only for what 1–2 didn't match.
4. **Fallback** — LLM result below a confidence threshold → human review.

## Evidence

- **L5 telecom router, 20 tickets, 100% correctly routed:** priority 10% / rule 65% / LLM 25% / fallback 0%.
  **75% never touched the classifier.** The LLM path (~3.2s) ran ~1.7× a rule path (~1.9s), and only a
  quarter paid it — that ratio *is* the argument for cheap gates in front of a model.
- **The LLM is a recall backstop, not just an ambiguity handler:** 4 technical tickets used words the
  keyword rule didn't enumerate, fell through, and the classifier labeled all 4 at 0.90–0.95. A keyword rule
  is an **allowlist** — it catches exactly the vocabulary you wrote and silently misses every paraphrase; the
  cascade degrades gracefully (a rule miss is a fall-through, not a misroute).
- **The fallback never fired** → that branch stayed *unverified*. A green run doesn't prove the
  low-confidence path works; test an actually-below-threshold input.

## Why it's Tier-1

Cost, latency, and graceful degradation, for any routing/classification/intent system. Pure-LLM routing
makes every request pay the classification round-trip; a deterministic pre-filter removes that for the
unambiguous majority. Pairs with `orchestrator-is-code` (deterministic control flow) and
`raw-tool-until-toil-scales` (measure before adding the layer).

## How to apply

1. Order layers by business priority; put the highest-stakes keyword scan first.
2. **Size the LLM budget for rule-misses + genuine ambiguity**, not ambiguity alone.
3. Don't add a pre-filter that rarely hits — it's just latency and code. Measure the rule-hit rate first.
4. A green run doesn't verify the fallback; exercise it with a sub-threshold case.

## Source
`reference/multi-agent-patterns.md` (§routing), L5 telecom-router.
