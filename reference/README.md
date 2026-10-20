# reference/ — deep reference (grep on demand)

The **raw, command-dense source docs** that the `principles/` and `.claude/skills/` distill. When a skill
summarizes a flag table or a long command chain, the full version lives here. Read/grep these when a skill
lacks a specific detail — they are **not** loaded every session (that's rules) and **not** auto-invoked
(that's skills).

**Scope note:** this is **repo-local** depth. Skills travel to other machines via the plugin; these docs do
not — you consult them when working *in* this repo (or by pointing an agent at the path). Skills = portable
80/20; `reference/` = the deep 20%.

## Contents — the raw source the distilled layer was built from
Every principle's `Source:` line and the skills' "deeper" pointers resolve here.
| doc | distilled into |
|---|---|
| `multi-agent-patterns.md` | ~10 principles (orchestrator-is-code, routing, saga, parallelize, shared-state, guardrails, give-agents-now, structured-output, capture-results, gateway) + the retrospective |
| `cli-command-map.md` | skill `agentcore-cli-reference` |
| `lesson-stack-runbook.md` | skills `agentcore-cli-reference` + `async-provisioning` |
| `gateway-runbook.md` | skill `agentcore-gateway` |
| `observability-runbook.md` | skill `observability-xray-cloudwatch` |
| `memory-patterns-from-aws-samples.md` | skill `agentcore-memory` + principle `memory-key-by-identity-not-session` |
| `rag-runbook.md` | skill `bedrock-kb-setup` |
| `deployment-architecture-template.md` | skill `deployment-readiness-checklist` |
| `tool-design-patterns.md` | principles `prefer-typed-tools-over-code-execution` + `tool-description-is-functional-not-docs` |

This is the full raw-source layer harvested from the retired `agentcore-reference/`. The principles/skills
are the distilled 80/20; these are the depth you grep when a detail isn't in the distillation. See
`../MIGRATION.md` for the per-doc disposition.
