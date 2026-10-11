# Retrospective: NovaMart multi-agent support system (Udacity cd14764 capstone)

A full worked AgentCore multi-agent system — passed 2026-10-11. This is the **reference blueprint**: how
the pieces fit, the decisions behind each, and what I'd change. Pattern depth lives in the `principles/`
and `playbooks/` docs (and the original `agentcore-reference/`); this is the map, not the territory.

## System at a glance

```
Customer request
      │
OrchestratorAgent (Haiku, temp 0, routing only — NEVER writes the reply)
      ├── InventoryAgent      (Sonnet, 0.1)  DynamoDB: order/customer facts
      ├── RefundAgent         (Sonnet, 0.1)  eligibility decision + initiate_refund
      ├── PolicyAgent         (Sonnet, 0.2)  coordinator over 3 parallel KB retrievers:
      │        ├── ReturnsRetriever   (Sonnet, 0.0) → Returns KB   (S3 Vectors)
      │        ├── ShippingRetriever  (Sonnet, 0.0) → Shipping KB
      │        └── WarrantyRetriever  (Sonnet, 0.0) → Warranty KB   [ThreadPoolExecutor]
      └── CommunicationAgent  (Sonnet, 0.3)  renders the final customer reply

Shared state: DynamoDB WorkflowState (optimistic locking)
Cross-cutting: Bedrock Guardrail (every model call) · AgentCore Memory · X-Ray + CloudWatch
Deploy: agentcore CLI → CDK → AgentCore Runtime (PUBLIC/HTTP)
```

## Component decisions (and the principle each embodies)

| Component | Decision | Why / principle |
|---|---|---|
| Orchestrator | Haiku, **temp 0**, routing-only, delegates final reply | cheap/deterministic routing; **orchestrator is code-like control flow, not a chatty agent** |
| Model tiering | Haiku router + Sonnet workers | cost: one capable model where reasoning is needed, cheap model for dispatch (L10 showed ~89% of cost is the one Sonnet) |
| Temperatures | retrieval 0.0 → synthesis 0.1–0.2 → customer prose 0.3 | determinism where facts/decisions live; warmth only in phrasing. **(Spec-exact: the reviewer failed us for 0.0 vs 0.1 — match the spec, don't reason your own value.)** |
| Shared state | DynamoDB WorkflowState, version + conditional update | **capture each agent's result in the orchestrator, don't trust the LLM to echo it**; optimistic locking prevents lost writes |
| Routing tools | all 4 go through one `_run_and_record` (read→run→write) | one code path, uniform behavior, trivially auditable |
| Policy RAG | 3 retriever sub-agents in parallel via `ThreadPoolExecutor(max_workers=3)` + `as_completed` | small-N blocking I/O → threads (not asyncio); genuine multi-agent RAG, one KB per retriever |
| Refund | reads inventory facts from WorkflowState, not a re-fetch | the ordered dependency (inventory→refund) is grounded in shared state, so the two can't disagree |
| Guardrail | one guardrail, attached to **every** model via env vars | **a declared guardrail ≠ an enforced one** — verified adversarially (9/10) |
| initiate_refund | `ConditionExpression=attribute_exists(order_id)`, failure → clean message | no phantom rows; conditional failure is a result, not an exception |

## Cross-cutting wiring

- **Guardrail attachment is config, not code.** `GUARDRAIL_ID`/`VERSION` ride in as runtime env vars;
  `build_agent_graph()` reads them and applies to each model via `_apply_guardrail()`. No per-agent change,
  no `guardrailConfiguration` block, **`serverProtocol: HTTP` not MCP** (reviewer confirmed both correct).
- **Observability is a decorator.** The `@tool` wrapper records every tool call as an X-Ray subsegment →
  the Service Map shows Orchestrator → each worker → each `KnowledgeBase:*`. Logs to CloudWatch at INFO.
- **Deploy is CLI → CDK.** `agentcore_cli.stage_runtime_code()` (CodeZip) → `configure_runtime()` →
  `deploy()` (`agentcore deploy -y`, first run bootstraps CDK ~5–10 min) → read ARN back from deployed-state.
- **Guardrail/memory creation is async with no boto3 waiter** (guardrails) / **a waiter** (`memory_created`).
  Poll-until-READY for guardrails is AWS's own documented pattern; memory has a waiter but the scaffold polls.

## If I built this again

1. **Give agents "now" via a tool.** The Refund Agent hallucinated the current date ("today is 2026-11-12")
   to compute days-elapsed. Correct only because the order was well inside the window; a boundary case (day
   58 vs 62) would flip wrongly, confidently. Inject the date deterministically — never let the model infer it.
2. **Add a `customWordsConfig`** to the guardrail. Bedrock's managed PROFANITY list missed mild words
   (damn/hell). Managed lists are lenient; back them with explicit words when it matters.
3. **Reach for Pydantic** for the DynamoDB records. The `Decimal` coercion / `json.dumps(default=str)` dance
   is exactly what a typed model dissolves. Table API was right for 3 tables; a model layer earns its keep as
   access patterns grow.
4. **Keep the orchestrator deterministic.** Haiku at temp 0 with explicit routing rules behaved perfectly
   across all 3 scenarios (ordered inventory→refund, parallel policy, math-without-workers, always comm-last).
   The win came from the *rules + code path*, not model cleverness.

## Verification notes (what the tests do and don't catch)

- `test_agent.py` is **structural** (agents instantiate, right tools, different models, ThreadPool present) —
  it does NOT check behavior or temperatures. 120/120 says "wired correctly," not "behaves correctly."
- **Behavior** was verified by the live `agentcore_orchestrator.py test` run (3 scenarios) + the **adversarial
  guardrail harness** (fire crafted inputs, read raw `assessments`). That's where real correctness shows.
- **PII direction matters when testing:** `BLOCK` fires on `source=INPUT`, `ANONYMIZE` on `source=OUTPUT`.
- **Submission-loop lessons:** dotfiles (`.env`) get stripped/hidden → ship a visible copy; a review can
  mis-describe your code (trust the code); include `config.py` + evidence screenshots proactively.

## Pointers
- Patterns in depth: `agentcore-reference/multi-agent-patterns.md`
- Deploy/async mechanics: `agentcore-reference/lesson-stack-runbook.md`
- Python idioms hit here: `language/python-vs-swift-kotlin.md` (#24 concurrency, #25 Pydantic)
