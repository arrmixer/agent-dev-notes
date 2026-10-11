# Declared ≠ enforced

A declared policy is not an enforced policy. For any guardrail, output validator, schema, or permission
check: verify it by firing **adversarial inputs** and reading the **raw results** — never trust the config,
the settings screen, or a pass-rate. Test false positives too (benign look-alikes must still pass). Test
every direction an action can take (e.g. Bedrock PII `ANONYMIZE` fires on output, `BLOCK` on input).

If a metric can't fail, it isn't measuring enforcement. A passing config/structural check means "wired
right," not "behaves right" — prove behavior, don't assume it.
