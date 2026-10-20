# Lesson exercise stacks — deploy, delete, and what they cost

The cd14764 lesson exercises ship a CloudFormation `infrastructure/stack.yaml` that provisions a
DynamoDB table (and sometimes little else). These are **throwaway per-exercise stacks**, distinct from
the capstone stack (`udacity-agentcore` + Gateway + runtime + role), which stays up until the review
clears. Deleting a lesson stack takes its table with it — no orphaned resources.

All commands assume the course account: `export AWS_PROFILE=default` and `--region us-east-1`.

## Deploy

```bash
aws cloudformation deploy --template-file infrastructure/stack.yaml \
    --stack-name <lesson-stack-name> --region us-east-1
```

The **table name is not the stack name.** Templates build it from a `ProjectName` parameter, e.g.
`TableName: !Sub ${ProjectName}-order-state` with `ProjectName` defaulting to something like
`lesson-06-shared-state`. So the table is `lesson-06-shared-state-order-state` regardless of what you
pass to `--stack-name`. Before running a script, confirm three places agree — code default,
`.env`, and the deployed table:

```bash
grep ORDER_STATE_TABLE food_delivery_state.py .env          # code + env
grep -E "ProjectName|TableName" infrastructure/stack.yaml   # what the stack builds
```

If you ever pass `--parameter-overrides ProjectName=foo`, set the matching var in `.env` too.

## Delete (cleanup)

```bash
aws cloudformation delete-stack --stack-name <lesson-stack-name> --region us-east-1
```

Stacks deleted so far: `lesson-05-exercise-routing`, `lesson-06-exercise-shared-state`.
Unsure of the exact name? `aws cloudformation list-stacks --region us-east-1 \
  --query 'StackSummaries[?StackStatus==\`CREATE_COMPLETE\`].StackName' --output text`.

## What idle stacks cost: essentially nothing (but verify the billing mode)

Every lesson table so far is **`PAY_PER_REQUEST` (on-demand)**. That's the reason leaving a lesson
stack up between sessions is safe:

| Cost component | On-demand (`PAY_PER_REQUEST`) | Provisioned (contrast) |
|---|---|---|
| Hourly capacity | **none** — you pay only per request | bills RCU/WCU 24/7, even idle |
| Storage | $0.25/GB-month — a few tiny rows ≈ $0 | same |
| Requests | only while a script runs | same |

So an idle on-demand table with a handful of rows costs ~$0. **Only the capstone stack matters for
cost vigilance.** But confirm the mode before assuming — a provisioned table *does* bill while idle:

```bash
aws dynamodb describe-table --table-name <table> --region us-east-1 \
  --query 'Table.[TableName,BillingModeSummary.BillingMode,ItemCount,TableSizeBytes]' --output text
```

## Gotchas learned the hard way

- **"The stack isn't in the console!"** — check the **region and account** before concluding it's gone.
  Both times this came up (L5), the console was pointed at the wrong region; the stack was fine. Same
  class of error as `AWS_PROFILE=ailab` (wrong account → AccessDenied that *looks* like a missing
  resource). See [[aws-profile-default-for-course-account]].
- **`ItemCount` / `TableSizeBytes` read `0` right after writes.** DynamoDB's counters lag ~6h; the rows
  are there. Query the table to confirm, don't trust the counter.

---

# CLI quick reference (verified in cd14764)

All assume `export AWS_PROFILE=default` and the lesson's region (`--region` shown per command; lessons
switch regions — trust the lesson's `.env`, not the previous lesson's default). Every command here was
run and confirmed working during the course.

## First, always: which account am I in?
An `AccessDenied` on *every* call (including read-only `describe-stacks`) is almost always the wrong
account or a dead lab session — not broken code or a deleted resource.
```bash
aws sts get-caller-identity --query 'Account' --output text    # want the voclabs course acct, not ailab
```
- `961341524503` = CapTech `ailab` SSO (WRONG for course work)
- course acct (e.g. `100139660723`) = reached via `--profile default`
- An explicit-deny policy named `voc-cancel-cred` on everything = **Vocareum lab session suspended** →
  re-launch the lab and reload credentials.

## CloudFormation — deploy, status, outputs, events, delete
```bash
# deploy (blocks until done)
aws cloudformation deploy --template-file infrastructure/stack.yaml \
  --stack-name <name> --region <r>

# status  (CREATE_IN_PROGRESS / CREATE_COMPLETE / ROLLBACK_* / "does not exist")
aws cloudformation describe-stacks --stack-name <name> --region <r> \
  --query 'Stacks[0].StackStatus' --output text

# pull a specific output value (e.g. a guardrail id) straight into your hand — no console
aws cloudformation describe-stacks --stack-name <name> --region <r> \
  --query "Stacks[0].Outputs[?OutputKey=='<OutputKey>'].OutputValue" --output text

# live events, newest first — use when a create/delete seems stuck
aws cloudformation describe-stack-events --stack-name <name> --region <r> \
  --query 'StackEvents[0:8].[ResourceType,ResourceStatus,ResourceStatusReason]' --output table

# list stacks by status (find the real name / confirm a prior delete finished)
aws cloudformation list-stacks --region <r> \
  --query "StackSummaries[?contains(StackName,'lesson')].[StackName,StackStatus]" --output table

# delete
aws cloudformation delete-stack --stack-name <name> --region <r>
```
Gotcha: a single trivial S3 bucket hanging >minutes in `CREATE_IN_PROGRESS` = a wedged lab account.
Probe S3 directly with `aws s3 mb s3://probe-$(date +%s) --region <r>`; if that works, the region's fine
and it's CloudFormation — if it hangs too, switch regions (us-east-1 ↔ us-west-2).

## Bedrock Guardrails (Lesson 9) — all CLI-able
```bash
# the guardrail is created by CloudFormation; grab its id from the stack output (see above).
# test what a guardrail ACTUALLY does to an input — read the raw assessments, don't trust config:
aws bedrock-runtime apply-guardrail --guardrail-identifier <GID> --guardrail-version DRAFT \
  --source INPUT --content '[{"text":{"text":"<text to test>"}}]' --region <r> --output json
```
- `action`: `GUARDRAIL_INTERVENED` (something fired) or `NONE` (clean).
- `assessments[].topicPolicy|contentPolicy|wordPolicy` = hard blocks;
  `assessments[].sensitiveInformationPolicy.piiEntities[].action` = `BLOCKED` or `ANONYMIZED` per entity.
- A *declared* policy ≠ an *enforced* one: managed detectors miss custom formats (`ACCT-#`), mild
  profanity, cleverly-worded threats. Always test adversarially. (See multi-agent-patterns.md §Guardrails.)

## Bedrock Knowledge Bases (Lesson 8) — mostly console, a few CLI checks
```bash
# list KBs and status (ACTIVE / DELETING)
aws bedrock-agent list-knowledge-bases --region <r> \
  --query 'knowledgeBaseSummaries[].[knowledgeBaseId,name,status]' --output table

# confirm a Sync actually ingested (don't run queries against a still-ingesting KB)
DS=$(aws bedrock-agent list-data-sources --knowledge-base-id <KB> --region <r> \
  --query 'dataSourceSummaries[0].dataSourceId' --output text)
aws bedrock-agent list-ingestion-jobs --knowledge-base-id <KB> --data-source-id $DS --region <r> \
  --query 'ingestionJobSummaries[0].[status,statistics.numberOfNewDocumentsIndexed]' --output text
# want: COMPLETE  <n-docs>

# prove retrieval from the CLI BEFORE blaming your code.
# S3 Vectors (MANAGED) KBs need managedSearchConfiguration, NOT vectorSearchConfiguration:
aws bedrock-agent-runtime retrieve --region <r> --knowledge-base-id <KB> \
  --retrieval-query '{"text":"<query>"}' \
  --retrieval-configuration '{"managedSearchConfiguration":{"numberOfResults":10}}'
```

## DynamoDB (Lessons 5–7) — billing mode & health
```bash
aws dynamodb describe-table --table-name <t> --region <r> \
  --query 'Table.[TableName,BillingModeSummary.BillingMode,TableStatus,ItemCount]' --output text
```
`PAY_PER_REQUEST` = ~$0 idle (no hourly charge). `ItemCount` lags ~6h — query the table to confirm rows,
don't trust the counter.

## S3 (seed + cleanup)
```bash
aws s3 ls s3://<bucket>/ --recursive --region <r>          # confirm seeded objects
aws s3 rm s3://<bucket> --recursive --region <r>           # empty before a stack delete (non-empty blocks it)
```

# Waiting on async AWS provisioning (create → READY)

Many AWS "create" calls are **asynchronous**: the API returns immediately with the resource in
`CREATING`, and it only becomes usable at `READY`/`ACTIVE` later. You must wait before the next step
(e.g. you can't version a guardrail, or invoke a runtime, until it's ready). There is **no push/callback
in the SDK** — the client learns the new state only by asking. So *something* polls; the question is
what, and how production-grade it needs to be. **Waiter coverage across AWS is uneven** — verify per
service with `client.waiter_names` (empty list = none).

**`async`/`await` is NOT an option here** — it only makes a poll non-blocking (frees the thread). Without
a push mechanism it's still polling underneath; it buys concurrency, not notification.

### Guardrails (`bedrock` service): no waiter — polling is the official answer
- `boto3.client('bedrock').waiter_names == []` (confirmed in boto3 docs: Client + Paginators only, no
  Waiters section). Status enum: `CREATING | UPDATING | VERSIONING | READY | FAILED | DELETING`.
- **You cannot borrow an AgentCore waiter for this.** Guardrails are a `bedrock`-owned resource; a waiter
  only works on a resource its own service owns. AgentCore *attaches* a guardrail but doesn't own its
  lifecycle — so `bedrock-agentcore-control` waiters don't apply to guardrails.
- AWS's **own documented remedy is a `get_guardrail` poll-until-READY loop** with `FAILED` handling via
  `statusReasons` / `failureRecommendations`. So a hand-rolled poll is the sanctioned pattern, not a
  shortcut. Must-have: wait for the DRAFT to be READY *before* `create_guardrail_version` or you hit
  `ConflictException` intermittently.

Decision guideline (options that solve the same guardrail problem — pick by project size/context):
| Approach | Use when |
|---|---|
| **1. Hand-rolled poll, fixed interval** | One-shot scripts, labs, the capstone. Simplest; AWS's doc example literally does this. What we shipped. |
| **2. Poll + exponential backoff + jitter** | The production default when no waiter exists. Fewer wasted calls, no thundering herd. Same code shape, smarter timing. |
| **4. Event-driven (EventBridge → Lambda/SNS/SQS, or Step Functions `waitForTaskToken`)** | Orchestrating *many* long-running provisioning steps, or when nothing should sit in a client loop. Needs standing infra — overkill for one resource in a deploy script. |

Rule of thumb: **one resource in a one-shot script → #1 (or #2 if it ships to prod); a provisioning
pipeline over many resources → #4.** #2 is the honest everyday default; a fixed `sleep` is only fine
because labs are one-shot.

**Prior art / the real production answer (researched 2026-10-10):**
- **First choice in prod is usually option 0: don't poll in app code — provision via IaC.** CloudFormation
  `AWS::Bedrock::Guardrail` (and CDK, which emits it) tracks the guardrail to a terminal state before
  `CREATE_COMPLETE` and exposes `Status`/`StatusReasons`/`FailureRecommendations`. Terraform does the same.
  So the most production-correct move, when you can, is to declare the guardrail in IaC and let the tool
  own the wait — not write imperative boto3 at all.
- **When you must do it imperatively (boto3 script / Lambda), emulate Terraform's `StateChangeConf`.** The
  provider's `waitGuardrailCreated` (`terraform-provider-aws/internal/service/bedrock/guardrail.go`) is the
  canonical shape: `Pending=[CREATING]`, `Target=[READY]`, refresh = `GetGuardrail`, **~5-min timeout**, and
  **`ContinuousTargetOccurence: 2`** — it must read `READY` *twice in a row* before trusting it (guards
  against an eventually-consistent status that briefly flaps to READY then changes). There is no boto3
  equivalent to call; you hand-roll these semantics (backoff + timeout + bail-on-FAILED + 2 consecutive
  READY reads). That's the production shape to copy — our fixed-interval single-read loop is the lab-grade
  version of it.
- Note: the official `aws-samples` guardrails notebook does NOT poll (versions immediately after create) —
  it only works because a human runs the cells with natural delay. That is exactly why an *automated*
  script needs the loop, and why skipping it yields intermittent `ConflictException`. No boto3 waiter
  exists and there's no open feature request for one; no EventBridge-based readiness approach was found.

### AgentCore Memory (`bedrock-agentcore-control`): a waiter EXISTS (option 3)
Unlike guardrails, the newer AgentCore control plane ships waiters:
`['memory_created', 'policy_active', 'policy_deleted', 'policy_engine_active', 'policy_engine_deleted',
'policy_generation_completed']`. The clean option is the waiter:
```python
ctrl = boto3.client('bedrock-agentcore-control', region_name=region)
ctrl.get_waiter('memory_created').wait(memoryId=mem_id)   # built-in backoff + max-attempts
```
**Caveat found in the cd14764 capstone:** the provided `configure_memory()` scaffold does NOT use the
waiter — it hand-rolls a `get_memory(memoryId=...)['memory']['status'] != 'ACTIVE'` poll loop (10s interval,
5-min cap, bail on FAILED). The student TODO is only the `create_memory(...)` call; the wait is provided
code you don't modify. So on this project we poll even though a waiter exists — swapping in the waiter would
be a legit cleanup/enhancement, not a rubric requirement.
Takeaway to remember: **guardrails have no waiter (always poll); AgentCore memory HAS one (`memory_created`)
— but check whether the surrounding code already waits before adding your own.** Always check
`waiter_names` first, then check what the existing code does.
