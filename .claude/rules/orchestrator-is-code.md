# Orchestrator is code, not a chatty agent

Keep coordination — routing, sequencing, when-to-stop — deterministic: explicit rules, cheapest capable
model at temp 0, one uniform path per route (read state → run worker → persist). Use the LLM for leaf
work, not for deciding the whole flow.

The orchestrator never does a worker's job and never writes the final user-facing output itself — it
delegates. Make control flow assertable (which tools fired, in what order); "it ran" is not a test.
