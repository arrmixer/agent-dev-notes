# Deployment architecture template — plan before you ship a multi-agent system

Copy this per project and fill each section. It's the checklist form of cd14764 L10: the decisions you
*will* forget to make otherwise. Provider-flavored for AgentCore/Bedrock but the sections are universal.
Rule of thumb: if you can't answer a section, you're not ready to deploy — not because the code isn't
done, but because you haven't decided how it runs, costs, fails, and recovers.

Each section lists **the questions to answer** + a **gotcha** learned the hard way (cross-refs to
`multi-agent-patterns.md`).

## 1. Runtime & network config
- Network mode: **PUBLIC** (internet-facing) or **VPC** (internal-only)? Regulated/financial/PII → VPC.
  If VPC: vpcId, subnetIds, securityGroupIds, and VPC endpoints for Bedrock + S3.
- Protocol (e.g. MCP), execution role ARN, environment.
- **Every resource ID (KBs, tables, guardrail) goes in `environmentVariables`, never hardcoded in agent code.**
- Guardrail: which id + **which non-DRAFT version**? (Note: the guardrail is applied at *model-invocation*
  time, not as a runtime parameter — a common misconception.)
- *Gotcha:* lessons/examples default to PUBLIC; production regulated systems almost always want VPC.

## 2. Agent roster & model tiering
- List every agent: name, role, tools, model, temperature, est. tokens/request, requests/day.
- **Tier models by job:** cheap model (Nova Lite) for routing + retrieval + high-volume lookup;
  premium model (Sonnet) ONLY for the step that needs reasoning/synthesis. Deterministic steps → temp 0.0.
- *Gotcha (quantified in L10):* the one premium-model agent can be ~90% of the entire bill. A high-volume
  router on a premium model can cost ~**59× more** than on a cheap model for zero quality gain. Compute
  the cost table (§4) before committing — cost intuition without arithmetic is unreliable.

## 3. Monitoring: dashboard + alarms + tracing
- **Dashboard widgets:** total invocations, latency P50/P99, error rate, guardrail blocks by type,
  retrieval/RAG quality, kill-switch status.
- **Alarms with explicit thresholds + actions:** error-rate (tighter for regulated: 2% not 5%),
  latency P99 (e.g. 8s), guardrail-violation spike (e.g. 50/5min → security + compliance). Each alarm
  needs a *destination* (SNS → pager / kill-switch Lambda), not just a threshold.
- **Tracing:** X-Ray enabled, sampling rate (higher — e.g. 10% — for audit trails), and annotations
  (query_type, agent_name, model_id, tenant/client_tier, compliance_flag).
- *Gotcha:* a volume-based circuit breaker (kill switch) can **mask** content-filter holes — a high
  blocked/killed rate may mean "rate-limited," not "caught." See multi-agent-patterns §Guardrails.

## 4. Cost model
- Per-agent monthly = ((in_tok/1k × in_price) + (out_tok/1k × out_price)) × req/day × 30. (~60/40 in/out.)
- Add infra: DynamoDB (per table), KBs (storage + embeddings + queries), CloudWatch (logs + metrics +
  $3/dashboard), X-Ray (× sampling rate), VPC (NAT ~$32/mo + endpoints). 
- Produce a line-item table with a TOTAL. **This is where model-tiering decisions get justified or killed.**

## 5. Operational runbooks (the part everyone skips)
Write these as concrete numbered steps BEFORE an incident, not during one:
- **Deploy:** test suite green → validate/promote guardrail version → deploy runtime → smoke-test N txns →
  watch error+latency for N min. Define the **rollback trigger** (e.g. err >2% or P99 >8s).
- **Rollback:** identify previous stable version → revert → verify on smoke tests → post-mortem.
- **Kill switch:** ack alert → check X-Ray → check audit logs → disable if attack/fraud → notify
  compliance. Define the **triggering threshold** + whether **human approval** is required.
- **Latency investigation:** X-Ray service map → per-agent latency → isolate bottleneck
  (agent / KB / DynamoDB throttle) → scale → re-measure.

## 6. Compliance / audit (if regulated)
- Audit table (separate from state), retention window, full guardrail audit log.
- Human-approval gates on destructive ops (kill switch, data deletion).
- *Gotcha:* a **declared** guardrail policy ≠ an **enforced** one — verify adversarially against raw
  `assessments`, don't trust the config. See multi-agent-patterns §Guardrails.

---
*Source: cd14764 L10 (vectrabank-architecture). Prime candidate to promote to a Claude Code skill — see
the `reference-docs-to-skills` memory. Re-verify cost numbers + API shapes on next real use.*
