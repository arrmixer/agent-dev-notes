# Structured output is not grounded output

> **Rule:** A schema-valid output is not a *true* one. The schema guarantees shape, not that the content is
> supported by evidence. Validate grounding (does each claim trace to a retrieved passage / tool result?)
> as a **separate** check from validating the schema — and degrade gracefully instead of backfilling.

*(Terse: `.claude/rules/structured-output-not-grounded.md`.)*

## What it means

Asking a model for structured output (fixed sections, a JSON schema, a Pydantic model) makes the *form*
reliable. It does nothing for *truth*. A model handed a 3-section template will happily fill all three —
even the one with no supporting evidence — because the schema demands a value there.

## Evidence

- **L8 (multi-agent RAG):** the synthesis model produced the required sections, but **backfilled an
  evidence-less section with mislabeled content** to satisfy the structure. The schema was satisfied; the
  grounding was not. "All sections present" read as success and masked a fabrication.

## Why it's Tier-1

Schema validation (JSON Schema, Pydantic) *feels* like correctness because it's binary and automatable — but
it only checks form. Grounding is an orthogonal property that needs its own verification. Same family as
[declared-guardrail-not-enforced](declared-guardrail-not-enforced.md) (config ≠ behavior) — the presence of
a structure never certifies the content inside it. Applies to any RAG / extraction / tool-synthesis system.

## How to apply

1. Require **evidence per claim** — citations, source IDs, or the tool result each field derives from.
2. Check each section/field **maps to a real retrieved source**; a field with no backing evidence is a defect,
   not a filled template.
3. **Degrade gracefully:** instruct the model to say "the source doesn't specify this" rather than invent a value.
4. Test with an input where *part* of the schema has **no** supporting evidence — a correct system leaves it
   empty/disclaimed; a broken one fabricates to fill the shape.

## Source
`agentcore-reference/multi-agent-patterns.md` (§multi-agent RAG), L8 clinical-literature exercise.
