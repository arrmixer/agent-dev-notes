# The journey — cd14764 (narrative)

The messy, human version of how the capstone got built — written to be mined for a blog post or
presentation. The clean architecture is in `retrospectives/capstone-cd14764.md`; this is the story,
dead-ends included. (Raw material — rough edges on purpose.)

## Phase 0 — the real goal

Coming from Swift/Kotlin, new to Python and AWS AgentCore. The stated goal was never "pass the course" —
it was *"learn the best practices and calling patterns well enough to tell whether an LLM like Claude is
doing agent work right or not."* That reframing changed everything downstream: every lesson got converted
hands-on, and the question at each step was "how would I *know* this is correct?" rather than "does it run?"

## Phase 1 — eleven lessons as reps

Worked all 11 lesson exercises end-to-end (smart-home, multi-model, parallelization, orchestration, routing,
shared state, saga, RAG, guardrails, deployment planning, gateway). The pattern insight that kept recurring:
**the headline primitive is necessary but not sufficient.** Locking doesn't give you correct derived state.
A barrier counter is an invariant, not a threshold. A structured output schema doesn't make the content
grounded. A *declared* guardrail isn't an *enforced* one. That thread became the spine of the reference docs.

Snags that taught the most:
- **L8:** us-east-1 CloudFormation hung ~20 minutes on a trivial S3 bucket — twice. Switched the lesson to
  us-west-2 and it worked instantly. Lesson: lab-account flakiness is real; don't assume your code is wrong.
  Also burned time on `vectorSearchConfiguration` vs `managedSearchConfiguration` — managed S3-Vectors KBs
  need the latter, and getting it wrong fails as a *fake* "tool config error" with zero passages.
- **L9:** the compliance agent fell into an infinite tool loop on a no-match query (fixed with a "call the
  tool at most once" prompt constraint). And the big one: the kill switch was *masking* guardrail misses —
  a KILLED request scored as a pass, inflating the score to a fake 13/15 when real coverage was 10/15.
- **L11:** `InvalidClientTokenId` right after "adding credentials" — stale keys in `.env` were overriding the
  good ones in `~/.aws` via `load_dotenv()`. Keep creds out of `.env`.

## Phase 2 — the capstone build

The CLI had quietly moved from a Python toolkit to the npm `agentcore` CLI; had to upgrade 0.27 → 0.30.
Built the 5-agent graph (orchestrator + inventory, refund, policy-with-3-parallel-retrievers, communication),
all wired through a DynamoDB WorkflowState with optimistic locking. Structural tests hit 40/40.

Order of operations mattered: stack → seed → KBs → deploy (the KBs need the stack's vector bucket + the
seeded policy docs). First `agentcore deploy` bootstraps CDK — 5–10 silent minutes that look like a hang but
aren't. One self-inflicted head-scratcher: `python src/agent_orchestrator.py deploy` printed the usage banner
— the `deploy` arg had been dropped; the code was fine.

## Phase 3 — the moment it ran

The first live run was the payoff. All six routing rules behaved: `initialize_session` first every time,
inventory→refund *in order*, three KB retrievers firing in parallel, math handled by the orchestrator with no
worker calls, and the communication agent always last — the orchestrator never wrote the reply itself.

And then the catch that justified the whole "learn to judge it" goal: the Refund Agent reasoned *"today is
2026-11-12… 45 days elapsed."* The real date was 2026-10-10. It **hallucinated the current date.** The
approval was correct only because the order sat comfortably inside the window — a boundary case would have
flipped wrong, with confident arithmetic to match. An LLM has no clock; give it "now," never let it guess.

## Phase 4 — pushing past the rubric

The tests only check *structure*, so I went looking for *behavior*. An adversarial guardrail harness fired
crafted inputs at the deployed guardrail and read the raw assessments: 9/10. The win — "5 items at $29.99
with 10% off" was *allowed* while "give me 30% off or I'll walk" was *blocked*, exactly the STANDARD-tier
carve-out we engineered. The gaps — managed profanity missed "damn/hell," and a PII "failure" turned out to
be me testing the wrong direction (ANONYMIZE is output-side, BLOCK is input-side). Then a CloudWatch
dashboard off real Bedrock metrics.

## Phase 5 — the review loop

Submitted. Bounced: missing `config.py`, missing KB console screenshots, missing a live-test screenshot, and
`.env` reported "missing" — the dotfile had been silently stripped from the zip (fix: ship a visible copy).
Resubmitted with all of it. Bounced again, smaller: Inventory/Refund were at temp 0.0, the spec said 0.1 (and
the Policy coordinator 0.1 vs 0.2). Fixed the temps, re-ran, 120/120, resubmitted. Passed.

The subtler lesson from that second review: it also *retracted* parts of the first review that had
mis-described my code (claimed an MCP protocol and an API call that weren't there). Trust your code over a
review that describes something that isn't in it.

## Phase 6 — turning it into an asset

Passing wasn't the end — it was the input. The reference docs became the seed for this repo: tier the
knowledge by portability (principles → rules, playbooks → skills), write the retrospective as a reusable
blueprint, and keep this journal as the story. The goal from Phase 0 held all the way through: not "I
finished a course," but "I can now look at an agent system and tell whether it's being done right."
