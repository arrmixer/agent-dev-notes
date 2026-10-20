# Framing: C2 = platform (real), C3 = patterns (simulated)

Narrative/course-meta from the Udacity nanodegree. Kept for the story (and because it explains a confusion
worth retelling in a blog post), not as decision-support.

The two courses have opposite philosophies, which explains why cd14764 "doesn't use" the Gateway/CLI from cd14763:

- **cd14763 = the platform course.** You used the *real* managed services — AgentCore Gateway, Identity,
  Memory, the `agentcore` CLI, runtime deployment. You learned *what the managed services are.*
- **cd14764 = the patterns course.** It deliberately **simulates** those services in local Python so the
  pattern is naked — unobscured by IAM/MCP wiring. A saga, a router, a gateway-registry — taught as plain code.

**Confirmed by grepping the course:**
- The new `agentcore` CLI and real Gateway service appear **nowhere in the L1–L11 lessons** (all local Python).
- **But the capstone is different:** its `agent_orchestrator.py` ships a real dual-path Gateway
  (`LambdaGateway` local / `deploy_agentcore_gateway()` prod) *and* an AgentCore Runtime deploy path. The
  rubric grades runtime+guardrails, memory, KBs, observability — but **not** Gateway (scaffolded, ungraded).
  So "no CLI/no Gateway in cd14764" is true for the lessons, false for the capstone.
- Even L11 "inter-agent communication via Gateway" is a **simulation** — `LambdaGateway` is a plain Python
  class doing registration → discovery → invocation, commented *"Production equivalent: AgentCore Gateway (MCP)."*

**Why this is the strong position (not wasted learning):** C2 is what lets you *decode* C3's mocks. L11's
`LambdaGateway` reads as an abstract toy to someone who skipped C2; to someone who built the real Gateway it's
instantly a stand-in for MCP Gateway. For judging correctness you end up able to read the **mock** (C3), the
**raw boto3** (capstone `create_agent_runtime`), *and* the **managed CLI** (C2) and see they're three views of
the same deployment/communication reality — most people see only one layer.

**Residue:** the C3 capstone won't keep your real-Gateway/CLI muscles warm (graded in C2). The reference docs
`gateway-runbook.md` / `cli-command-map.md` are how you keep the real commands fresh for actual projects.

(Distilled portable lesson lives in `principles/gateway-needs-an-invocation-contract.md`; this is the course story.)
