# Give agents "now" — never let them infer time-dependent facts

> **Rule:** An LLM has no clock and no view of current state. Inject "now" (the date/time, and any volatile
> fact like current price or account state) deterministically via a tool or the prompt/state. Never let the
> model *infer* a time-dependent value — it will confidently hallucinate one.

*(Terse: `.claude/rules/give-agents-now.md`.)*

## What it means

Anything the model needs that depends on the present moment — "how many days since X," "is this expired,"
"is the account past due" — must be *supplied*, not guessed. The model has no reliable notion of today's
date; asked to compute from "now," it invents a "now."

## Evidence

- **Capstone:** the Refund Agent reasoned *"today is 2026-11-12 … 45 days elapsed"* to approve a 60-day
  return. The real date was 2026-10-10 (12 days). It **hallucinated the current date.** The approval was
  correct only because the order sat well inside the window — **a boundary case (day 58 vs 62) would flip
  the decision wrong, with confident arithmetic to match.** The scariest failures are the silent, edge-only ones.

## Why it's Tier-1

This generalizes to *any* eligibility / expiry / age / SLA / rate calculation, and beyond dates to any
environment fact the model can't observe (live prices, inventory, user tier at this instant). It's a special
case of [capture-results-in-orchestrator](capture-results-in-orchestrator.md): load-bearing facts come from
code, not model memory. And it passes demos (mid-window data) while failing in production (edges) — the
worst detection profile.

## How to apply

1. Provide a `get_current_date()`/`get_current_time()` tool the agent must call, **or** stamp the value into
   the routed request / shared state so the worker reads a fact.
2. Do the same for any other volatile fact (price, balance, tier) — fetch it, don't let the model assume it.
3. Audit the reasoning trace: **if a date or number isn't traceable to a tool result or the prompt, it was
   hallucinated.** Test boundary cases explicitly, not just the comfortable middle.

## Source
Capstone live run (Refund Agent date hallucination); `reference/multi-agent-patterns.md`.
