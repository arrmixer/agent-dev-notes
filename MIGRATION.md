# Migration map: agentcore-reference → agent-dev-notes

Disposition for **all 15 docs** in `~/Developer/agentcore-reference/`. Most mapped docs were *split* and
distilled (principle + evidence, or a skill), not moved wholesale. Updated 2026-10-11 after a loss-audit
found the original 5-doc plan had dropped routing, saga detail, gateway/MCP, C2-vs-C3, and portable
RAG/cost/CLI specifics — those are now addressed or backlogged below.

## Disposition

### ✅ Distilled into agent-dev-notes (fully)
| Source | Landed as |
|---|---|
| `multi-agent-patterns.md` | principles: orchestrator-is-code, capture-results-in-orchestrator, give-agents-now, structured-output-not-grounded, parallelize-gate-decompose-aggregate, shared-state-not-correct-derived-state, declared-guardrail-not-enforced, **layer-cheap-checks-before-the-llm (routing)**, **saga-compensation-correctness**, **gateway-needs-an-invocation-contract** · `retrospectives/capstone-cd14764.md` · `_archive/c2-vs-c3-framing.md` |
| `rag-runbook.md` | skill `bedrock-kb-setup` (incl. vector-store specifics + cost) |
| `deployment-architecture-template.md` | skill `deployment-readiness-checklist` (6 sections) |
| `python-vs-swift-kotlin.md` | `language/python-vs-swift-kotlin.md` (verbatim copy) |

### ◐ Partially distilled
| Source | Landed | Still owed → backlog |
|---|---|---|
| `lesson-stack-runbook.md` | async content → skill `async-provisioning` | CLI command reference + DynamoDB/S3/cost verification + account-identity diagnostics → backlog #1 |

### ✅ Distilled in round 2 (backlog cleared 2026-10-11)
| Source | Landed as |
|---|---|
| `gateway-runbook.md` | skill `agentcore-gateway` (Lambda-target path; dynamic/OpenAPI targets noted as a future section) |
| `cli-command-map.md` | skill `agentcore-cli-reference` (+ lesson-stack CLI + account-identity diagnostics) |
| `observability-runbook.md` | folded into skill `observability-xray-cloudwatch` (managed-OTel path, traces CLI, metric-filter alarm, cost gotchas) |
| `memory-patterns-from-aws-samples.md` | principle `memory-key-by-identity-not-session` + skill `agentcore-memory` |
| `python-venv-guide.md` | `language/python-venv-guide.md` |
| `tool-design-patterns.md` | principles `prefer-typed-tools-over-code-execution` + `tool-description-is-functional-not-docs` |

### 🗄 Archival (course-specific; leave in place or move to `_archive/` — superseded)
| Source | Lines | Note |
|---|---|---|
| `conversion-log.md` | 1630 | superseded by `_archive/journey-cd14764.md` |
| `capstone-skill-matrix.md` | 151 | cd14764-specific |
| `cd14764-task-map.md` | 91 | cd14764-specific |
| `README.md` | 57 | old reference index |

## Deferred backlog — ✅ FULLY CLEARED (2026-10-11)
All six items distilled, **and** the dynamic (API Gateway / OpenAPI / Smithy) targets section of
`gateway-runbook.md` is now folded into the `agentcore-gateway` skill (static-vs-dynamic trade-off, the
3-step documentation-publish fix + step-2 trap, `basePath` quirk). No partial remainders.

## Retire criteria — MET (raw sources preserved in `reference/`)
Both layers are now in-repo: the **distilled 80/20** (principles/skills) *and* the **raw source depth** — the
9 source docs were copied into **`reference/`** (grep-on-demand), so even the detail the distillation dropped
survives, and every `Source:` citation resolves to `reference/`. The only `agentcore-reference/` docs NOT
copied are the 4 **course-specific archival** ones (`conversion-log` [superseded by
`_archive/journey-cd14764.md`], `capstone-skill-matrix`, `cd14764-task-map`, old `README`).

**→ `agentcore-reference/` is fully retireable now:** confirm those 4 are disposable, then delete the dir (or
move the 4 into `_archive/` first if you want to keep them). Nothing else there is unique.

## Method
Author the distilled artifact (principle `rule + evidence`, or skill); leave the source in
`agentcore-reference/` until its row is fully migrated, then retire. `_archive/` holds narrative, not
decision-support.
