# A saga is three invariants, not just "run the compensations"

> **Rule:** A saga replaces a cross-service transaction with N local commits, each paired with a
> compensating undo. Correctness is three invariants: compensate in **reverse** order, only for
> **completed** steps, and gate resolution on a **barrier that is an invariant (`==`), not a threshold**.
> Plus end-to-end **idempotency tokens** — the local state machine alone isn't enough.

*(Terse: `.claude/rules/saga-compensation-correctness.md`.)*

## What it means

You can't hold one ACID transaction across services, so you run steps forward and, on failure, undo the
already-done ones. The orchestrator (code, not an LLM) owns this. The naive version — compensate all steps,
in forward order, resolve on a threshold — silently refunds charges that failed and cancels shipments that
never existed.

## Evidence (L7 e-commerce checkout: Inventory → Payment → Shipping)

| Checkout | Fails at | Compensated | Barrier |
|---|---|---|---|
| CHK-002 | payment | inventory only (shipping never ran) | 1/1 |
| CHK-003 | shipping | **payment → inventory** (reverse of forward) | 2/2 |

1. **Reverse order.** CHK-003 undid payment *before* inventory. Forward-order compensation is the bug — you'd
   undo dependencies before the things depending on them.
2. **Only completed steps compensate.** Filter `status == "completed"` first. CHK-002's shipping stayed
   `pending`, so it was never "cancelled."
3. **Barrier is an invariant, not a threshold.** Flip to `failed` only when `completed == needed` (`==`, not
   `>=`). A surprising count (short *or* a double-increment) leaves the saga `compensating` for an operator —
   for money-movement, refusing to self-resolve on odd bookkeeping is the safe default. Implement the counter
   with an atomic op (DynamoDB `ADD`), not read-modify-write, so parallel compensators don't lose increments.

## Known weaknesses (design for them)

- **`needed == 0` gap:** a *first-step* failure (nothing completed) can fall through the `needed > 0` guard
  and never resolve cleanly — add a third branch: `needed == 0` → resolve `failed` immediately.
- **Idempotency is the real gap:** compensations must be safe to run twice (cancel twice ≠ double refund). A
  local `status` flag is the weakest form — the orchestrator can crash between the downstream call and the
  state update. Production needs idempotency tokens the downstream service de-dups.

## Why it's Tier-1

Distributed-transaction correctness, provider-agnostic. Same family as `shared-state-not-correct-derived-state`
(the primitive is necessary but not sufficient) and `capture-results-in-orchestrator` (read the outcome from
state — a worker's stray `<report>` tag didn't matter because the orchestrator read `step["status"]`, not prose).

## Source
`reference/multi-agent-patterns.md` (§saga), L7 ecommerce-checkout.
