# The orchestrator is code, not a chatty agent

> **Rule:** Keep coordination (routing, sequencing, control flow) deterministic — explicit rules, a cheap
> model at temp 0, one uniform path per route. Use the LLM for the leaf work, not for deciding the whole
> flow. The orchestrator never does a worker's job or writes the final output itself.

*(Terse: `.claude/rules/orchestrator-is-code.md`.)*

## What it means

A multi-agent "orchestrator" is tempting to build as one clever LLM that figures out what to do. Don't. The
conductor is **control-flow code**; the LLM's judgment belongs in the leaves (the specialist workers). The
orchestrator's own decisions (which worker, in what order, when to stop) should be as deterministic and
testable as a function.

## Evidence

- **Capstone:** the orchestrator is Haiku at **temp 0** with six explicit routing rules and one shared
  `_run_and_record(read → run worker → write)` path for all four routes. It behaved correctly on every
  scenario — ordered inventory→refund, parallel policy, math-without-workers, communication-agent-always-last.
  The reliability came from **the rules + the uniform code path, not model cleverness.**
- **L4 (orchestration):** "the conductor is code, not a prompt." Control flow reduces to three primitives
  (sequence, branch, parallel); extensibility comes from a *uniform agent interface*, not a smarter prompt.

## Why it's Tier-1

Deterministic control flow is **reproducible, cheap, debuggable, and testable**. An LLM freely deciding the
entire flow is none of those — same input, different path; higher cost; and you can't unit-test "did it
route correctly." Push non-determinism to the leaves where judgment is actually needed and can be bounded.

## How to apply

1. Write the routing rules **explicitly** (a table), and encode them as the orchestrator's tools + prompt.
2. Router model: cheapest capable, **temp 0** (deterministic).
3. One uniform path per route (read state → run worker → persist) — not bespoke logic per branch.
4. The orchestrator **delegates leaf work and final output**; it never answers the user directly.
5. Make the control flow assertable (which tools fired, in what order) — that's your real test, beyond "it ran."

## Source
`agentcore-reference/multi-agent-patterns.md` (§orchestration), capstone `build_orchestrator_agent` + `_run_and_record`.
