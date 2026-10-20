---
name: agentcore-gateway
description: >-
  Stand up a REAL Amazon Bedrock AgentCore Gateway with Lambda-backed tool targets (the production
  counterpart to a local LambdaGateway simulation): an MCP endpoint that advertises tool schemas and routes
  invocations to Lambda. Covers the agentcore CLI (add gateway / add gateway-target / deploy) vs AWS CLI
  (Lambda + role) split, the static tool-schema-as-binding, the MCP tool-name prefix convention, and the
  gotchas (no underscores in target names, /mcp double-suffix, REVIEW_IN_PROGRESS stuck stacks, two-half
  teardown). Use when creating/debugging a real AgentCore Gateway or Lambda tool targets.
---

# Real AgentCore Gateway + Lambda targets

## What it is
An MCP endpoint (`.../mcp`) that fronts Lambda-backed tools. You register targets; the Gateway advertises
their tool schemas for model discovery and routes invocations. Production counterpart to a local sim.

## Division of tools
- **Lambda + its exec role → AWS CLI.** Lambdas aren't AgentCore resources; `agentcore` can't create them
  (so they're orphans — `agentcore remove` won't touch them).
- **Gateway + targets → `agentcore add` + `deploy`.** CDK-managed, tracked in `agentcore.json`, removable
  with `agentcore remove`. Change-set validation rejects bad input *before* any resource is created.

## Setup sequence
```bash
# 1. Lambda role + function (AWS CLI)
aws iam create-role --role-name <role> \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name <role> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
cd <lambda-src> && zip -j /tmp/fn.zip handler.py && cd -   # -j = flat, module at zip root
sleep 10   # IAM propagation; else ValidationException
aws lambda create-function --function-name <fn> --runtime python3.12 \
  --role arn:aws:iam::<acct>:role/<role> --handler handler.lambda_handler \
  --zip-file fileb:///tmp/fn.zip --timeout 15   # fileb:// (binary), not file://

# 2. Gateway + target (agentcore CLI — local config only until deploy)
agentcore add gateway --name <gw> --protocol-type MCP --authorizer-type NONE
agentcore add gateway-target --name booking-tools --gateway <gw> \
  --type lambda-function-arn --lambda-arn arn:aws:lambda:<region>:<acct>:function:<fn> \
  --tool-schema-file schema/<fn>.json

# 3. Verify then deploy (whole-project; deploys all runtimes too)
agentcore status; agentcore deploy --diff
agentcore deploy --yes -v   # --yes auto-bootstraps CDK (schema uploaded as S3 asset)
```

## Schema is the binding
For `lambda-function-arn` targets the tool schema is **static** — the JSON you pass is the contract (write
rich per-param descriptions). MCP tool name becomes `<target>___<tool>`; the Lambda strips the prefix via
`raw_tool.split("___",1)[-1]`.

## Gotchas
- **Target names: alphanumerics + dashes only, NO underscores** (`^([0-9a-zA-Z][-]?){1,100}$`). `booking_tools` fails validation.
- **Endpoint already ends in `/mcp`** — don't append another (`/mcp/mcp` fails, looks like a Gateway bug). Fetch it, don't hardcode (CDK prefixes the project name):
  ```bash
  GW=$(aws bedrock-agentcore-control list-gateways --query 'items[0].gatewayId' --output text)
  aws bedrock-agentcore-control get-gateway --gateway-identifier $GW --query gatewayUrl --output text
  ```
- **Failed deploy → stack stuck in `REVIEW_IN_PROGRESS`** (nothing's running; "please wait" is wrong). Gate on `StackStatus`, not resource count (which reports *planned* resources); only `REVIEW_IN_PROGRESS` is safe to `delete-stack`.
- **IAM:** CDK auto-grants `lambda:InvokeFunction` scoped to the function ARN — no hand-written gateway role. Lambda role needs only `AWSLambdaBasicExecutionRole`.
- **Teardown is two halves:** `agentcore remove gateway-target/gateway` + `deploy` for CDK resources; manual `delete-function`/`delete-role` for the orphaned Lambda.

## Debug: is it the Lambda or the wiring?
Plain invoke lands in the "unknown tool" branch (proves the function runs); set the tool name via base64 client context:
```bash
CTX=$(printf '{"custom":{"bedrockAgentCoreToolName":"booking-tools___get_booking"}}' | base64)
aws lambda invoke --function-name <fn> --client-context "$CTX" \
  --payload '{"booking_ref":"BK-1001"}' --cli-binary-format raw-in-base64-out /dev/stdout
aws logs tail /aws/lambda/<fn> --since 10m --follow
```

## Dynamic targets (API Gateway / OpenAPI / Smithy)

Beyond `lambda-function-arn`, a target can be `api-gateway` (front an existing REST API), `open-api-schema`
(any OpenAPI 3 service), or `smithy-model` (AWS-style Smithy service). Use these when the backend is already
an HTTP API rather than a Lambda you own.

**Schema source.** Lambda targets are *static* (you hand-write `--tool-schema-file`, stored as an S3 asset);
dynamic targets are *derived* (Gateway exports the API as OpenAPI and generates the tools). Mutually exclusive:

| | lambda | dynamic |
|---|---|---|
| `synchronize-gateway-targets` | rejected (static schema) | required |
| change a tool | edit JSON, `agentcore deploy` | change API, redeploy stage, re-sync |

**Why descriptions start bad.** A derived tool is only as good as the API's documentation parts — by default
you get `description: "<operationName>"`, parameters with no description, and a leaked `basePath` input. Write
the METHOD description as tool-selection guidance (ties to `tool-description-is-functional-not-docs`).

**Publish fix (3 doc steps + redeploy + sync):**
```bash
# 1. descriptions — DRAFT ONLY, invisible to export
aws apigateway create-documentation-part --rest-api-id <id> ...
# 2. THE TRAP: snapshot parts + bind to stage, or get-export omits them forever
aws apigateway create-documentation-version --rest-api-id <id> --stage-name <stage>
# 3. push definition to the stage, then re-sync the Gateway
aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>
aws bedrock-agentcore-control synchronize-gateway-targets --region <region> \
  --gateway-identifier <gw> --target-id-list <target-id>
```
Step 2 is the trap: `create-documentation-part` + `get-documentation-parts` both succeed (looks done), but
`get-export` omits content until a **version is bound to the stage**. **Order matters:** deploy the stage
*before* syncing, or Gateway caches an incomplete spec. Check `lastSynchronizedAt` — it lags silently.

**Gotchas.** `basePath` is an export artifact offered as an undescribed input that documentation parts can't
remove — a real argument for static (Lambda) schemas. Publish response schemas via `create-model` +
`update-method-response`, or returns stay `Empty`.

(Principle: `gateway-needs-an-invocation-contract`. Dropped detail — before/after payloads, the get-export
verification recipe — stays in `reference/gateway-runbook.md`.)
