# Multi-agent patterns

Findings from cd14764 (Course 3 — agentic architectures). Each entry is a *choice* with the evidence
that settled it, not a tutorial. This is the C3 landing doc, the multi-agent equivalent of C2's
[gateway-runbook.md](gateway-runbook.md) / [rag-runbook.md](rag-runbook.md).

---

## The self-test: questions to answer by the end of C3 + capstone

A north-star framing of orchestration (paraphrased): *an orchestration layer needs flexibility to add
agents without redesign; an orchestrator that delegates, handles responses, manages errors, and absorbs
new capabilities. Managing workflow, state, and errors is the script the conductor follows.*

That reads as gold, but it's a slogan until each claim is backed by a **mechanism plus evidence from a
real run** — not a plausible-sounding sentence. So it lives here as a scoreboard, not a conclusion. If I
can answer every row concretely by the end of the course, I can judge whether an implementation (mine or
an LLM's) is doing it right. **A row flips to ✅ only when a mechanism and a run back it** — the same bar
this whole doc holds itself to.

| Question (unpacked from the paragraph) | Status | Mechanism / where |
|---|---|---|
| What makes a system extensible — add an agent without redesign? | ✅ | Uniform agent interface + composition → Orchestration §Extensibility (L1–L4) |
| Orchestrator: code or LLM? | ✅ | Code unless the decision needs judgment → Orchestration §correction. L4 run: code orchestrator drove gate + parallel + conditional, all correct |
| How does the orchestrator reliably consume a worker's result? | ✅ | Structured-state capture, not prose → Capture principle (L1–L2) |
| When is parallelism safe, and how to split/merge? | ✅ | Gate→decompose→aggregate (L3) |
| How do you manage errors — which modes, which responses? | ◐ | Fail-fast gate (L4: PKG-003 halted, saved 4 calls); fail-soft retry; **saga compensation/rollback** (L7 run: CHK-003 undid payment→inventory in reverse, barrier 2/2 gated resolution, only *completed* steps compensated). Awaiting: guardrail refusals (L9) |
| How is state shared across agents at production scale? | ✅ | DynamoDB + optimistic locking (version + `ConditionExpression`, re-read on conflict) → Shared-state §. L6 run: 4 agents on one record, 1 conflict resolved, no lost writes — but locking ≠ correct *derived* state (concurrent run ended `confirmed` not `ready`) |
| How do you delegate when intent is ambiguous? | ✅ | Cascade: priority→rule→LLM→fallback → Routing §. L5 run: 75% routed by cheap layers, LLM as recall backstop, 20/20 correct |
| How do you evaluate and guard the system? | ✅ | Bedrock Guardrails (`apply_guardrail`) + kill switch + rate limit + LLM-as-judge → Guardrails §. L9 run: adversarial testing exposed that a *declared* policy ≠ an *enforced* one — Bedrock's managed detectors missed custom formats, mild profanity, a worded threat; and the kill switch masked those misses (KILLED scored as pass). Verify guardrails adversarially, reading raw assessments |
| How do you deploy, observe, monitor in production? | ✅ | Deployment-architecture planning (L10): VPC runtime, 6-widget dashboard + 3 alarms, X-Ray 10%, ops runbooks, cost model (one Sonnet agent = 89% of a $2,630/mo bill; router ~59× cheaper on Lite). Inter-agent comms via gateway (L11): registration→discovery→invocation; run showed the sim is dynamic in registry+discovery but static in invocation — real MCP Gateway is the contract that closes it → §Gateway. Capstone operationalizes with the real X-Ray service-map screenshot |

---

## Don't make correctness depend on the LLM echoing structured data — capture it in the orchestrator

**The principle:** a system prompt is a *request*, not a *guarantee*. When the orchestrator can obtain a
tool's output deterministically, do that instead of parsing what the agent chose to say about it. Reserve
prompt-shaping for genuinely generative text, not for data you already have in hand.

**Where this showed up** (lesson 1, smart-home device management). Two versions of the same 3-agent
pipeline, differing in exactly one design decision:

| | How the coordinator gets a worker's result | Depends on the prompt? |
|---|---|---|
| **Demo** (`healthcare_triage.py`) | A module-level `_tool_results` dict. Each tool writes its raw JSON there; the coordinator reads *from the dict* and ignores the agent's text reply. | No |
| **Exercise** (`smart_home_device_mgmt.py`) | Parses the agent's **text return value** with `json.loads`, plus a regex `{...}` fallback. | Yes |

The demo says why, in a comment (lines 118–121):

> *"LLMs may rephrase or modify tool output in their text response, so we capture the raw tool result
> here for reliable downstream use."*

**Two failure surfaces the prompt-dependent path adds**, neither of which the deterministic path has:

1. **Format drift → crash.** The model prepends "Sure, here's the data:" and `json.loads` throws on the
   first token. The exercise solution needs a `try/except` + `re.search(r'\{[\s\S]*\}', ...)` salvage per
   step just to survive this.

   **Observed, not hypothetical** (lesson-1 run, 2026-09-26): with all three worker prompts explicitly
   ending in *"output ONLY the raw JSON — no preamble, no explanation, nothing else,"* and
   `temperature=0.0`, the model *still* wrapped every tool result in a ```` ```json ```` markdown fence.
   `json.loads` failed on the leading fence; the regex salvage extracted the `{...}` and the run came out
   clean. The strict prompt did not prevent the decoration — the orchestrator-side parse did. That is the
   whole argument in one data point: the prompt was followed in spirit and still broke the naive parse.
2. **Paraphrase → silently wrong.** Worse than a crash: the model rounds `92.5` to `93` or renames a key
   while "echoing" it, and the pipeline runs clean on corrupted data.

Everything the exercise then spends on defense — the strict "output ONLY the raw JSON, nothing else"
prompt line, `temperature=0.0`, the regex fallback — is *buying back* reliability that the demo's
`_tool_results` capture had for free. That effort is the cost of the design choice, not a feature.

**The tell:** if you are writing "output ONLY the JSON" in a prompt *and* a regex to extract JSON from the
reply, the data should not have been routed through the LLM's text channel at all.

**Stronger still** (lesson-2 run, 2026-09-26): a hate-speech post's screening agent had its *entire text
response blocked by a Bedrock guardrail* — the console showed `Classification: - The generated text has
been blocked by our content filters.` The pipeline still classified it HARMFUL and routed it to the notice
agent, correctly, because `main()` reads the `screening_cache` dict the tool wrote *before* the model's
text was generated and blocked. Had routing parsed the agent's text, a hate-speech post would have slipped
through on empty input — a moderation failure caused by a *parser*, not a model. The deterministic capture
made the text channel's total failure a non-event. This is the same lesson as the markdown-fence case, at
the far end of the severity range: the more the output matters, the less you want it traveling as prose.

### How it scales: DynamoDB workflow state

Project Task 2 is the same shape at production grade: the five agents don't hand results to each other as
chat text — they write to a **shared workflow-state store** (DynamoDB), and the orchestrator reads it back
(`agent_orchestrator.py` → `_read_workflow_state`). Same principle, durable backend: state is captured
deterministically and never scraped from a model's prose. So the reliable pattern to carry forward is
*worker writes structured state → orchestrator reads state*, not *worker narrates → orchestrator parses*.

### When prompt-shaping is still the right tool

The output is genuinely generative — a customer-facing summary, an explanation, a rewrite. There is no
"raw tool result" to capture because the text *is* the product. That is the Communication agent's job, and
there the prompt carries real weight. The rule isn't "never trust the prompt"; it's "don't trust it for
data you can capture."

Related: [tool-design-patterns.md](tool-design-patterns.md) — "a tool's return format is part of its
contract"; and the retry-vs-degrade discussion, another case of not letting a model paper over a
mechanical failure.

---

## Parallelizing agent work: gate → decompose → aggregate

Three decisions, in order. The common mistake is jumping to "how do I split this" before asking "should
I, and is it safe." Framework worked out on lesson 3 (contract compliance).

### Layer 1 — the gate: should you parallelize, and is it safe?

Four checks. If any is "no," parallelism is wrong or premature — stay sequential.

1. **Independent?** The work items must not depend on each other's output. Three specialists analyzing
   one contract (regulatory / financial / IP) don't read each other's results — that independence is
   what *permits* parallelism. If task B needs task A's output, threads are the wrong tool.
2. **State partitioned or locked?** Concurrent writes must not collide. Lesson 3 gives each worker its
   own cache (`regulatory_cache` / `financial_cache` / `ip_cache`), each keyed by `contract_id`, so no
   two threads touch the same key — no lock needed. The unsafe version is one shared dict written by all
   workers; then you need a `Lock` (relying on GIL atomicity is fragile). **Partition the state.**
   *Observed corollary:* `stdout` is shared state too. The lesson-3 parallel run garbled the three
   agents' `<thinking>` blocks into each other (`<thinking><thinking> I need to check the contract
   CONTRACT-00 To assess the intellectual property...`) because three threads wrote one unsynchronized
   stream. Harmless here (caches are separate, final report clean) but it's the one piece of shared state
   the exercise did *not* partition — and the reason production parallel code uses structured per-worker
   logging, never `print()`.
3. **Fan-in barrier correct?** Aggregation must wait for *all* workers. `as_completed` blocks until every
   future finishes; only then does the synthesizer run. You can't merge results that don't exist yet.
4. **Right concurrency primitive?** Agent work is **I/O-bound** (waiting on model/API HTTP calls), so
   `ThreadPoolExecutor` is correct — the GIL is not the bottleneck when you're blocked on the network.
   multiprocessing adds pointless serialization overhead; asyncio needs an async SDK. Threads for I/O.

### Layer 2 — decomposition: how to split the work

- **Sectioning / sharding** — a large *divisible* input split into chunks, one per worker (each chapter
  of a book summarized concurrently).
- **Aspect-based** — one subject, multiple *independent facets* analyzed at once (technical specs, user
  sentiment, competitor pricing for a product; regulatory/financial/IP for a contract). ← **lesson 3**.
- **Identical-for-diversity / voting** — the *same* task run N times with varied model/prompt/temperature,
  for a range of creative outputs or for consensus reliability.

### Layer 3 — aggregation: how to merge (largely *determined* by Layer 2)

Decomposition and aggregation are **coupled, not free mix-and-match** — the way you split picks the way
you merge:

| Decomposition | Natural aggregation |
|---|---|
| Sectioning / sharding | **Concatenation** — join the chunk outputs in order |
| Identical-for-voting | **Voting / majority** or **comparison-and-selection** |
| Aspect-based | **Synthesizer / aggregator LLM** — weave diverse facets into one coherent verdict ← lesson 3 |

### Two cautions

- **Voting is often a trap.** Majority-rule only raises accuracy when the errors are *independent*. The
  same model + same prompt run N times tends to make the *same* mistake in a correlated way — so you pay
  N× the cost for near-zero reliability gain. Voting earns its keep only with **diversity**: different
  models, prompts, or a higher temperature. Without that it is theater: an expensive way to get the same
  wrong answer three times.
- **A synthesizer LLM re-introduces the capture question above.** Feed the aggregator the workers'
  *structured* outputs (state/cache), not their chat text, and instruct it to use only those findings and
  *not* re-analyze the source. Lesson 3's synthesizer prompt does exactly this: *"Use ONLY the specialist
  findings returned by the tool. Do not re-analyze the contract yourself."* Otherwise the aggregator
  becomes a fourth place for prose-parsing and re-derivation errors to creep in — and it's running on an
  expensive model.

### What the numbers actually showed (lesson-3 run, 2026-09-26)

| Contract | Decision | Parallel | Sequential | Speedup |
|---|---|---|---|---|
| CONTRACT-001 | APPROVE | 8.3s | 10.7s | 1.3× |
| CONTRACT-002 | REJECT | 8.4s | 11.2s | 1.3× |

Only **1.3×**, not 3×, and the gap *is* the lesson:

- **Parallel latency = the slowest worker, not the average.** The three specialists ran concurrently in
  ~3.6s — but that is the Claude financial agent's time (the straggler); Nova Lite and Nova Pro finished
  earlier and their threads just waited. A fan-out is only as fast as its slowest branch, so one slow
  model caps the whole stage. The specialist portion alone was ~1.7× (3.6s parallel vs ~6s summed).
- **The serial tail dilutes it (Amdahl's law).** The synthesizer cannot start until all specialists
  finish and it costs ~4.5s — the same in both runs. That un-parallelizable ~4.5s is ~45% of the
  sequential total, which mathematically caps total speedup no matter how many specialists you add. You
  speed up the wide part and still pay full price for the serial part.

### How it scales: parallelizing `search_all_policies`

`search_all_policies` (Task 2) is aspect-based decomposition + synthesizer: three Knowledge-Base
retrievers (returns / shipping / warranty) fanned out in parallel, results merged. Same four gate checks
apply — the retrievers are independent, write disjoint results, the agent waits for all three, and the
calls are I/O-bound. The lesson-3 engine is the template to lift. **But** the numbers above set the
expectation: the fan-out pays off only if the retrievers dominate latency and the merge is cheap. If the
synthesis/answer step is expensive, the serial tail shrinks the win — so measure before assuming 3×.

---

## Orchestration: the conductor is code, not a prompt

A good north-star framing (worth grounding, not just quoting): *an orchestration layer needs flexibility
to add agents without redesign; an orchestrator that delegates tasks, handles responses, manages errors,
and incorporates new capabilities. Managing workflow, state, and errors is the script the conductor
follows.* Grounded on lesson 4 (package-delivery workflow), each virtue becomes a mechanism.

### The load-bearing correction: the orchestrator is usually deterministic code, not an LLM "agent"

The delivery workflow's own comment says it, and it is the whole lesson:

> *"This is a CODE decision (deterministic if/else), not an LLM decision. Routing logic that can be
> expressed as code should be — it's cheaper, faster, and impossible to hallucinate."*

`orchestrate_delivery` is plain Python. Control flow — sequence, branch, parallel, gate, retry — is what
code does reliably and an LLM does not. Reserve an *LLM* orchestrator for the case where the routing
decision itself needs judgment.

- **Code orchestrator** — lesson 4 delivery: "same sender/dest country?" is a boolean, so `if` beats a model.
- **LLM orchestrator** — the project (Task 2): `route_to_*` tools, because classifying a customer's intent
  needs judgment a boolean can't express.

"Orchestrator agent" is a misleading default. Ask first whether the decision is expressible as code; only
what genuinely needs judgment goes to a model. The conductor's script should be code you can read, test,
and breakpoint — a prompt-based conductor can be neither unit-tested nor stepped through.

### Three control-flow primitives (the conductor's vocabulary)

The delivery workflow contains all three, which is why it's a good reference:

1. **Sequential gate** — a precondition that must pass or the workflow halts. Address invalid → return
   early with `halted: True` + a `halt_reason`, *skipping the four downstream agents*. Fail-fast: never
   spend on work that cannot matter.
2. **Parallel fan-out** — independent steps at once (label + insurance + carrier), the lesson-3 engine
   reused. Same four gate checks apply (independent, disjoint state, fan-in barrier, I/O-bound).
3. **Conditional routing** — branch on a decision; in code when expressible (domestic vs international),
   via LLM only when judgment is required.

### Error handling has two distinct shapes — don't confuse them

- **Fail-fast (the gate):** a precondition failure that makes downstream work pointless → stop now,
  report why. A bad address can't be labeled, insured, or shipped, so halting saves four agent calls.
- **Fail-soft (retry):** a *transient* failure — throttle, 5xx, cold start → back off and retry
  (`run_agent_with_retry`). The work is still valid; the failure is temporary.

Swapping them is a bug: retrying a bad address burns calls on something that will never succeed; halting
on a throttle discards a run that a 2-second backoff would have saved. Same distinction as the
retry-vs-degrade discussion in [tool-design-patterns.md](tool-design-patterns.md).

### Extensibility comes from the uniform agent interface, not a clever orchestrator

Every agent is the same shape — model + prompt + tool — invoked the same way,
`run_agent_with_retry(builder, prompt)`. Adding a capability is: write one more builder, insert one line
into the workflow at the right phase. No redesign, because the orchestrator composes *uniform* pieces.
The sameness first noticed in lesson 1 ("only the tool changes") is precisely what buys the flexibility
the north-star paragraph asks for — flexibility is a property of the interface, not of orchestrator
cleverness.

### State

`workflow_state` dict: tools write, orchestrator reads — the capture principle again. The gate reads
`workflow_state["validation"]`, not the agent's prose, to decide whether to halt. Control flow that
branches on an agent's result must branch on its *structured* result, or a paraphrase silently reroutes
the workflow.

### Run evidence (lesson-4 run, 2026-09-26)

Three packages, all three primitives fired:

| Package | Route | Gate | Phase 2 (parallel) | Cond. | Total |
|---|---|---|---|---|---|
| PKG-001 | Domestic | 1.3s | **2.0s** (vs 1.8+1.6+1.7 = 5.1s summed) | 1.6s | 4.9s |
| PKG-002 | International | 1.3s | **1.7s** (vs 1.5×3 = 4.5s summed) | 1.3s | 4.3s |
| PKG-003 | **HALTED** (empty address) | 1.5s | 0.0 | 0.0 | 1.5s |

- **Fail-fast, measured:** PKG-003's empty address halted at the gate in 1.5s, skipping the four
  downstream agents (label, insurance, carrier, shipping). A full run was ~4.3–4.9s, so the gate saved
  ~3s and four Bedrock calls on a package that could never ship. That is the gate earning its place.
- **Parallel, measured:** Phase 2 wall-clock was ~2.0s / 1.7s against ~5.1s / 4.5s summed — ~2.5–2.6×.
  Not 3×, for the lesson-3 reason: bounded by the slowest of the three workers, not their average.
- **Capture principle, confirmed a third time:** the workers wrapped their replies in `<answer>…</answer>`
  and `<response>…</response>` tags (Nova's habit), yet the gate and summary were correct — because the
  orchestrator branched on `workflow_state["validation"]["status"]`, never the tagged prose. Control flow
  reading structured state shrugged off output the text channel would have choked on. (Phase 2 also
  reproduced the interleaved-`<thinking>` stdout garble from lesson 3 — shared unsynchronized stream.)

---

## Routing: layer cheap deterministic checks before the LLM

A router should be a **cascade**, cheapest and most-deterministic layer first, the model only for what the
cheap layers can't decide. Worked out on lesson 5 (telecom ticket router). Four layers, in priority order:

1. **Priority route** — keyword scan for the highest-stakes intent (cancellation → Retention). First, so
   "cancel my service and refund my bill" goes to Retention, not Billing. Order encodes business priority.
2. **Rule-based route** — keyword scan for the common intents (billing / technical).
3. **LLM classifier** — only for what layers 1–2 didn't match.
4. **Fallback** — low-confidence LLM result (< threshold) → GeneralSupport for human review.

This refines the lesson-4 "code vs LLM orchestrator" point: it isn't code *xor* LLM, it's code *then*
LLM. Cheap deterministic layers handle the bulk; the model earns its cost on the remainder.

### Run evidence (lesson-5 run, 2026-09-29) — 20 tickets, 100% correctly routed

| Method | Count | Share | ~Latency |
|---|---|---|---|
| priority | 2 | 10% | ~1.9s |
| rule | 13 | 65% | ~1.9s |
| llm | 5 | 25% | ~3.2s |
| fallback | 0 | 0% | — |

**75% of tickets never touched the classifier.** The LLM path (~3.2s) ran ~1.7× the latency of a
rule/priority path (~1.9s) — the ~1.3s delta is the classification round-trip — and only a quarter of
tickets paid it. That ratio is the entire argument for putting cheap gates in front of a model.

### The lesson the numbers taught, beyond "LLM handles the ambiguous"

**The LLM is the recall backstop for the rules' vocabulary gaps — not just an ambiguity handler.** Four
*technical* tickets (TKT-013, 017, 018, 019) did not match the technical keyword rule
(`outage|no signal|slow|dropped|disconnect|no service|tower`) and fell through to the classifier, which
labelled all four correctly at 0.90–0.95. They described technical problems in words the rule list didn't
enumerate. A keyword rule is an **allowlist**: it catches exactly the vocabulary you wrote down and
silently misses every paraphrase. So the cascade degrades gracefully — a rule miss isn't a
misroute, it's a fall-through to a layer that generalizes. Design consequence: size the LLM budget for
*rule misses + genuine ambiguity*, not ambiguity alone; here the header predicted 4 ambiguous but 5
tickets hit the model.

**The fallback path did not fire.** The one "nonsense" ticket (TKT-020) was confidently classified
"general" (0.90, above the 0.6 threshold) by the LLM, so it routed as `llm`→General, never tripping the
low-confidence fallback. The fallback is the safety net that this run did not need — meaning it is also
the layer this run did **not** exercise. Confidence-threshold behaviour stays unverified until a ticket
actually classifies below 0.6; don't assume the branch works because the run was green.

### How it maps to the project

Task 2's orchestrator is pure LLM routing (`route_to_*` tools) — every request pays the classification
cost. Lesson 5 is the upgrade path: put a keyword/priority pre-filter in front so unambiguous requests
(an order ID pattern, an explicit "cancel") skip the model. Whether that's worth it depends on the
rule-hit rate; at 75% deterministic here it plainly would be, but measure the traffic before adding the
layer — a pre-filter that almost never hits is just latency and code.

---

## Shared state: optimistic locking prevents lost writes, not stale-derived state

*Lesson 6, food-delivery shared state — four agents write the same DynamoDB order record.*

**The pattern.** Every agent writes through `update_order()`, which does optimistic locking: READ the
record and capture its `version` → MODIFY locally and bump the version → WRITE with
`ConditionExpression="version = :expected_ver"`. If another agent wrote in between, DynamoDB rejects the
write with `ConditionalCheckFailedException`; the loop logs a CONFLICT, backs off, and **re-reads** to get
the fresh version before retrying. The re-read is the whole point — reusing the stale version on retry, or
catching the exception without re-reading, silently loses updates. That's the first thing to check in any
implementation of this.

**What the run proved — and the trap.** Two scenarios on the same 4-agent pipeline:

| Scenario | Writes | Conflicts | Final `status` |
|---|---|---|---|
| Sequential (ORD-001) | 4, v0→v4 | 0 | `ready` ✅ |
| Concurrent (ORD-002) | 4, v0→v4 | 1, resolved | **`confirmed`, not `ready`** |

Both finished at v4 with every field written and **nothing lost** — locking did its job. But the
concurrent run ended in a *different logical state*. `StatusTracker` computes
`new_status = "ready" if (driver and total_price) else …`. Run last (sequential) it reads a fully
populated record → `ready`. Run in parallel it read the record *before* driver/price were written
(the output literally shows `Status for ORD-002: pending`), wrote a stale status, and a later agent
overwrote it with `confirmed`.

**The insight worth keeping:** optimistic locking guarantees **no lost writes** — it does **not**
guarantee **correct derived state**. A field computed *from other fields* is only right if those fields
already exist at read time, and locking says nothing about ordering. This is the counterexample to
"add optimistic locking and concurrency is solved."

**Two "is the LLM doing it right?" tells:**
- If an LLM claims a standalone state *service* removes the need for conditional writes because it
  "serializes access" — push back. A stateless service behind concurrent callers serializes nothing; it
  still needs the version-conditional write internally. A service *relocates* the locking, it doesn't
  remove it.
- If derived/aggregate fields must be correct, they can't be computed by an agent racing its inputs.
  Either serialize the deriving step (run it last, as a gate) or recompute the derived field from a
  consistent read after all writers finish — don't trust a parallel worker's snapshot.

**State recovery (new here, a saga preview).** When the restaurant rejects after driver+price were
already written, `recover_order()` resets those fields and cancels — and it routes *through*
`update_order()`, so the cleanup itself respects the version lock (recovery is just another concurrent
writer). An implementation that "cleans up" with a raw `put_item` bypasses the guard and can clobber a
concurrent write. This is a hand-rolled, single-table preview of a saga compensating transaction (L7).

---

## Saga pattern: compensation in reverse, gated by a barrier

*Lesson 7, e-commerce checkout — Inventory → Payment → Shipping, each with a compensating action.*

**The pattern.** A saga replaces one cross-service ACID transaction (impossible across services) with N
local commits, each paired with a **compensating action** that undoes it. The orchestrator runs steps
forward; on failure it runs the compensations for **already-completed** steps **in reverse order**. A
Python-code orchestrator holds this logic — the agents are thin single-tool executors (the conductor is
code, not an LLM; same principle as routing/orchestration).

**What the run proved:**

| Checkout | Fails at | Compensated | Barrier |
|---|---|---|---|
| CHK-001 | — | none, `completed` | n/a |
| CHK-002 | payment | inventory only (shipping never ran) | 1/1 |
| CHK-003 | shipping | **payment → inventory** (reverse of forward order) | 1/2 → 2/2 |

Three correctness properties, each with a tell for a wrong implementation:

1. **Reverse order.** CHK-003 cancelled payment *before* inventory — opposite of the forward sequence.
   Compensating forward-order is the bug (you'd try to undo dependencies before the things that depend
   on them).
2. **Only completed steps compensate.** The orchestrator filters `status == "completed"` before
   building the compensation list. CHK-002's shipping stayed `pending`, so it was never "cancelled." A
   naive version that compensates *all* steps would refund a charge that failed and cancel a shipment
   that never existed.
3. **Barrier gates resolution.** The saga flips to `failed` only when `compensations_completed ==
   compensations_needed`. This is an **invariant check, not a threshold** (`==`, not `>=`): an
   unexpected count — short *or* over (e.g. a double-increment) — leaves the saga `compensating` for an
   operator rather than declaring a clean outcome. For a money-moving saga, refusing to self-resolve on
   surprising bookkeeping is the safe default.

**The barrier is a distributed primitive on training wheels.** In this exercise compensations run
sequentially in one loop, so the loop *already* knows when they're all done — the counter is redundant
here. It exists to teach the mechanism you need once compensators run in parallel / on separate workers,
where no single loop sees them all finish and each must atomically signal "done." Implement it with
DynamoDB's `ADD` (atomic, lock-free) — a read-modify-write would lose concurrent increments.

**Where the course version is weak (noted, not fixed — matches the solution key):** the `needed > 0`
guard means a *first-step* failure (nothing completed, `needed == 0`) falls through to "stays
compensating for operator review" instead of resolving `failed` cleanly. The three scenarios never
trip it (inventory never fails). A correct version needs a third branch: `needed == 0` → resolve
`failed` immediately.

**Idempotency is the real-world gap.** Compensating actions must be safe to run twice (cancel twice ≠
double refund). The demo relies on the local state machine (`status` → `compensating` first), which is
the weakest form — the orchestrator can crash *between* the downstream call and the local state update.
Production needs end-to-end idempotency tokens the downstream service de-duplicates. This is the saga
version of L6's "locking ≠ correct derived state": the primitive is necessary but not sufficient.

**Capture principle, again.** CHK-003's payment agent wrapped its reply in a stray `<report>…</report>`
tag (Nova improvising). It didn't matter — the orchestrator read success/failure from `step["status"]`
in DynamoDB, never the prose. Regex-parsing the LLM text for "charged" would have broken on that tag.

---

## Multi-agent RAG: structured output ≠ grounded sections

*Lesson 8, clinical-literature RAG — two KB retrievers (parallel) → aggregate/dedup → synthesis.*

**The pattern.** Specialized retrievers (one Bedrock KB each) run in parallel via `ThreadPoolExecutor`;
results are merged, deduplicated, ranked, and top-K passed to a single synthesis agent (Nova **Pro**,
`tools=[]` so it answers only from the passages in its prompt). Retrieval is parallel (fast, independent);
synthesis is sequential (needs all evidence first). Standard multi-agent RAG shape.

**What the run proved:** Query 1/2 hit both KBs with 3-section cited output; Query 3 (drug KB forced
offline) degraded to guidelines-only, `Partial=YES`, with a disclaimer — graceful degradation: a narrower
*flagged* answer, not a failed query.

**The catch worth keeping — rigid schemas induce ungrounded backfill.** The synthesis prompt mandates
three sections (Drug Interactions / Clinical Guidelines / Integrated Recommendation). In Query 3 the drug
KB returned **zero** passages, so there was no drug-interaction evidence — yet the model filled the
**DRUG INTERACTIONS** section with *guideline* content cited to a guidelines doc (`[Clinical
Guidelines-1]`). The citation was honest; the section labeling was not. A schema that *demands* N sections
pressures the model to fill a section it has no evidence for, rather than admit the gap. For a
decision-support tool that is a real safety issue (a clinician reads mislabeled evidence).

- **Fix:** instruct the model to emit "No data retrieved for this section" when a section's evidence is
  absent — the schema should *permit* empty sections, not force them full. A bottom-of-output partial
  notice (which did fire) is not enough; grounding must hold per-section.
- **"Is the LLM doing it right?" tell:** check whether each section's citations actually come from the KB
  that section names. Cross-KB citations under a domain header = schema-induced backfill.

This is the RAG member of the recurring family: the headline primitive is necessary but not sufficient.
Optimistic locking ≠ correct derived state (L6); barrier `==` is an invariant not a threshold (L7);
structured schema ≠ grounded sections (L8).

**Dedup caveat.** The exercise dedups by `doc_id`, but the retriever assigns `doc_id = f"{kb_name}-{i+1}"`
— unique per KB, never colliding across KBs — so exact-id dedup removes nothing here; top-K ranking does
the trimming. Real near-duplicate suppression needs embedding cosine similarity, which this data never
exercises. Don't mistake "dedup ran" for "dedup did anything."

**Infra reality (logged in `rag-runbook.md`):** S3 Vectors KBs are MANAGED and need
`managedSearchConfiguration` in `retrieve()`, not `vectorSearchConfiguration` — the course's provided code
had the stale key and failed as a fake "tool configuration error" that returned 0 passages. Prove
retrieval from the CLI before blaming your code.

---

## Guardrails & evaluation: a declared policy is not an enforced policy

*Lesson 9, trading-compliance — Bedrock Guardrails + kill switch + rate limiter + LLM-as-judge.*

**The stack.** Pipeline = rate limit → input guardrail (`bedrock-runtime.apply_guardrail`) → compliance
agent → output guardrail → metrics/kill-switch → LLM-as-judge evaluation. Guardrail is a real
`AWS::Bedrock::Guardrail` (CloudFormation, so fully CLI — unlike KBs). `apply_guardrail` maps the API's
`GUARDRAIL_INTERVENED`/`NONE` to BLOCKED/ALLOWED; the exercise adds ANONYMIZED by inspecting per-entity
PII actions in `assessments` and failing toward BLOCKED on anything unclassified.

**Three findings, each a reusable "is it right?" check:**

1. **A declared policy ≠ an enforced one.** The guardrail config *declared* bank-account/CC/SSN blocks,
   email/phone anonymization, insider/recommendation/competitor topic denials, managed profanity, and
   violence filters. But adversarial testing showed Bedrock's **managed detectors silently miss**: a
   custom `ACCT-00123456789` format (not a recognized `US_BANK_ACCOUNT_NUMBER`), mild profanity
   (damn/crap not in the managed list), a worded threat ("threaten the exchange" didn't trip VIOLENCE).
   Reading the YAML would tell you it's covered; only firing adversarial inputs and reading the raw
   `assessments` tells you it isn't. **Verify guardrails adversarially, not by config review or pass-rate.**

2. **A volume circuit-breaker can mask content-filter holes.** First run scored 13/15 — but the kill
   switch (3 violations/60s) latched mid-adversarial-block and converted inputs it never inspected into
   `KILLED`, which the scorer counts as a pass. Raising the threshold so every input reached the
   guardrail dropped the score to a *truthful* 10/15, exposing 4 misses the kill switch had hidden. A
   high "blocked/killed" rate can mean "rate-limited," not "caught."

3. **Agents loop without a stop condition.** The compliance agent hit an **infinite tool-call loop** on a
   query whose answer wasn't in its rules table — the bland "consult a compliance officer" fallback left
   the model "unsatisfied," so it re-called the tool forever. Fix is belt-and-suspenders: a prompt
   constraint ("call the tool AT MOST ONCE, then answer") *and* a hard `max_iterations` cap. A tool whose
   result doesn't satisfy the model + no iteration ceiling = runaway. (Contrast the thin single-tool
   retrievers of L1–L8, which were implicitly capped by "do one thing.")

**LLM-as-judge worked well** — Sonnet scored the ALLOWED responses on compliance/relevance/safety/
completeness and was genuinely discerning (caught that a "margin requirements" answer never actually gave
the margin number). Aggregate completeness ~3.0 flagged the real weakness: the Nova-Lite compliance agent
gives thin answers. Note the judge needs the agent's *response* threaded into the results rows — a trivial
plumbing miss (dropping `agent_response`) silently disables the entire evaluation.

**Family note:** the recurring thread completes — the headline primitive is necessary but not sufficient.
Locking ≠ correct derived state (L6); barrier `==` is an invariant not a threshold (L7); structured schema
≠ grounded sections (L8); **declared guardrail ≠ enforced guardrail (L9)**.

**PII action direction (capstone, 2026-10-10): `BLOCK` is input-side, `ANONYMIZE` is output-side — test
accordingly or you'll report a false gap.** Fired adversarial PII at the deployed guardrail with
`apply_guardrail`: `CREDIT_DEBIT_CARD_NUMBER`/`US_SOCIAL_SECURITY_NUMBER` (action BLOCK) intervene on
`source='INPUT'` (reject the user's message before it reaches the model); `EMAIL`/`PHONE` (action
ANONYMIZE) return `NONE` on INPUT but on `source='OUTPUT'` return `GUARDRAIL_INTERVENED` with the masked
text (`"My email is {EMAIL}..."`). Makes sense: you block dangerous input *coming in*, but you mask PII in
what the model *emits going out*. A deployed Bedrock model-invocation guardrail runs on BOTH directions, so
coverage is end-to-end — but a one-direction `apply_guardrail` test will show a phantom "ANONYMIZE not
enforced" FAIL if you only test INPUT. Verify each PII entity on the source its action actually applies to.
The STANDARD-tier narrow pricing topic worked exactly as designed: "5 items at $29.99 with 10% off" →
ALLOW, "give me 30% off or I'll shop elsewhere" → BLOCK. (Still-real gap, matching L9: the managed
PROFANITY list misses mild words like "damn"/"hell" — add a `customWordsConfig` to catch those.)

---

## Framing: C2 = platform (real), C3 = patterns (simulated) — even L11 Gateway is a mock

A lens worth holding for the whole course and the capstone. The two nanodegree courses have opposite
philosophies, and it explains why cd14764 "doesn't use" the Gateway/CLI you learned in cd14763:

- **cd14763 = the platform course.** You used the *real* managed services — AgentCore Gateway, Identity,
  Memory, the `agentcore` CLI, runtime deployment. You learned *what the managed services are.*
- **cd14764 = the patterns course.** It deliberately **simulates** those services in local Python so the
  pattern is naked and unobscured by IAM/MCP wiring. A saga, a router, a gateway-registry — taught as
  plain code.

**Confirmed by grepping the whole course:**
- The new `agentcore` CLI and real Gateway service appear **nowhere in the lessons** (L1–L11 are local
  Python). **Correction (verified against the capstone + rubric):** the *capstone* is different — its
  `agent_orchestrator.py` ships a real dual-path Gateway (`LambdaGateway` local / `deploy_agentcore_gateway()`
  prod, line ~922) **and** an AgentCore Runtime deploy path. The rubric grades runtime+guardrails, memory,
  KBs, and observability — but **not** Gateway (it's scaffolded, ungraded). So "no CLI/no Gateway in
  cd14764" was too strong: true for lessons, false for the capstone. (Rubric Task 3 wording says
  `agentcore_cli.configure_runtime/deploy`; the on-disk starter uses `boto3 bedrock-agentcore` +
  `BedrockAgentCoreApp` — the rubric and starter may be slightly different versions.)
- **Even L11 "inter-agent communication via Gateway" is a simulation**: its `LambdaGateway` is a plain
  Python class doing registration → discovery → invocation, commented *"Production equivalent: Amazon
  Bedrock AgentCore Gateway (MCP protocol)."* Its `deploy_stack.py` only creates an IAM role via boto3
  ("No AWS CLI required").
- The capstone's TODOs are pure C3 patterns: specialized agents, `search_all_policies` (parallel RAG),
  `route_to_*_agent` (routing), guardrails — i.e. L1–L9.

**Why this is the strong position (not wasted learning):** C2 is what lets you *decode* C3's mocks. L11's
`LambdaGateway` reads as an abstract toy to someone who skipped C2; to someone who built the real Gateway
it's instantly recognizable as a stand-in for MCP Gateway. For judging correctness, you end up able to
read the **mock** (C3), the **raw boto3** (capstone `create_agent_runtime`), *and* the **managed CLI** (C2)
and see they're three views of the same deployment/communication reality — most people see only one layer.

**Residue:** the C3 capstone won't keep your real-Gateway/CLI muscles warm (those were graded in C2). The
runbooks (`gateway-runbook.md`, `cli-command-map.md`) are how you keep the real commands fresh for actual
projects even though C3 simulates past them.

---

## Gateway pattern: what's actually dynamic, and why MCP is the enabling contract

*Lesson 11, analytics-gateway — a `LambdaGateway` sim of AgentCore Gateway (registration → discovery → invocation).*

The gateway pattern has **three layers**, and a "tool gateway" claim is only as dynamic as its weakest one:

| Layer | In the L11 sim | Dynamic? |
|---|---|---|
| **Registry** (`gateway.targets`, `register_target()`) | add targets at runtime | ✅ |
| **Discovery** (`discover_tools()` → agent system prompt) | agent is *advertised* new tools on rebuild | ✅ |
| **Invocation binding** (one `@tool` per backend) | hardcoded in the agent builder | ❌ **static** |

So the sim's TODO 9 ("register a 4th tool, no code changes") **overclaims**: the new target is *advertised* but not *callable* — Strands can only invoke a tool that has a matching `@tool`. Registry + discovery are dynamic; the binding that makes a call possible is hand-wired.

**The limitation is a design choice, not inherent.** One generic proxy tool collapses the gap:
```python
@tool
def call_gateway_tool(tool_name: str, params: dict) -> str:
    return json.dumps(gateway.invoke_tool(tool_name, params), indent=2)
```
With that single binding, any registry entry is callable with zero new `@tool` code. Three ways to read the mechanism, in increasing dynamism: **naive sim** (per-tool bindings) → **generic proxy** (one binding, mostly dynamic) → **real MCP Gateway** (fully dynamic).

**Why the real one is fully dynamic: MCP is the contract.** The Model Context Protocol standardizes how a tool advertises its name/description/input-schema and how a call + result are framed. Once both sides speak it, **the schema *is* the binding** — the client needs no bespoke per-tool code to discover *or* invoke. The generic-proxy tool above is a hand-rolled, single-app imitation of that contract; MCP is the real, cross-vendor version. No standard interface → back to hardcoding a `@tool` per backend, which is exactly where the sim lands.

**"Is it really a dynamic gateway?" test:** can you add a backend and have the agent *invoke* it with no code change? If invocation still needs a new per-tool binding, it's a registry with a nice name, not a gateway. The thing that earns the word "gateway" is a standard invocation contract (MCP), not the registry.

**Ties to the C2-vs-C3 framing:** this is why cd14763 had you stand up the *real* Gateway — so that cd14764's `LambdaGateway` mock reads as "the registry/discovery half of MCP, minus the protocol-level invocation." The mock teaches the shape; MCP is what makes the shape work in production.

## An agent that reasons over dates/time must be *given* "now" — never let it guess

Capstone live run (2026-10-10): the Refund Agent decided a Premium 60-day return was eligible and
reasoned *"today, 2026-11-12 … 45 days elapsed."* The real date was 2026-10-10 (12 days elapsed). The
model **hallucinated the current date** — LLMs have no clock, so a prompt that asks "is this within N days
of the order date?" invites it to invent "today." The approval happened to be correct only because the
order was comfortably inside the window; **near a boundary (day 58 vs 62) the guessed date flips the
decision wrongly, with confident wrong arithmetic to match.**

This is a general class, not a one-off: any eligibility/expiry/SLA/age check where the model computes a
duration from "now" is unsafe if "now" is implicit. The decision looks right in a demo and fails silently
at the edges — the worst failure mode because tests with mid-window data never catch it.

**Fix (deterministic injection, not a smarter prompt):** supply the current date to the agent explicitly —
either a `get_current_date()` tool it must call, or (simpler) have the orchestrator stamp today's date into
the routed request / WorkflowState so the worker reads a fact instead of inventing one. Same principle as
[[#Don't make correctness depend on the LLM echoing structured data — capture it in the orchestrator]]:
anything load-bearing and knowable should come from code, not from the model's imagination. **"Is the LLM
doing it right?" tell:** if a date/number in the reasoning trace isn't traceable to a tool result or the
prompt, it was hallucinated — check the boundary cases, not the happy path.
