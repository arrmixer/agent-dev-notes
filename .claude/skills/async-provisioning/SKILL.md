---
name: async-provisioning
description: >-
  Wait correctly for an asynchronous AWS resource to become READY/ACTIVE (Bedrock Guardrail, AgentCore Memory/
  Runtime, etc.) when the create call returns while still CREATING. Covers the decision tiers (IaC-owned wait >
  boto3 waiter > hand-rolled poll with backoff > fixed sleep), checking client.waiter_names first, why guardrails
  have NO waiter (poll-until-READY is AWS's own documented pattern) and must be READY before versioning (else
  ConflictException), and why async/await is not a substitute for polling. Use when a create call returns before
  the resource is usable, or you hit a "not in a state that allows this operation" error.
---

# Waiting on async AWS provisioning

Many AWS create calls are asynchronous: they return with the resource `CREATING`, usable only at
`READY`/`ACTIVE`. There's no SDK push — something must poll. The question is *what*, and how production-grade.

## Decision tiers (prefer the top)
1. **Let IaC own the wait.** CloudFormation/CDK/Terraform block until the resource is terminal
   (`AWS::Bedrock::Guardrail` exposes `Status`/`StatusReasons`). If you can declare it in IaC, you don't poll at all.
2. **boto3 Waiter** — `client.get_waiter('<name>').wait(...)`: AWS-tuned polling (backoff, max-attempts). But
   **only exists where AWS defined one** — check `client.waiter_names` first (empty list = none).
3. **Hand-rolled poll with exponential backoff + jitter** — the production default when no waiter exists.
4. **Fixed `sleep` loop** — lab-grade only.

`async`/`await` is **not** a tier — it only makes a poll non-blocking; without a push mechanism you're still polling.

## Known coverage (verified)
- **Bedrock Guardrails: NO waiter** (`boto3.client('bedrock').waiter_names == []`). Status enum:
  `CREATING|UPDATING|VERSIONING|READY|FAILED|DELETING`. Poll `get_guardrail` until `READY`, bail on `FAILED`
  (surface `statusReasons`/`failureRecommendations`). This IS AWS's documented pattern. **Wait for the DRAFT to be
  READY before `create_guardrail_version`** or you hit `ConflictException: not in a state that allows this`.
- **AgentCore control plane HAS waiters**: `bedrock-agentcore-control` → `memory_created`, `policy_active`, etc.
  Use `get_waiter('memory_created').wait(memoryId=...)` — unless the surrounding code already polls (check first).

## Reference shape for a hand-rolled poll (emulate Terraform's StateChangeConf)
Pending=`[CREATING]`, Target=`[READY]`, refresh via the describe/get call, ~5-min timeout, and optionally
`ContinuousTargetOccurence: 2` (require READY twice in a row — survives an eventually-consistent flap).

## Rule of thumb
One resource in a one-shot script → poll (backoff). A provisioning pipeline over many resources → event-driven
(EventBridge / Step Functions `waitForTaskToken`). Always `client.waiter_names` first, then check if existing code
already waits before adding your own. (Principle: this is a platform instance of "prefer the maintained poller.")
