# Use the raw tool until the toil scales, then add a layer

> **Rule:** Reach for the raw/stdlib tool first; add an abstraction layer (ORM, framework, async model) only
> when the *repetitive toil* actually scales. The signal to add a layer is the same manual friction
> repeating across many sites — not aesthetics, and not "a framework exists." A dependency has real cost;
> pay it when it earns its keep.

*(Terse: `.claude/rules/raw-tool-until-toil-scales.md`.)*

## What it means

Every abstraction trades a one-time learning/dependency cost for reduced per-use toil. At small N the trade
loses (you carry a framework to save three lines); as N grows the trade flips (manual toil spreads to dozens
of sites). The discipline is to locate *where* you are on that curve, not to default to either end.

## Evidence

- **DynamoDB access (capstone):** `boto3.resource(...).Table()` was right for 3 tables — a model layer
  (PynamoDB) would be over-engineering. But the friction it leaves (numbers as `Decimal`, reserved-word
  aliases, hand-written `UpdateExpression`) is *exactly* what a layer dissolves once access patterns multiply.
  Right tool for N=3; wrong tool for N=30.
- **Concurrency (capstone):** `ThreadPoolExecutor` (raw) was right for the 3-way parallel RAG; `asyncio`
  (the heavier model) earns its keep only at large N with async-native libraries. Same curve, different axis.

## Why it's Tier-1

It's a judgment principle, not a tech tip — it governs DB layers, concurrency models, agent frameworks,
config systems, and when to wrap vs. inline. Both failure modes are real: **premature abstraction** (carry a
dependency you didn't need, fight its opinions) and **too-late abstraction** (copy-pasted toil rots across
the codebase). The tell is *repetition*, measured, not vibes. (Language-specific note: Python culture is
"stdlib until it hurts," unlike framework-first ecosystems — see `language/` #25.)

## How to apply

1. Start with the raw tool; get it working.
2. Watch for the **same manual toil repeating** across sites (the `default=str` dance, the reserved-word
   alias, the retry boilerplate).
3. When that repetition crosses from "a few" to "spreading," introduce the layer deliberately — and only the
   layer that removes *that* toil.
4. Don't add a framework for N=few; don't hand-roll the same thing N=many times. Name the N where you'd switch.

## Source
Capstone DynamoDB + concurrency choices; `agentcore-reference/*` and `language/python-vs-swift-kotlin.md` #24–25.
