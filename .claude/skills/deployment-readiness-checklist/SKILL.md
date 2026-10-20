---
name: deployment-readiness-checklist
description: >-
  Plan a production deployment of a multi-agent system before shipping — a 6-section readiness pass covering
  runtime/network config, agent roster & model tiering, monitoring (dashboard + alarms + tracing), a cost model,
  operational runbooks (deploy/rollback/kill-switch/latency), and compliance/audit. Use when planning to deploy
  a multi-agent/agentic system to production, doing a pre-ship review, estimating run cost, or writing ops
  runbooks. Produces a plan, not code — the point is deciding WHAT to configure and WHY.
---

# Deployment readiness checklist (plan before you ship)

Work the six sections before deploying anything non-trivial. Each is a decision to make deliberately, not a
default to accept. Copy this into a per-project planning doc and fill it in.

## 1. Runtime & network config
Network mode (PUBLIC vs VPC — VPC for regulated/internal), server protocol, execution role (reuse your IaC
role so the deploy tool doesn't mint one), env vars (resource IDs, guardrail, log group). For VPC: subnets,
security groups, and the endpoints (Bedrock, S3) you'll need.

## 2. Agent roster & model tiering
List every agent: model, temperature, role, tools, est. tokens/request, requests/day. **Tier models** — a
cheap fast model for routing/retrieval, a capable model only where reasoning/synthesis lives. (In one capstone
plan the single synthesis model was ~89% of the bill — tiering is a cost lever, not a style choice.)

## 3. Monitoring: dashboard + alarms + tracing
Dashboard widgets (invocations, p50/p99 latency, error rate, guardrail blocks, RAG quality, kill-switch status).
Alarms with thresholds + actions (e.g. error >2% → kill-switch topic; p99 > Ns → page; guardrail-block spike →
security). Tracing sampling rate (100% for audit, 5–10% for high-volume prod). Build dashboards off metrics that
actually exist (`AWS/Bedrock` via `SEARCH()` + Logs Insights), not invented names.

## 4. Cost model
Per-agent monthly estimate (tokens × price × requests), plus infra (DynamoDB, KBs/S3-Vectors, CloudWatch+X-Ray,
VPC/NAT). Surface the dominant line item — it's usually one model or one always-on resource.

## 5. Operational runbooks (the part everyone skips)
Write the step-by-step for: **deploy** (test → promote guardrail version → deploy → smoke → watch), **rollback**
(revert to previous version → verify → post-mortem), **kill switch** (ack → inspect X-Ray/audit → disable →
notify), **latency investigation** (X-Ray map → per-agent latency → bottleneck → scale). Include triggers and
rough durations.

## 6. Compliance / audit (if regulated)
Audit trail (X-Ray sampling, guardrail assessment log), data retention, PII handling (block vs anonymize, and on
which direction), human-approval gates for destructive/financial actions.

## Related
Principles: `orchestrator-is-code`, `declared-guardrail-not-enforced`. Playbooks: `deploy-to-agentcore`,
`observability-xray-cloudwatch`, `async-provisioning`. Full template: `reference/deployment-architecture-template.md`.
