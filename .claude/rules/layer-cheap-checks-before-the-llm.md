# Route with a cascade — cheap checks before the LLM

Make routing/classification a cascade, cheapest and most-deterministic first: priority keyword → rule
keyword → LLM classifier → low-confidence fallback. The model only handles what the cheap layers can't.
Order the keyword layers by business priority (highest-stakes intent first).

A keyword rule is an allowlist — it silently misses paraphrases — so the LLM is the recall backstop for
vocabulary gaps, not just an ambiguity handler; budget it for rule-misses + ambiguity. Don't add a
pre-filter that rarely hits (just latency); measure the hit rate first. A green run does NOT verify the
fallback branch — test a sub-threshold input.
