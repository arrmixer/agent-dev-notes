# A tool description is functional text, not documentation

> **Rule:** Write a tool's description for the model's *routing decision* — WHEN to call it and WHAT it
> returns — not as human docs. The description is the only thing the model reads to decide; its return
> format is part of the contract. Know your framework's schema quirks, and never carry meaning only in
> formatting that can be stripped.

*(Terse: `.claude/rules/tool-description-is-functional-not-docs.md`.)*

## What it means

The model picks tools by reading descriptions. A vague or doc-style description produces mis-selection and
mis-sequencing — the failure isn't in the tool's code, it's in how the tool advertised itself. Treat the
description as functional prompt text aimed at the selection decision.

## The specifics that bite

- **Framework schema collapse (Strands):** the *entire* docstring collapses into one `description` field —
  an `Args:` block gives the model **no per-parameter schema**. So any format constraint ("ISO-8601 date",
  "uppercase ticker") must read **naturally in the prose**, not sit in an Args table the model never sees as
  structure. Required-vs-optional is conveyed only by the presence of a Python default. (Verify your own
  framework's behavior — this varies.)
- **The return format is part of the contract.** Downstream agents/code consume it. Meaning carried only in
  emoji, color, or layout can be silently dropped — put the semantics in the data/text itself.

## Why it's Tier-1

Tool-selection quality in *any* tool-using agent depends on description quality; this is upstream of all the
orchestration patterns. Pairs with `orchestrator-is-code` (routing tools must advertise themselves clearly)
and `capture-results-in-orchestrator` (the return contract is what downstream reads).

## How to apply

1. Lead the description with *when to call this* and *what it returns*, in plain prose.
2. Put parameter format/constraints in the prose, not only in an `Args:` block; mark optional via defaults.
3. Make the return contract explicit and machine-meaningful — never encode meaning solely in formatting.

## Source
`reference/tool-design-patterns.md`.
