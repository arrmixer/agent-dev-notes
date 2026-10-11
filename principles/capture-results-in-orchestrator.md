# Capture results in the orchestrator, don't trust the LLM to carry them

> **Rule:** Persist each step's output in orchestrator-owned state (a shared store), and have downstream
> steps read from that state — not from the model re-stating prior results in its context. Correctness
> should never depend on an LLM faithfully echoing structured data forward.

*(Terse: `.claude/rules/capture-results-in-orchestrator.md`.)*

## What it means

In a multi-step agent flow it's tempting to let each step "pass along" what the previous one found by
restating it. LLMs drop fields, round numbers, rename things, and paraphrase. The reliable design: after a
worker runs, the **orchestrator writes its result to a store it owns**; the next worker reads the fact from
the store, not from a chat message.

## Evidence

- **Capstone:** every route goes through `_run_and_record`, which writes the worker's output to its own
  column in a DynamoDB `WorkflowState` row (with optimistic locking). The Refund Agent reads the order facts
  the Inventory Agent gathered **from state**, not by re-deriving them — so the two literally cannot disagree
  about the order date. The Communication Agent renders the whole accumulated state at the end.
- Contrast (the failure this prevents): if the orchestrator had asked each agent to "include the previous
  findings in your answer," a dropped or mutated field would silently corrupt the final reply.

## Why it's Tier-1

The store is the **single source of truth**; the LLM's context is lossy transport. This holds for any
multi-step agent system regardless of provider or storage (DynamoDB, Redis, a dict, a DB row). It's the
structural fix behind the ordered-dependency pattern (step B grounds its decision in step A's recorded facts)
and a prerequisite for [give-agents-"now"](give-agents-now.md) — facts come from code, not model memory.

## How to apply

1. Define an orchestrator-owned state object keyed per session/request.
2. After each worker runs, **write its result** to a dedicated field (with concurrency control if parallel).
3. Downstream workers **read inputs from state**, not from prior model turns.
4. The final renderer reads the full state; it doesn't reconstruct earlier results from memory.
5. If a value is load-bearing, it must exist in state — if it only lives in a model's prose, it's not trustworthy.

## Source
`agentcore-reference/multi-agent-patterns.md` (§"don't make correctness depend on the LLM echoing data"),
capstone `WorkflowState` + `_run_and_record`.
