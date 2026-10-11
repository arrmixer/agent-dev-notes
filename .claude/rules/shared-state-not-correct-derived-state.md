# Locking ≠ correct derived state

Concurrency control (optimistic locking, conditional writes) guarantees no write is lost. It does NOT
guarantee your derived/aggregate state (totals, counts, completeness) is correct. "No lost writes" and "the
state is right" are different claims — don't conflate them.

Use locking for write safety; separately assert the business invariant (reconcile counts/totals; a saga's
barrier is an invariant `==`, not a threshold `>=`; compensation must be verified complete, not just
attempted). After recovery or compensation, re-derive and check the aggregate. The dangerous failure is
subtle: everything looks consistent, so no error fires.
