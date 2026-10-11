# Give agents "now"

An LLM has no clock and no view of current state. Inject "now" — the date/time, and any volatile fact
(live price, balance, current tier) — via a tool the agent calls or a value stamped into the prompt/state.
Never let the model infer a time-dependent value; asked to compute from "now," it invents one.

Any eligibility / expiry / age / SLA calculation is unsafe if "now" is implicit: it passes mid-range demo
data and fails silently at the boundaries. Audit tell: if a date or number in the reasoning isn't traceable
to a tool result or the prompt, it was hallucinated — test boundary cases, not just the middle.
