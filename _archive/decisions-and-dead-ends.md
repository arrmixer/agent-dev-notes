# Decisions & dead-ends — cd14764 (high-level snapshot)

A scannable index of the load-bearing *choices* and the *things that didn't work*, from the lessons +
capstone. This is a **snapshot** — the full "why" for anything still-true lives with its rule in
`principles/`; this table points there. Dead-ends are the high-signal part: the stuff nobody can
reconstruct later.

## Key decisions

| Decision | Why | Alternative rejected | Rule |
|---|---|---|---|
| Haiku router + Sonnet workers | cost; one capable model only where reasoning lives (~89% of spend was the one Sonnet) | all-Sonnet (expensive) | model-tiering |
| Temps 0.0 / 0.1 / 0.2 / 0.3 (route/retrieve → synth → prose) | determinism gradient | "pick my own value" — reviewer failed us on 0.0 vs 0.1 | **match the spec exactly, don't reason your own number** |
| WorkflowState + optimistic locking | capture each result in the orchestrator, don't trust the LLM to echo it | pass results through prompts | capture-in-orchestrator |
| `ThreadPoolExecutor(3)` for parallel RAG | small-N blocking I/O (boto3/Strands are sync) | `asyncio` (needs async libs all the way down) | concurrency: threads for small-N |
| Guardrail via env vars, `serverProtocol: HTTP` | config not code; one guardrail on every model | `guardrailConfiguration` block / MCP (wrong for this project) | guardrail-as-config |
| Self-managed S3-Vectors KBs (`vectorSearchConfiguration`) | capstone spec | managed KBs (`managedSearchConfiguration`) — that was L8 | KB retrieval config |
| Poll-until-READY for guardrail | boto3 `bedrock` has **no waiter** (verified); AWS's own documented pattern | a waiter (doesn't exist for guardrails; memory has one) | async-provisioning |
| `dynamodb.Table` (no ORM) | right for 3 tables, dependency-free | PynamoDB/Pydantic | raw-tool-until-toil-scales |

## Dead-ends & gotchas (tried → failed → fix)

| What happened | Why | Fix / takeaway |
|---|---|---|
| us-east-1 CloudFormation hung ~20 min on a trivial S3 bucket (×2, L8) | lab-account flakiness | switched that lesson to us-west-2; capstone stack later worked in us-east-1 |
| KBs returned 0 passages (L8) | `vectorSearchConfiguration` on **managed** S3-Vectors KBs | managed KBs need `managedSearchConfiguration`; it failed as a fake "tool config error" |
| `InvalidClientTokenId` on deploy (L11) | stale AWS creds in `.env` overrode `~/.aws` via `load_dotenv()` | keep creds in `~/.aws` only, never `.env` |
| Guardrail blocked the math scenario | Classic tier over-broad on "pricing" | STANDARD tier + pricing topic defined to *exclude* arithmetic on a stated price |
| Fake 13/15 guardrail pass rate (L9) | kill switch masked filter misses (KILLED scored as pass) | score only real detections; verify from raw `assessments` |
| Refund approved with "today = 2026-11-12" | agent **hallucinated the current date** (real: 10-10) | right by luck (inside window); inject date via a tool — never let the model infer it |
| Profanity "damn/hell" not blocked | Bedrock **managed** PROFANITY list is lenient | add `customWordsConfig` for the words that matter |
| PII email/phone looked unenforced | tested `apply_guardrail` on `source=INPUT` | `ANONYMIZE` fires on **OUTPUT**, `BLOCK` on INPUT — test each on the right source |
| `.env` reported "missing" by reviewer | dotfiles get stripped/hidden in the uploaded zip | ship a visible copy (`env.txt`) alongside |
| `deploy` printed the usage banner | the `deploy` arg didn't reach the script | not a bug — the dispatcher only banners on an unrecognized/empty command |
| Review mis-described our code (MCP, `put_agent_runtime_logging_configuration`) | reviewer pattern-matched a template | **trust your code over a review that describes something that isn't there** |
| My sandboxed Bash hit the wrong AWS account | different creds than the user's shell | user runs all AWS calls via `!`; agent diagnoses, doesn't execute them |

## The arc in one line
Convert 11 lessons to *learn to judge* LLM output → build the capstone → pass 120/120 → two review
bounces (missing evidence, then a temp-spec miss) → pass → pivot to this reusable knowledge base.
