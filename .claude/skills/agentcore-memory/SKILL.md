---
name: agentcore-memory
description: >-
  Work with Amazon Bedrock AgentCore Memory: retrieval APIs (RetrieveMemoryRecords / ListMemoryRecords /
  GetMemoryRecord), namespace template variables and their limits, extraction jobs and extractionMode, the
  data-plane vs control-plane exception taxonomy, and which errors to retry. Use when creating/querying
  AgentCore Memory, debugging extraction, choosing a namespace scheme, or handling memory API errors. Pair
  with the principle `memory-key-by-identity-not-session` for the identity/namespace design rules.
---

# AgentCore Memory (platform how-to)

Design rules (key-by-identity, namespace shape, prefix-safety, score-filtering) live in the principle
`memory-key-by-identity-not-session`. This is the API surface.

## Retrieval APIs
- `RetrieveMemoryRecords` — semantic search (the common one). `topK` default 20, max 100.
- `ListMemoryRecords` — browse (also how you watch extraction *progress*).
- `GetMemoryRecord` — fetch by id.
- `namespacePath=` reads one prefix across strategies (replaces a per-strategy loop); `metadataFilters` apply
  hard constraints (e.g. `tier=premium`) enforced at the index.

## Namespace template variables
Resolved by the service, not the caller: `{actorId}`/`{sessionId}` from `CreateEvent` (sessionId is
**required** by summary strategies), `{memoryStrategyId}` from the strategy, customs via
`extractionConfig.namespaceVariables` (declare with `namespaceKeys` on CreateMemory/UpdateMemory).
Limits: key `[a-z][a-z0-9]*` ≤32, value lowercase ≤64, ≤5 keys, ≤5 vars/event.
⚠️ **A missing var = silent data loss; a rejected value drops the whole event (including the short-term turns).**

## Extraction
- `list-memory-extraction-jobs` is a **FAILURE log, not progress** — watch progress via `list-memory-records`.
- `StartMemoryExtractionJob` redrives.
- `extractionMode="SKIP"` on CreateEvent = store short-term, skip extraction (use for bulk import, system/tool
  messages, and sensitive turns you don't want fed back into extraction).

## Exceptions (retry only transient; never 4xx)
`ClientError`; code in `e.response["Error"]["Code"]`. **Retry with backoff+jitter:** `ThrottledException`(429),
`RetryableConflictException`(409, CreateEvent only), `ServiceException`(500). **Never retry:**
`ServiceQuotaExceeded`(402), `Validation`/`InvalidInput`(400), `ResourceNotFound`(404), `AccessDenied`(403) —
"a retry storms your own logs." Classify explicitly; never bare `except Exception`. Note the **data plane**
(`bedrock-agentcore`) and **control plane** (`bedrock-agentcore-control`) raise *different* exception sets.

## Prefer the SDK session manager
Use `AgentCoreMemorySessionManager` over a hand-rolled memory hook where available.

(Capstone used the simpler `SESSION_SUMMARY` summaryMemoryStrategy, 7-day retention — see
`retrospectives/capstone-cd14764.md`. Deeper roadmap/unread sections in `reference/memory-patterns-from-aws-samples.md`.)
