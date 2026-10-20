# A declared policy is not an enforced policy

> **Rule:** Verify guardrails, validators, and permission policies by firing adversarial inputs and
> reading the **raw results** — never trust the config, the settings screen, or a pass-rate. If a metric
> can't fail, it isn't measuring enforcement.

*(Terse always-on form: `.claude/rules/declared-guardrail-not-enforced.md`. Companion skill:
`.claude/skills/guardrail-adversarial-test/`.)*

## What it means

What a guardrail is *configured* to do (declared) ≠ what it *actually does* against real inputs (enforced).
Config review proves the policy exists; only behavioral testing proves it works. The two diverge constantly.

## Evidence (all observed first-hand)

- **Managed detectors miss cases you assume they catch.** Config said "PROFANITY list enabled" ✓ — but the
  deployed guardrail waved through "damn" and "hell." Declared, not enforced. (Capstone adversarial run: 9/10.)
- **Custom formats aren't built-in types.** "PII protection on" doesn't cover *your* `ACCT-1234` — it isn't a
  standard PII entity, so it sails through (L9 trading-compliance).
- **Actions fire in an untested direction.** Bedrock PII `ANONYMIZE` triggers on model *output*, `BLOCK` on
  *input*. Test the wrong `source` and a working guardrail *looks* broken (phantom gap) — or an untested
  direction hides a real one. (Capstone: email/phone looked unenforced on INPUT; fine on OUTPUT.)
- **A masking metric lies.** L9 kill switch counted KILLED requests as passes → fake 13/15 when real coverage
  was 10/15. The number couldn't fail, so it measured nothing.

## Why checking config isn't enough

The capstone rubric only checked the guardrail had the right policies *declared* (3 DENY topics present,
PROFANITY enabled…). A guardrail that declares everything but enforces poorly **passes that check**. The
only ground truth is firing crafted inputs at the *deployed* guardrail (`apply_guardrail`) and reading the
raw `assessments` to see what actually fired vs. passed.

## Why it's Tier-1 (provider-agnostic)

The declared-vs-enforced gap is everywhere in agent systems — guardrails, output validators, JSON schemas,
tool permissions, RAG grounding. **Config review ≠ behavioral verification.** Same family as:
- *structured output ≠ grounded content* (a schema doesn't make the content true)
- *optimistic locking ≠ correct derived state* (the lock protects writes, not your math)
- *a passing structural test ≠ correct behavior* (120/120 said "wired right," not "behaves right")

The headline feature being present never means the behavior is correct. Prove behavior adversarially.

## How to apply

1. Enumerate what the policy *claims* to do (per declared rule/filter/entity).
2. For each claim, craft inputs that *should* trip it **and** benign look-alikes that should NOT (false-positive check — e.g. "5 items at $29.99, 10% off" must stay allowed while haggling is blocked).
3. Fire them at the deployed thing; read the **raw evidence** (`assessments`), not a summary pass-rate.
4. Test every direction/mode the action can take (INPUT vs OUTPUT for PII).
5. A gap found here is real; a "pass" from config review is not proof. See the companion skill.

## Source
`reference/multi-agent-patterns.md` (§guardrails), capstone `enhancements/adversarial_guardrail.py`.
