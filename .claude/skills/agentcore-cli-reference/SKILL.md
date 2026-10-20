---
name: agentcore-cli-reference
description: >-
  Command reference for the @aws/agentcore npm CLI and the AWS CLI verification commands used around it:
  old Python toolkit → new CLI mapping, the agentcore lifecycle (validate/deploy/status/invoke/logs/traces),
  and AWS CLI checks for CloudFormation, DynamoDB, S3, Bedrock Guardrails, and Knowledge Bases. Leads with
  account-identity diagnostics (AccessDenied-on-everything = wrong account or dead lab session, not a deleted
  resource). Use when looking up an agentcore/AWS CLI command, diagnosing AccessDenied, or migrating from the
  old bedrock-agentcore-starter-toolkit.
---

# AgentCore / AWS CLI reference

Assumes `export AWS_PROFILE=default` and an explicit `--region` on AWS calls.

## Account identity — ALWAYS check first
An `AccessDenied` on *every* call (even read-only `describe-stacks`) is almost always the wrong account or a
dead lab session — not broken code or a deleted resource.
```bash
aws sts get-caller-identity --query 'Account' --output text
```
- An SSO/`ailab`-type account is WRONG for course work (fails as AccessDenied → looks like a missing resource).
- Course account reached via `--profile default` / `AWS_PROFILE=default`.
- Explicit-deny `voc-cancel-cred` on everything = Vocareum lab suspended → re-launch lab, reload creds.
- "Not in the console!" → check region AND account before concluding it's gone.

## Old Python toolkit → new npm CLI
`bedrock-agentcore-starter-toolkit` (pip, deprecated) → `@aws/agentcore` (npm). Both install an `agentcore`
binary; they CANNOT coexist — uninstall the old one.
| Old | New |
|---|---|
| `agentcore configure -e … --non-interactive` | `agentcore add agent --name X --framework Strands --model-provider Bedrock --protocol HTTP` (no `configure` verb, no `.bedrock_agentcore.yaml`) |
| `agentcore create` | `agentcore create --project-name X --name Y` |
| `pip install -r requirements.txt` | CLI-managed per-agent `.venv` (ships `pyproject.toml`+`uv.lock`) |
| *(none)* | `agentcore validate` → `Valid` |
| toolkit `dev` | `agentcore dev -r <Agent> --skip-deploy [--logs]` |
| `agentcore invoke '{"message":…}'` | `agentcore invoke "<prompt>"` (sends `{"prompt":…}`) |
| `agentcore destroy` | `agentcore remove agent --name X -y` then `agentcore deploy` |
| `… memory/gateway/identity create` | `agentcore add memory\|gateway\|credential` → `deploy` |

## agentcore lifecycle
```bash
agentcore --version          # pin per project (capstone used 0.30.0; cli-command-map was written at 0.27.1)
agentcore validate           # blocks, prints Valid
agentcore deploy             # BLOCKS; re-stages ALL runtimes (~50MB each); first run bootstraps CDK
agentcore deploy --diff      # NOT read-only — eagerly syncs credential providers
agentcore status
agentcore invoke --session-id <id> "<prompt>"   # session id needs 33+ chars
agentcore logs; agentcore traces list; agentcore traces get <id>
```

## CloudFormation
```bash
aws cloudformation deploy --template-file <tpl> --stack-name <n> --region <r> --capabilities CAPABILITY_NAMED_IAM   # blocks
aws cloudformation describe-stacks --stack-name <n> --region <r> --query 'Stacks[0].StackStatus' --output text
aws cloudformation describe-stacks --stack-name <n> --region <r> --query "Stacks[0].Outputs[?OutputKey=='<K>'].OutputValue" --output text
aws cloudformation describe-stack-events --stack-name <n> --region <r> --query 'StackEvents[0:8].[ResourceType,ResourceStatus,ResourceStatusReason]' --output table
aws cloudformation delete-stack --stack-name <n> --region <r>
```
Table name ≠ stack name (built from a `ProjectName` param). Trivial resource wedged in `CREATE_IN_PROGRESS`
= dead lab account; probe with `aws s3 mb s3://probe-$(date +%s)`.

## DynamoDB · S3
```bash
aws dynamodb describe-table --table-name <t> --region <r> --query 'Table.[TableName,BillingModeSummary.BillingMode,TableStatus,ItemCount]' --output text
# PAY_PER_REQUEST = ~$0 idle. ItemCount/TableSizeBytes lag ~6h — query rows, don't trust the counter.
aws s3 ls s3://<bucket>/ --recursive --region <r>
aws s3 rm s3://<bucket> --recursive --region <r>   # empty before stack delete (non-empty blocks it)
```

## Bedrock Guardrails · Knowledge Bases
```bash
aws bedrock-runtime apply-guardrail --guardrail-identifier <GID> --guardrail-version DRAFT \
  --source INPUT --content '[{"text":{"text":"<text>"}}]' --region <r> --output json   # action: GUARDRAIL_INTERVENED | NONE

aws bedrock-agent list-knowledge-bases --region <r> --query 'knowledgeBaseSummaries[].[knowledgeBaseId,name,status]' --output table
DS=$(aws bedrock-agent list-data-sources --knowledge-base-id <KB> --region <r> --query 'dataSourceSummaries[0].dataSourceId' --output text)
aws bedrock-agent list-ingestion-jobs --knowledge-base-id <KB> --data-source-id $DS --region <r> \
  --query 'ingestionJobSummaries[0].[status,statistics.numberOfNewDocumentsIndexed]' --output text   # want: COMPLETE <n>
# retrieve test — self-managed KB → vectorSearchConfiguration; MANAGED S3-Vectors KB → managedSearchConfiguration:
aws bedrock-agent-runtime retrieve --region <r> --knowledge-base-id <KB> \
  --retrieval-query '{"text":"<query>"}' --retrieval-configuration '{"vectorSearchConfiguration":{"numberOfResults":10}}'
```

## Gotchas
- `deploy` / CFN `deploy` / `validate` BLOCK; most AWS `create-*` calls are ASYNC (return CREATING → poll to READY/ACTIVE).
- `deploy --diff` / `--dry-run` are NOT read-only — they sync credential providers.
- Add `.bedrock_agentcore/` and `agentcore/cdk/cdk.out/` to `.gitignore` BEFORE first commit (old toolkit's `dependencies.zip` = 102MB > GitHub's 100MB limit).
- Never give an identity field a plausible fallback — fail closed or use an obvious sentinel. (See `declared-guardrail-not-enforced`, `async-provisioning`.)

Deeper flag/table detail (`add agent` flags, `dev` local modes, two-service `bedrock-agentcore` vs `-control` split):
`reference/cli-command-map.md`.
