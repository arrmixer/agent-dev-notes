# Prefer typed-arg tools; justify a code-execution sandbox before reaching for one

> **Rule:** Default to a plain, typed `@tool` function. Add a code-execution sandbox / remote executor only
> when you can name why: the code is untrusted (LLM/user-authored), dynamic (assembled at runtime from
> data), needs a provable per-call isolation boundary, or needs capabilities the agent runtime lacks.
> Otherwise it's a network hop + serialization cost for nothing. When you *do* run generated code: minimize
> the data/code mix surface, verify locally first, and pair it with a degraded-but-correct fallback.

*(Terse: `.claude/rules/prefer-typed-tools-over-code-execution.md`.)*

## The practices (each a sub-rule)

1. **Typed function is the default.** A sandbox buys isolation; if nothing untrusted/dynamic is happening,
   it buys only latency and serialization. Make yourself justify it.
2. **Tool-authors-code beats LLM-authors-code.** When the model only supplies *values* (typed args), it
   can't emit arbitrary code, so there's little to isolate and the design is far safer. Be honest that the
   sandbox does less work in this shape rather than overselling it.
3. **If you must marshal code as a string, shrink the data/code mix to ONE interpolation point.** A remote
   stateless executor can't take function objects — only self-contained text. Big f-strings force doubled
   braces / `!r` quoting that fail only at runtime, remotely. Isolate inputs on one line
   (`f"INPUT = {json.dumps(inputs)}\n" + STATIC_ALGORITHM`). Same logic as parameterized SQL vs concatenation.
4. **Verify generated code locally before any remote run** — compile + exec locally, and assert against
   expected values derived *independently from the source data*, not from what the code itself produces
   (or the test is circular).
5. **Pair every remote/sandbox dependency with an in-process fallback** that returns a degraded-but-correct
   result and flags it (`degraded: true`) instead of raising at the user. Telling signal: if the fallback is
   trivial, the sandbox wasn't load-bearing — notice that and reconsider whether you need it.

## Why it's Tier-1

Agent frameworks make code-execution tools look powerful, so people reach for them by default and inherit
cost, fragility, and a false sense of safety. Provider-agnostic. Pairs with `raw-tool-until-toil-scales`
(don't add machinery you can't justify) and `declared-guardrail-not-enforced` (an isolation boundary you
didn't test isn't one).

## Source
`reference/tool-design-patterns.md`.
