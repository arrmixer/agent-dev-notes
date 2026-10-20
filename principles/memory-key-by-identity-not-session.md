# Key agent memory by identity, not session — and never default an identity field

> **Rule:** Cache/retrieve agent memory and per-user context keyed on **identity** (`actor_id`), not on
> session alone. Session-only keys reuse the *previous* actor's data → a cross-user leak. Never default an
> identity field; read it from the trusted channel. Decide namespace shape day-one (records are never
> re-pathed), and make prefixes leak-safe.

*(Terse: `.claude/rules/memory-key-by-identity-not-session.md`.)*

## What it means

Memory/personalization keys are a correctness **and security** decision. Get the key wrong and one user sees
another's data — silently, because nothing errors. These are day-one choices that are expensive to change later.

## The specifics (each a sub-rule)

- **Cache per-identity, not per-session.** Key on `(session_id, actor_id)`. A session-only key reuses the
  prior actor's config/memory — AWS's own sample has exactly this cross-user-leak bug.
- **Never default an identity field.** `actor_id = "default_user"` is the same smell as a guardrail fail-open
  — read identity from the trusted channel, never a fallback. (Mirrors the CLI-reference "fail closed" note.)
- **Namespace shape is permanent.** Existing records are never re-pathed, so order the hierarchy by your
  dominant query: actor-first (`/users/{actor}/facts/`) for "one user, everything"; type-first for "all users."
- **Prefix-safety:** namespace templates need leading **and** trailing `/`, or `/alice/` prefix-matches
  `/alice2/` — a leak on prefix reads.
- **Filter retrievals by relevance score client-side** when the API has no threshold param; keep topK 3–5 for
  direct injection, and **cite** namespace + record id so you can catch a hallucinated/extracted "memory."
- **Don't feed assembled or sensitive turns back into extraction** — it breaks the self-reinforcing loop at the source.

## Why it's Tier-1

Any agent with memory/personalization, any provider. The cross-user leak is a real security bug that passes
every happy-path test. Same family as `declared-guardrail-not-enforced` (fail closed, verify) and
`capture-results-in-orchestrator` (identity is load-bearing — don't let the model/default supply it).

## Source
`reference/memory-patterns-from-aws-samples.md`; platform how-to in skill `agentcore-memory`.
