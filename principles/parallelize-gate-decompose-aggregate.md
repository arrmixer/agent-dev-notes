# Parallelize through three layers: gate → decompose → aggregate

> **Rule:** Before running agent work in parallel, pass it through three layers. **Gate:** should you (are
> the subtasks independent, safe to run concurrently, worth the overhead)? **Decompose:** split along the
> independent dimension. **Aggregate:** merge the results (the merge strategy is largely *determined* by how
> you split). Parallelism is for independent, I/O-bound subtasks — not a default.

*(Terse: `.claude/rules/parallelize-gate-decompose-aggregate.md`.)*

## What it means

"Run it in parallel" is a decision with a safety precondition, not a free speedup. The gate comes first:
parallelizing *dependent* or side-effecting work corrupts results or races. Only once independence is
established do you decompose and then merge.

## Evidence

- **Capstone (Policy Agent):** three KB retrievers are genuinely independent (one KB each, read-only), so
  they run via `ThreadPoolExecutor(max_workers=3)` + `as_completed`, then the coordinator aggregates. The
  gate passed (independent, I/O-bound, 3× latency win), so parallelism was correct.
- **L3 (parallelization):** the numbers showed the win is real *when the gate holds*; the method is
  gate → decompose → aggregate, and the merge (dedup/rank/combine) follows from the decomposition.

## Why it's Tier-1

The failure mode of naive parallelism — racing on shared state, double side-effects, merging things that
weren't independent — is silent and data-dependent. The gate is the cheap check that prevents it. Pairs with
the concurrency-model choice (small-N blocking I/O → threads, not asyncio; see `language/` #24).

## How to apply

1. **Gate:** Are the subtasks independent (no shared mutable state, no ordering dependency)? Side-effect-free
   or idempotent? Is the concurrency worth the overhead at this N? If any "no," don't parallelize.
2. **Decompose:** split along the independent axis (per-KB, per-document, per-region).
3. **Aggregate:** choose the merge the split implies — dedup, rank, top-K, or concatenate-with-labels.
4. For small-N blocking I/O use a thread pool; reserve `asyncio` for large-N with async-native libraries.

## Source
`reference/multi-agent-patterns.md` (§parallelizing agent work), capstone `search_all_policies`.
