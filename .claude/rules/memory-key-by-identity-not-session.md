# Key memory by identity, not session

Cache/retrieve agent memory and per-user context keyed on identity (`actor_id`), not session alone — a
session-only key reuses the previous actor's data, a silent cross-user leak. Never default an identity field
(`actor_id="default_user"` is fail-open); read it from the trusted channel.

Namespace shape is a day-one, permanent decision (records are never re-pathed) — order the hierarchy by your
dominant query. Make prefixes leak-safe with leading AND trailing `/` (else `/alice/` matches `/alice2/`).
Filter retrievals by relevance score client-side, keep topK small for direct injection, and cite
namespace+record id to catch hallucinated memories. Don't feed assembled/sensitive turns back into extraction.
