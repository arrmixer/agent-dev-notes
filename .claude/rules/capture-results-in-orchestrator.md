# Capture results in the orchestrator

In a multi-step agent flow, persist each step's output to orchestrator-owned state and have downstream
steps read inputs from that state — not from the model re-stating prior results in its context. LLMs drop,
round, and paraphrase data they're asked to carry forward.

The store is the source of truth; the model's context is lossy transport. If a value is load-bearing it
must exist in state — if it only lives in a model's prose, it isn't trustworthy. Add concurrency control
(e.g. optimistic locking) when steps write in parallel.
