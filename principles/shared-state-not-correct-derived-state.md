# Locking protects writes, not the correctness of derived state

> **Rule:** Concurrency control (optimistic locking, conditional writes) guarantees no write is *lost* — it
> does **not** guarantee your *derived/aggregate* state is correct. Verify the business invariant your
> derived values depend on as a separate check, especially after recovery or compensation.

*(Terse: `.claude/rules/shared-state-not-correct-derived-state.md`.)*

## What it means

"No lost writes" and "the state is right" are different claims. A lock serializes concurrent updates so none
clobbers another; it says nothing about whether the *values you compute from* those updates (totals, counts,
completeness) are correct. Teams routinely conflate the two and ship wrong-but-consistent state.

## Evidence

- **L6 (shared state):** optimistic locking (version + `ConditionExpression`) correctly prevented lost
  writes under concurrency — but **derived order state could still be wrong**; locking the write path doesn't
  validate the aggregate. Logged as "locking ≠ correct derived state."
- **L7 (saga):** the barrier counter is an **invariant check (`==`), not a threshold (`>=`)** — the
  correctness came from asserting the invariant, not from the atomic increment alone. Compensation must be
  verified complete, not just "the undo writes landed."

## Why it's Tier-1

Concurrency control is necessary but not sufficient — the same shape as every other entry here (the headline
mechanism doesn't certify the outcome). It applies to any stateful multi-agent/distributed system regardless
of store. The dangerous version is subtle: everything looks consistent, so no error fires.

## How to apply

1. Use optimistic locking / conditional writes for **write safety** (prevent lost updates).
2. **Separately** assert the business invariant: do the per-step counts/totals reconcile? Is a saga's
   compensation complete (barrier `==` expected), not merely attempted?
3. After **recovery or compensation**, re-derive and check the aggregate — don't assume landed writes imply a
   correct end state.
4. Treat "writes succeeded" and "state is correct" as two tests, never one.

## Source
`reference/multi-agent-patterns.md` (§shared state, §saga), L6 food-delivery + L7 ecommerce-checkout.
