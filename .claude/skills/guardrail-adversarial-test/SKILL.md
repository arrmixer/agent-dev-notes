---
name: guardrail-adversarial-test
description: >-
  Adversarially verify that a DEPLOYED Amazon Bedrock Guardrail actually ENFORCES its policy, not just
  that the config is declared. Fire crafted inputs via bedrock-runtime apply_guardrail, read the raw
  assessments, and assert each resolves to BLOCK / ANONYMIZE / ALLOW as expected (including benign
  look-alikes that must stay allowed). Use when verifying or auditing a Bedrock guardrail, checking
  content/PII/topic/word filters, or when a guardrail "passed config checks" but you need proof it works.
  Covers the key gotcha that PII ANONYMIZE fires on OUTPUT while BLOCK fires on INPUT.
---

# Guardrail adversarial test

Embodies the rule *declared ≠ enforced*: config review proves a policy exists; only firing adversarial
inputs and reading raw `assessments` proves it works.

## Method

1. **Enumerate the claims.** List what the guardrail declares — each content filter + strength, each PII
   entity + action (BLOCK vs ANONYMIZE), each DENY topic, the word policy.
2. **Write cases per claim — both directions:**
   - a hostile input that *should* trip it, and
   - a benign look-alike that must **stay allowed** (false-positive guard — e.g. arithmetic on a stated
     price must pass while haggling is blocked).
3. **Pick the right `source` per case.** `BLOCK`/topic/content/word fire on `source='INPUT'`.
   PII `ANONYMIZE` fires on `source='OUTPUT'` (it masks what the model emits). Testing the wrong direction
   produces a *phantom* fail or hides a real one.
4. **Fire at the DEPLOYED guardrail and read raw evidence.** Call `apply_guardrail(guardrailIdentifier,
   guardrailVersion, source, content=[{'text':{'text': ...}}])`; classify from `response['action']`
   (`GUARDRAIL_INTERVENED` vs `NONE`) and the `assessments` list — never a summary pass-rate.
5. **Report what actually fired** per case, and treat any mismatch as a real enforcement gap, not a test bug.

## Known gotchas (verified)

- Managed `PROFANITY` list is **lenient** — misses mild words (damn/hell). Back it with `customWordsConfig`.
- Custom formats (e.g. `ACCT-1234`) are **not** standard PII entities — "PII on" won't catch them.
- A masking metric (e.g. a kill switch counting KILLED as pass) inflates the score — ensure the metric *can* fail.
- STANDARD tier + a narrowly-worded pricing topic lets arithmetic pass while blocking haggling (Classic over-blocks).

## Starter script

`scripts/adversarial_guardrail.py` — edit the `CASES` list for your guardrail's declared policy, set
`GUARDRAIL_ID`/`GUARDRAIL_VERSION` (env or args), run it. It prints PASS/FAIL + the raw `fired` policies
per case. Adapt per project; the method matters more than the exact cases.
