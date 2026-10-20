# Memory patterns from the AWS samples repo

Read 2026-09-18 from
[`awslabs/amazon-bedrock-agentcore-samples`](https://github.com/awslabs/amazon-bedrock-agentcore-samples),
under `01-features/04-manage-context-of-your-agent/memory/`.

Consulted because the L09 exercise left several questions open. This records only what **changes a
conclusion or a plan** — not a summary of the repo.

---

## 1. Identity: the payload is the intended channel

`03-integrations/01-runtime-integration/runtime_memory_agent.py`:

```python
user_input = payload.get("prompt")
actor_id   = payload.get("actor_id", "default_user")   # Provide default for demo
session_id = context.session_id                         # Get session_id from context
```

AWS's own runtime sample reads `actor_id` from the **payload** — not a header, not `context.user_id`.
This confirms our finding: `agentcore invoke` cannot drive this sample either, because it sends only
`{"prompt": …}`. The raw `InvokeAgentRuntime --payload` route we used is the intended one.

### ⚠️ The sample has the leak we produced this morning

```python
# Check if session or actor changed - need to reinitialize
current_session = getattr(session_manager, "_session_id", None) if session_manager else None
if current_session != session_id:
    initialize_agent(actor_id, session_id)
```

The comment says *"session or actor"*; the code compares **only the session**. Same `session_id` with a
different `actor_id` reuses the previous actor's `AgentCoreMemoryConfig`, reading and writing the wrong
customer's namespace. `agent` and `session_manager` are also module-level globals mutated per request,
so the guard is a read-then-write race under concurrency.

**Capstone rule:** the agent cache key must be `(session_id, actor_id)`. The scaffold's
`agent_factory()` already does this — `key = f"{session_id}/{_actor_id}"` — so prefer the scaffold's
shape over this sample's.

---

## 2. Namespaces — three things we got wrong or didn't know

From `02-long-term-memory/04-namespaces/README.md`.

### Templates should start **and** end with `/`

> Templates start **and** end with `/`. The trailing slash is what stops `/users/alice/` from also
> matching `/users/alice2/`, and a retrieval argument without it is rejected.

Ours are `wanderbot/{actorId}/facts` — **neither** leading nor trailing slash. They were accepted at
`CreateMemory`, stored verbatim, and exact-match retrieval returned 5 records, so the "rejected" claim
does not hold for `namespace=` via `retrieve_memories`. We are relying on lenient behaviour.

The prefix hazard is real regardless: without a trailing slash, `wanderbot/alice-001/facts` would
prefix-match `wanderbot/alice-0011/facts`. Harmless for exact queries, a data leak for prefix ones.
**Use leading and trailing slashes in the capstone.**

### The service resolves the template; the caller does not

| Variable | Resolves from |
|---|---|
| `{actorId}` | `CreateEvent` `actorId` |
| `{sessionId}` | `CreateEvent` `sessionId` — **required** in a summary strategy's template |
| `{memoryStrategyId}` | the strategy itself |
| custom (`{tenantid}`, …) | `CreateEvent` `extractionConfig.namespaceVariables` |

So `.format()` in agent code is only for building a **retrieval** path. A template containing
`{sessionId}` genuinely cannot be resolved from `actorId` alone — which is exactly the `KeyError` we
predicted for the CLI's default templates. The design mismatch is real, not our misreading.

### Actor-first vs type-first is a day-one decision

> existing records are never re-pathed — so the shape is a day-one decision

`/users/{actorId}/facts/` makes "everything about one user" a cheap prefix query;
`/facts/{actorId}/` makes "all users' facts" cheap. Pick for the dominant query. For customer support
that is *one customer, everything* → **actor-first**, which is what our `wanderbot/{actorId}/…` happens
to be.

### Custom variables exist for multi-tenancy

`namespaceKeys` on `CreateMemory`/`UpdateMemory` declares them; `extractionConfig.namespaceVariables`
on `CreateEvent` supplies values. Limits: key `[a-z][a-z0-9]*` max 32 (no hyphens/underscores, and
never `actorId`/`sessionId`/`memoryStrategyId`), value `[a-z0-9][a-z0-9-_]*` max 64 lowercase, max 5
keys per memory, max 5 variables per event.

Their gotcha list matches our own failure class almost word for word:

> **A missing variable is not an error — it's silent data loss.** `CreateEvent` succeeds … but the
> strategy whose template can't be resolved extracts nothing, and nothing reports it.

Also: a *rejected* value drops the **whole event**, short-term turns included.

---

## 3. Retrieval — the exercise throws away the data it needs

From `02-long-term-memory/05-retrieval/README.md`.

| API | Use for |
|---|---|
| `RetrieveMemoryRecords` | relevance-ranked semantic search — the common case |
| `ListMemoryRecords` | enumerate/browse a namespace |
| `GetMemoryRecord` | fetch one by `memoryRecordId` |

`topK` **defaults to 20**, max 100. Best practice is 3–5 for direct injection — so the exercise's
`top_k=5` is at the top of the sane range.

**Each hit carries a `score`.** The exercise's hook reads only `memory["content"]["text"]` and discards
it. That is the fix for the missing relevance floor: no new API call, just stop throwing the field
away. (`searchCriteria` has no threshold parameter, so the scaffold's
`RetrievalConfig(relevance_score=0.5)` must be client-side filtering — meaning we can do the same in
four lines.)

Two more we don't do:

- **`namespacePath=`** does a prefix read across strategies. Our hook loops namespaces and issues one
  call per strategy; a single `namespacePath="/users/alice-001/"` would replace the loop.
- **`metadataFilters`** are enforced at the index — the right place for hard constraints like
  `tier=premium`, unlike LLM-side filtering.
- **Cite retrievals** — surface namespace + record id when an answer leans on memory. Useful precisely
  for catching the hallucinated-extraction case we hit.

---

## 4. Extraction — this explains our empty job list

From `02-long-term-memory/08-manage-extraction/README.md`:

> When it fails — model throttle, transient error, malformed payload — AgentCore records the attempt as
> an extraction job with `status=FAILED` and a `failureReason`.

So `list-memory-extraction-jobs` is a **failure log, not a progress indicator**. Ours returned empty
because extraction *succeeded*. That closes the question we logged as unexplained, and confirms
`list-memory-records` is the only way to watch progress.

`StartMemoryExtractionJob` redrives failed jobs.

**`extractionMode="SKIP"` on `CreateEvent`** stores an event in short-term memory but never extracts it.
Documented uses: bulk import, system/tool messages, **sensitive turns**, and replicated events. This is
a direct mitigation for our feedback loop — a turn whose text was assembled from retrieved records is
exactly a turn that should not feed extraction.

---

## 5. Error handling — our `except Exception` swallows retryable errors

From `06-production-patterns/01-error-handling.md`. Data-plane errors, all arriving as
`botocore.exceptions.ClientError` with the name in `e.response["Error"]["Code"]`:

| Exception | HTTP | Retry? |
|---|---|---|
| `ThrottledException` | 429 | **yes** — exponential backoff with jitter |
| `RetryableConflictException` | 409 | **yes** — concurrent writes to one session; **`CreateEvent` only** |
| `ServiceException` | 500 | **yes** |
| `ServiceQuotaExceededException` | 402 | no — reduce the request or raise the quota |
| `ValidationException` / `InvalidInputException` | 400 | no — fix the input |
| `ResourceNotFoundException` | 404 | no |
| `AccessDeniedException` | 403 | no — "a retry storms your own logs" |

The exercise wraps both hook methods in bare `except Exception` and logs. So a 429 or a 409 — which AWS
explicitly says to retry — is silently dropped, and the turn is simply not saved. Under concurrency
`RetryableConflictException` is the *expected* outcome of two writes to one session, which makes this a
correctness bug rather than a robustness nicety.

Note also the two API surfaces raise **different** exception sets: data plane `bedrock-agentcore`
vs control plane `bedrock-agentcore-control`.

---

## What to change in the capstone, in order

| # | Change | Why |
|---|---|---|
| 1 | Cache agents on `(session_id, actor_id)` | AWS's own sample leaks here; the scaffold gets it right |
| 2 | Never default an identity field | our leak; the sample's `"default_user"` is the same smell |
| 3 | Leading **and** trailing slashes on namespace templates | prefix-match safety; day-one decision |
| 4 | Filter retrieved records by `score` | the field is already returned and discarded |
| 5 | One `namespacePath=` query, not a loop per strategy | fewer calls, hierarchical read |
| 6 | Retry `Throttled` / `RetryableConflict` / `Service`; never retry 4xx | bare `except Exception` hides all of it |
| 7 | `extractionMode="SKIP"` for assembled or sensitive turns | breaks the feedback loop at the source |
| 8 | `AgentCoreMemorySessionManager` over a hand-rolled hook | AWS's current pattern in both sample and scaffold |

---

## Revisit after the capstone

Only the `memory/` section was read. One pass closed a question we'd logged as unexplained,
contradicted two of our own choices, and found a cross-user leak in AWS's reference sample — a hit rate
worth repeating on the rest of the repo, but **after** the capstone ships, so their design doesn't
substitute for the learning.

Unread, under `01-features/`:

| Path | Why it's likely useful |
|---|---|
| `05-authenticate-and-authorize` | the caller-asserted-identity question — JWT claims as a trustworthy actor id |
| `06-observe-evaluate-optimize-your-agent/01-observe` | maps directly onto the observability exercise |
| `01-harness`, `02-host-your-agent` | runtime hosting patterns |
| `03-connect-your-agent-to-anything` | gateway/MCP beyond what we exercised |
| `07-centralize-and-govern-your-ai-infrastructure` | the governance half of monitoring & governance |
| `08-agents-that-transact` | payments; the CLI already exposes `--payment-*` flags we never touched |

Also unread inside `memory/`: `05-security`, and in `06-production-patterns/` —
`00-multi-region-replication`, `02-cost-optimization.md`, `03-production-checklist.md`.

### How to read it

`gh` is not installed here, and `api.github.com/repos/awslabs/amazon-bedrock-agentcore-samples/contents/<path>`
returns `301 Moved Permanently`. Either follow redirects or use the numeric repo id directly:

```bash
curl -sSL https://api.github.com/repositories/1013404946/contents/01-features | \
  python3 -c "import json,sys; [print(e['type'][:3], e['name']) for e in json.load(sys.stdin)]"

RAW=https://raw.githubusercontent.com/awslabs/amazon-bedrock-agentcore-samples/main
curl -sSL "$RAW/01-features/.../README.md"
```

`WebFetch` on a `github.com/tree/...` URL returns 404 — use the API or `raw.githubusercontent.com`.

**Keep this file's format:** record only what changes a conclusion or a plan. A summary of the repo is
worth nothing that the repo isn't already worth.
