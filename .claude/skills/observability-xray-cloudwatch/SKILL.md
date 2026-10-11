---
name: observability-xray-cloudwatch
description: >-
  Wire observability for a Bedrock AgentCore multi-agent system: CloudWatch logs at INFO + AWS X-Ray tracing
  at 100% sampling, so the Service Map shows Orchestrator -> each worker -> each KnowledgeBase node. Covers
  the @tool tracing-wrapper pattern (every tool call = one X-Ray subsegment), applying a loggingConfiguration,
  enabling CloudWatch Transaction Search, generating traces, and what to screenshot. Use when adding tracing/
  logging to an agent, building the X-Ray Service Map, or debugging why traces/logs aren't showing up.
---

# Observability: X-Ray Service Map + CloudWatch logs

Goal: one trace per request, with each agent and each KB retrieval as a node on the X-Ray Service Map, and
INFO logs in CloudWatch. The map is the proof your multi-agent graph actually fanned out as designed.

## The pattern that makes it work
- **Trace tool calls with a decorator, not by hand.** Wrap the Strands `@tool` decorator so every tool
  invocation opens an **X-Ray subsegment** (and logs at INFO). Each orchestrator `route_to_*` tool becomes a
  worker-agent node; each KB retrieval becomes a `KnowledgeBase:<domain>` node. You decorate once; the map
  builds itself.
- **Config, not scattered calls.** Build one `loggingConfiguration` and apply it:
  ```python
  logging_configuration = {
      'cloudWatchConfig': {'logGroupName': LOG_GROUP, 'logLevel': 'INFO', 'enabled': True},
      'xRayConfig':       {'enabled': True, 'samplingRate': 1.0},   # 100% — full audit trail
  }
  apply_observability_config(runtime_arn, logging_configuration)   # wrap in try/except
  ```
  This creates the log group, enables **CloudWatch Transaction Search** (how AgentCore traces reach X-Ray),
  and sets the runtime env (`AGENT_LOG_GROUP`, `AGENT_LOG_LEVEL`, `AGENT_TRACING_ENABLED`, sampling rate).
- OTel auto-instrumentation is OFF because the project ships its own tracing — don't expect auto-spans.

## Generating + capturing the evidence
1. Run a live request (`python src/agent_orchestrator.py test` or `invoke`).
2. **Wait ~60 s** — traces lag; Transaction Search flips `PENDING` → `ACTIVE` once.
3. Open CloudWatch → X-Ray traces → **Service map**, set the window to "Last 5 minutes" (or wider).
4. Screenshot the full chain: `<App>-Orchestrator → each worker → each KnowledgeBase:* node`.

## Gotchas
- "No traces yet" / "no log events yet" right after deploy is normal — generate a request first, then wait.
- The trace window lapses: an old `test` run won't show if you look hours later — re-run before screenshotting.
- 100% sampling is deliberate for a graded/audited system; drop to 5–10% for cost in high-volume prod.
- A CloudWatch dashboard off real metrics: use `SEARCH()` on `AWS/Bedrock` (invocations/latency/tokens by
  ModelId) + a Logs Insights widget over the log group — no custom metrics needed, no hardcoded model IDs.
