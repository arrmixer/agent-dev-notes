# Migration map: agentcore-reference → agent-dev-notes

Which existing `~/Developer/agentcore-reference/*.md` content lands in which tier. Most source docs are
*mixed* (principle + war story + platform mechanic), so they get **split**, not moved wholesale.

## Source docs and where their pieces go

| Source doc | → `principles/` (rule) | → `playbooks/` (how-to) | → `language/` or `_archive/` |
|---|---|---|---|
| `multi-agent-patterns.md` | orchestrator-is-code; parallelize gate→decompose→aggregate; shared-state ≠ correct derived state; declared≠enforced guardrail; structured-output≠grounded; give-agents-"now"; capture-in-orchestrator; raw-tool-until-toil-scales | — | C2-vs-C3 framing → `_archive/` |
| `lesson-stack-runbook.md` | async-wait: prefer IaC-owned wait, else emulate StateChangeConf | AgentCore/Bedrock CLI reference; async-provisioning recipes | lesson stack costs → `_archive/` |
| `rag-runbook.md` | retrieval faithfulness; verify retrieval before blaming code | S3-Vectors KB setup; managed vs self-managed `*SearchConfiguration` | — |
| `deployment-architecture-template.md` | — | the 6-section production-readiness checklist | — |
| `python-vs-swift-kotlin.md` | — | — | → `language/` (move whole) |

## New docs to author (gaps found post-capstone)

- `retrospectives/capstone-cd14764.md` — full-system blueprint. **(first — in progress)**
- `playbooks/deploy-to-agentcore.md` — CLI stage→configure→deploy, CodeZip/CDK, HTTP-not-MCP, guardrail-via-env-vars, ARN readback.
- `playbooks/observability-xray-cloudwatch.md` — the `@tool`→X-Ray subsegment wrapper, Transaction Search, Service Map, what to screenshot.
- `playbooks/guardrail-adversarial-test.md` — fire crafted inputs, read raw assessments; PII BLOCK=input-side / ANONYMIZE=output-side.
- `principles/` — one file per rule listed above (terse statement + the evidence pulled from the source doc).

## Method

1. Author the retrospective (surfaces which items are truly Tier-1 principles vs. platform mechanics).
2. Create `principles/<rule>.md` one at a time: rule up top, evidence below (lifted from the source).
3. Move `python-vs-swift-kotlin.md` into `language/` as-is.
4. Build `playbooks/` from the mechanic sections.
5. Leave originals in `agentcore-reference/` until each is fully migrated, then retire.
