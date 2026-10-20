# Saga: three invariants, not just "run compensations"

A saga runs steps forward and undoes already-done ones on failure. Correctness is three invariants:
(1) compensate in REVERSE order; (2) only for steps with status == completed; (3) gate resolution on a
barrier that's an invariant `==` (completed == needed), not a threshold `>=` — a surprising count leaves it
`compensating` for an operator. Use an atomic counter (e.g. DynamoDB ADD) so parallel compensators don't
lose increments.

Add a `needed == 0` branch (first-step failure resolves `failed` immediately), and use end-to-end
idempotency tokens — a local status flag can't survive a crash between the downstream call and the state
update. The naive version (compensate all, forward order, threshold) silently refunds failed charges and
cancels nonexistent shipments.
