# Gateway + Lambda runbook

The exact sequence used for the AgentCore Gateway exercise. Supersedes the
gateway half of `scripts/gateway_setup.py` — see "Which approach won" below.

Prereqs: `AWS_PROFILE=default` (sandbox `905473793005`), region `us-east-1`,
CLI `@aws/agentcore` 0.27.1.

```bash
aws sts get-caller-identity --query Account --output text   # want 905473793005
```

Sandbox credentials expire, and there are **three** distinct failure signatures worth telling apart:

| Symptom | Meaning | Fix |
|---|---|---|
| `ExpiredToken` on every call | credentials aged out | refresh from the Udacity lab page |
| `AccessDenied` naming `arn:aws:iam::905473793005:policy/voc-cancel-cred` | **the lab session has ended** | restart the lab, then refresh credentials |
| `AccessDenied` naming any other policy | a real permission gap | fix the role |

The middle one is the misleading one: it is account-wide (Bedrock, AgentCore, S3 — everything) yet
`sts get-caller-identity` still returns the account, so it presents as a broken IAM setup rather than
an expired lab. Deployed resources are untouched and become reachable again on restart.

---

## Which approach won

Two paths were considered. **The `agentcore` CLI won for the Gateway; AWS CLI is
still required for the Lambdas.**

| Piece | Tool | Why |
|---|---|---|
| Lambda function + execution role | **AWS CLI** | Lambdas are not AgentCore resources. `agentcore` has no verb that creates one |
| Gateway + gateway target | **`agentcore add` + `deploy`** | CDK-managed, tracked in `agentcore.json`, removable with `agentcore remove` |

What settled it — all four questions the earlier draft raised:

1. **`add gateway-target` does accept an inline tool schema** — `--tool-schema-file <path>`.
2. **`add gateway` does expose `--authorizer-type NONE`**, which the exercise requires.
3. **CDK grants `lambda:InvokeFunction` correctly**, scoped to the exact function:
   ```
   Action:   ['lambda:InvokeFunction']
   Resource: ["arn:aws:lambda:us-east-1:905473793005:function:wanderbot-booking-tools",
              "arn:aws:lambda:us-east-1:905473793005:function:wanderbot-booking-tools:*"]
   ```
   So the hand-written gateway role in `scripts/gateway_setup.py` is unnecessary.
4. **No orphan problem** for the gateway, unlike the AgentCore Memory resource. It appears in
   `agentcore.json` and CloudFormation.

The Lambda *is* still an orphan — created outside CDK, so `agentcore remove` will not touch it.

### The deciding argument: validation happens before side effects

The first deploy attempt failed on an invalid target name — and **nothing was created**, because
CDK rejects at change-set validation, before any resource exists. There is a validation ladder,
and only the declarative path has the middle rungs:

| Stage | Catches |
|---|---|
| `agentcore validate` | project config schema |
| synth / `--dry-run` | CloudFormation generation |
| **change-set validation** | **AWS-side property rules — where the bad target name was caught** |
| deploy | real resource creation |
| invoking it | whether the wiring actually works |

`scripts/gateway_setup.py` has no equivalent of the third row. It would have created the gateway
successfully, then failed on `create_gateway_target`, leaving a half-built gateway and two IAM
roles to identify and delete by hand — with nothing recording that they existed.

Same benefit a type checker gives you over a runtime error: the failure moves from *discovered
while running, side effects already committed* to *rejected before anything happened*. See
`python-vs-swift-kotlin.md` entry 13 for the code-level version of the same ladder.

---

## 1. Lambda (AWS CLI)

```bash
cd ~/Developer/agentcore-course/AgentCoreCourse

aws iam create-role --role-name wanderbot-lambda-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name wanderbot-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

cd _source/gateway/lambda && zip -j /tmp/booking.zip booking_lambda.py && cd -

sleep 10   # IAM propagation; create-function fails with ValidationException if too early

aws lambda create-function --region us-east-1 \
  --function-name wanderbot-booking-tools \
  --runtime python3.12 \
  --role "arn:aws:iam::905473793005:role/wanderbot-lambda-role" \
  --handler booking_lambda.lambda_handler \
  --zip-file fileb:///tmp/booking.zip \
  --timeout 15 \
  --query 'FunctionArn' --output text
```

`-j` on `zip` matters: it strips directory paths so `booking_lambda.py` sits at the archive
root, which is what `--handler booking_lambda.lambda_handler` resolves against.

Updating after a code change:

```bash
cd _source/gateway/lambda && zip -j /tmp/booking.zip booking_lambda.py && cd -
aws lambda update-function-code --region us-east-1 \
  --function-name wanderbot-booking-tools --zip-file fileb:///tmp/booking.zip
```

### Second Lambda (hotel, phase 2 — behind API Gateway)

```bash
cd _source/gateway/lambda && zip -j /tmp/hotel.zip hotel_lambda.py && cd -
aws lambda create-function --region us-east-1 \
  --function-name wanderbot-hotel-tools \
  --runtime python3.12 \
  --role "arn:aws:iam::905473793005:role/wanderbot-lambda-role" \
  --handler hotel_lambda.lambda_handler \
  --zip-file fileb:///tmp/hotel.zip \
  --timeout 15 \
  --query 'FunctionArn' --output text
```

---

## 2. Gateway (agentcore CLI)

```bash
agentcore add gateway --name wanderbot-gateway --protocol-type MCP --authorizer-type NONE

agentcore add gateway-target --name booking-tools --gateway wanderbot-gateway \
  --type lambda-function-arn \
  --lambda-arn arn:aws:lambda:us-east-1:905473793005:function:wanderbot-booking-tools \
  --tool-schema-file _source/gateway/schema/booking_lambda.json
```

### ⚠️ Target names: dashes yes, underscores NO

The authoritative pattern comes from the CloudFormation validation error:

```
Property value [booking_tools] does not match pattern: ^([0-9a-zA-Z][-]?){1,100}$
```

Alphanumerics, each optionally followed by a dash. **Underscores are rejected.** So
`booking-tools` is valid and `booking_tools` fails at change-set validation — before anything
is created, so a rejected deploy leaves no partial state.

*(Corrected: an earlier version of this file claimed the opposite — "underscores only, no
dashes" — taken from a comment in the cd14762 demo script. That demo's own target names were
`weather` and `attractions`, containing neither, so its comment was never exercised. The
exercise README's suggested `wanderbot-booking-tools` would in fact have been accepted.)*

The MCP tool name the model sees becomes `booking-tools___get_booking`, which is what
`booking_lambda.py`'s `raw_tool.split("___", 1)[-1]` strips off — unaffected by which
separator the target name uses.

Both commands only edit local config. Verify before spending:

```bash
agentcore status          # gateway shows "Local only (1 target)" until deployed
agentcore deploy --dry-run
agentcore deploy --diff
```

`--dry-run` stops at `Check bootstrap status` on a fresh account. Synthesis has already
happened at that point, so the full resource list can be read without deploying:

```bash
python3 -c "
import json, collections
t = json.load(open('agentcore/cdk/cdk.out/AgentCore-AgentCoreCourse-default.template.json'))
c = collections.Counter(v['Type'] for v in t['Resources'].values())
[print(f'{n:3}  {k}') for k, n in sorted(c.items(), key=lambda x: -x[1])]
"
```

For this project that printed 20 resources: 6 IAM roles, 6 IAM policies, **5
`AWS::BedrockAgentCore::Runtime`**, 1 Gateway, 1 GatewayTarget, 1 CDK metadata. No
OpenSearch Serverless, no NAT gateway, no ECR, no provisioned concurrency — nothing with a
standing hourly charge.

---

## 3. Deploy

```bash
agentcore deploy --yes -v
```

`deploy` is **whole-project** — there is no per-runtime scoping (`--target` selects a
deployment target like dev/prod, not a runtime). So this deploys all five runtimes alongside
the gateway.

`--yes` also auto-bootstraps CDK, which a fresh account needs because the tool schema is
uploaded as an **S3 asset** rather than inlined:

```
ToolSchema: {"S3": {"Uri": "s3://cdk-hnb659fds-assets-905473793005-us-east-1/<hash>.json"}}
```

---

## Getting the Gateway endpoint

The URL is generated by AWS, so it changes if the gateway is recreated. Fetch it rather than
hardcoding:

```bash
GW=$(aws bedrock-agentcore-control list-gateways --region us-east-1 \
       --query 'items[0].gatewayId' --output text)

aws bedrock-agentcore-control get-gateway --region us-east-1 \
  --gateway-identifier "$GW" --query 'gatewayUrl' --output text
```

As deployed 2026-09-11:

```
https://agentcorecourse-wanderbot-gateway-2gzsgent0l.gateway.bedrock-agentcore.us-east-1.amazonaws.com/mcp
```

⚠️ **It already ends in `/mcp`.** The starter's placeholder is `"https://PLACEHOLDER/mcp"`, which
invites appending a second one. `/mcp/mcp` fails with a connection error that looks like a Gateway
problem rather than a typo.

Note also that CDK prefixed the name with the project: the gateway is
`AgentCoreCourse-wanderbot-gateway`, not `wanderbot-gateway` as passed to `add gateway`.

---

## Trap: a failed deploy leaves the stack in REVIEW_IN_PROGRESS

When change-set validation rejects a deploy, the stack is left behind as an empty shell and the
next attempt refuses to run:

```
✗ Check stack status
Stack "AgentCore-AgentCoreCourse-default" is currently in REVIEW_IN_PROGRESS state.
Please wait for the operation to complete before deploying.
```

**"Please wait" is wrong.** `REVIEW_IN_PROGRESS` on a never-deployed stack means "a change set
was created and never executed" — there is no operation in flight and it will sit there
indefinitely.

### Gate on StackStatus, NOT on resource count

```bash
S=AgentCore-AgentCoreCourse-default

# THE guard. Only REVIEW_IN_PROGRESS makes deletion safe.
aws cloudformation describe-stacks --region us-east-1 --stack-name $S \
  --query 'Stacks[0].StackStatus' --output text
```

⚠️ **`describe-stack-resources` is misleading here.** On a `REVIEW_IN_PROGRESS` stack it reports
the **planned** resources of the pending change set, not created ones. Observed live: the same
stack reported `0` after a change set failed validation, then `20` after a valid change set was
created — while having created nothing either time. So a resource count of 20 does *not* mean 20
resources exist, and a count of 0 does *not* prove the stack is safe to delete on other statuses.

`StackStatus` is the only reliable signal:

| StackStatus | Safe to `delete-stack`? |
|---|---|
| `REVIEW_IN_PROGRESS` | **yes** — nothing has been created |
| `CREATE_COMPLETE` / `UPDATE_COMPLETE` | **NO** — this destroys every runtime and gateway in it |
| `ROLLBACK_COMPLETE` | usually yes, but check `describe-stack-events` first |
| anything `_IN_PROGRESS` | no — an operation really is running; wait |

Then, only if the status is `REVIEW_IN_PROGRESS`:

```bash
aws cloudformation delete-stack --region us-east-1 --stack-name $S
aws cloudformation wait stack-delete-complete --region us-east-1 --stack-name $S
agentcore deploy --yes -v
```

Do not chain the check and the delete in one block — read the status, *then* decide. (Learned by
doing exactly that: the check printed 20, the delete ran anyway, and it was only harmless because
`REVIEW_IN_PROGRESS` meant those 20 were hypothetical.)

The `CDKToolkit` stack is separate and should be left alone — deleting the project stack does not
remove bootstrap, so subsequent deploys skip that step.

---

## Flag reference

Every flag used above, and why it matters.

### `agentcore deploy --yes -v`

| Flag | Long form | Meaning |
|---|---|---|
| `-v` | `--verbose` | Resource-level deployment events. Without it you get five spinner lines; with it, each of the 20 CloudFormation resources as it is created — which is what makes a failure diagnosable, because you learn *which* resource broke |
| `--yes` | `-y` | Auto-confirm prompts **and auto-bootstrap CDK**. This is why `--dry-run` halts on a fresh account: there is no CDK assets bucket yet |
| | `--dry-run` | Synthesize and stop. Note it stops *before* printing resources, so read the template directly (see above) |
| | `--diff` | CDK diff against what is already deployed — the useful one for a second or later deploy |
| | `--target <name>` | Deployment target (dev/prod style), **not** a runtime selector. There is no way to deploy one runtime |

Neither `--yes` nor `-v` changes *what* is deployed, only what you see and whether it pauses.

### `aws iam create-role`

- **`--assume-role-policy-document`** is the **trust policy**: who may *become* this role, not
  what it may do. `Principal.Service: lambda.amazonaws.com` lets the Lambda service assume it.
  Wrong here and the function cannot start at all — a confusing failure, because the symptom
  looks like a permissions problem in the code.
- **`attach-role-policy` vs `put-role-policy`**: `attach` links an existing managed policy by
  ARN; `put` inlines your own JSON document. `AWSLambdaBasicExecutionRole` grants only
  CloudWatch Logs writes, which is the entire permission set these Lambdas need.

### `aws lambda create-function`

| Flag | Meaning |
|---|---|
| `--handler booking_lambda.lambda_handler` | `<module>.<function>`. The module is the **filename without `.py`**, and it must sit at the zip root — which is what `zip -j` guarantees |
| `--zip-file fileb:///tmp/booking.zip` | **`fileb://` not `file://`** — the `b` is binary. `file://` sends it as text and corrupts the archive |
| `--runtime python3.12` | The proven demo used 3.12. Exercise headers say 3.14; untested whether that runtime exists yet |
| `--timeout 15` | Seconds before Lambda kills the invocation. Default is 3, tight for a cold start |
| `--query 'FunctionArn'` | **Client-side** JMESPath filter over the response. Prints one line instead of a JSON blob. Nothing is sent to AWS — the CLI applies it locally |
| `--output text` | Strips JSON quoting so the value is paste-ready |

### When the package outgrows `--zip-file` — stage it in S3

`--zip-file fileb://` embeds the archive in the API request, so it is capped. The published Lambda
quotas:

| Route | Limit |
|---|---|
| `--zip-file fileb://` (direct, what we used) | **50 MB** zipped |
| via S3 (`--code S3Bucket=…`) | **250 MB** unzipped, including layers |
| console inline editor | 3 MB |
| container image | 10 GB |

Our Lambdas are a couple of KB, so direct upload is right. Past 50 MB — which any function bundling
`boto3` extras, `pandas`, `playwright` or ML wheels will hit — the zip goes to S3 first and Lambda
pulls it:

```bash
aws s3 cp /tmp/big.zip s3://my-artifacts/lambda/big.zip --cli-read-timeout 0

aws lambda create-function --region us-east-1 \
  --function-name big-fn --runtime python3.12 \
  --role "arn:aws:iam::<acct>:role/<role>" \
  --handler app.lambda_handler \
  --code S3Bucket=my-artifacts,S3Key=lambda/big.zip \
  --timeout 30
```

Note `--code S3Bucket=…,S3Key=…` **replaces** `--zip-file`; they are mutually exclusive. To update
later: `aws lambda update-function-code --s3-bucket … --s3-key …`.

Two alternatives to a bigger zip, for shared dependencies: **layers** (5 per function, and they count
against the same 250 MB unzipped budget) and **container images** (10 GB, but you own a Dockerfile and
an ECR repo).

⚠️ **This is the same wall that broke two `agentcore deploy` runs** (see
`observability-runbook.md`): a large binary upload over a slow link. Different API, identical
failure shape. `--cli-read-timeout 0` above is there for the same reason — the AWS CLI's transfer
manager tolerates a slow link where other uploaders time out. If a 40 MB+ upload stalls, that flag
plus retry is the first thing to try.

`--query` and `--output text` work on **every** `aws` command and are the difference between
readable output and a wall of JSON. Worth making a habit.

### `zip -j`

`-j` means "junk the paths" — store `booking_lambda.py` at the archive root rather than
`_source/gateway/lambda/booking_lambda.py`. Without it, `--handler booking_lambda.lambda_handler`
cannot find the module and the function fails at import with
`Unable to import module 'booking_lambda'`.

### `agentcore add gateway` / `add gateway-target`

| Flag | Meaning |
|---|---|
| `--protocol-type MCP` | Only `MCP` is valid for a tool gateway (the other option is `None`) |
| `--authorizer-type NONE` | No authentication, as the exercise requires. Also valid: `AWS_IAM`, `CUSTOM_JWT`, `AUTHENTICATE_ONLY` |
| `--type lambda-function-arn` | Direct Lambda target. Others: `api-gateway`, `open-api-schema`, `smithy-model`, `mcp-server`, `http-runtime`, `connector`, `passthrough` |
| `--lambda-arn` | The function to invoke. Only for `--type lambda-function-arn` |
| `--tool-schema-file` | Local JSON file listing the tools. CDK uploads it as an S3 asset — which is what makes bootstrapping a prerequisite |
| `--rest-api-id` / `--stage` | For `--type api-gateway` instead of `--lambda-arn`. Needed for the hotel Lambda in phase 2 |

Both commands only edit local config. Nothing reaches AWS until `deploy`.

### `aws lambda invoke`

| Flag | Meaning |
|---|---|
| `--cli-binary-format raw-in-base64-out` | Lets `--payload` be plain JSON. Without it, AWS CLI v2 expects base64 and rejects the payload |
| `--client-context` | Base64-encoded JSON. The only way to set `bedrockAgentCoreToolName` by hand, which is how you exercise one specific tool without going through the Gateway |
| `/dev/stdout` | The output file positional argument. Writing to stdout prints the response instead of leaving it in a file |

---

## 🔴 Static vs dynamic tool schemas — the trade-off nobody mentions

The two gateway target types get their tool definitions in fundamentally different ways, and this
determines how you change a tool later. Neither the exercise READMEs nor the CLI help says so.

| | `lambda-function-arn` | `api-gateway` / `open-api-schema` |
|---|---|---|
| Tool source | **static** — `--tool-schema-file <path>`, stored as an S3 asset | **dynamic** — the Gateway exports your REST API as OpenAPI and derives tools |
| Change a tool | edit the JSON, `agentcore deploy` | change the API, redeploy the stage, then **re-sync the target** |
| `synchronize-gateway-targets` | ❌ **rejected** | ✅ required |
| Description quality | whatever you write in the JSON — full prose, per-parameter descriptions | **only as good as the API's documentation parts** |

From `aws bedrock-agentcore-control synchronize-gateway-targets help`:

> You cannot synchronize a target that has a static tool schema (`mcpToolSchema`) configured. Remove
> the static schema through an `UpdateGatewayTarget` call to enable dynamic tool synchronization.

So the two are mutually exclusive: a target is either statically described or dynamically
synchronized, never both.

### Consequence: dynamic targets produce far worse tool descriptions by default

Measured on this project. Same gateway, two targets:

**Static (`booking-tools`, from `booking_lambda.json`):**
```json
"description": "Retrieve full details for a single booking using its booking reference number.",
"booking_ref": {"description": "Booking reference number (e.g. BK-1001)", "type": "string"}
```

**Dynamic (`loyalty-api`, derived from the REST API), before any documentation work:**
```json
"description": "get_loyalty_account",          // just the operationName
"member_id":   {"type": "string"},             // no description
"basePath":    {"type": "string"}              // leaked export artifact, offered to the model
```

Three separate problems: the description is the bare operation name, parameters are undocumented, and
`basePath` appears as an input the model can try to fill. See
`evidence/loyalty-tool-schema-BEFORE.txt`.

### Fixing a dynamic target's descriptions takes THREE publish steps

This is the part that cost the most time. Each step is silently optional — skip one and the export
just omits the content, with no error anywhere.

| Step | Publishes | Skip it and… |
|---|---|---|
| 1. `create-model` + `update-method-response` | response schemas into the API definition | responses stay `Empty`; the model learns nothing about the return shape |
| 2. `create-documentation-part` | descriptions — **as a draft only** | nothing happens at all |
| 3. **`create-documentation-version --stage-name <stage>`** | snapshots the parts and binds them to the stage | **parts exist but never appear in any export** |
| 4. `create-deployment --stage-name <stage>` | the API definition to the stage | the stage serves the previous definition |
| 5. `synchronize-gateway-targets` | re-reads the export into the Gateway | the Gateway serves a cached spec indefinitely |

**Step 3 is the trap.** `create-documentation-part` succeeds and `get-documentation-parts` lists your
content, so it looks done. But `get-export` omits it — even with `--parameters
extensions=documentation` — until a documentation version exists and is bound to the stage. Verified:
before `create-documentation-version`, `summary` and `description` were both absent from the export;
immediately after, both were present with no other change.

**Step 5 is the other trap.** Compare timestamps — a target's `lastSynchronizedAt` will happily sit
half an hour behind your API changes:

```bash
aws bedrock-agentcore-control list-gateway-targets --region us-east-1 \
  --gateway-identifier <gw> \
  --query 'items[].{name:name,synced:lastSynchronizedAt}' --output table

aws bedrock-agentcore-control synchronize-gateway-targets --region us-east-1 \
  --gateway-identifier <gw> --target-id-list <target-id>
```

**Order matters:** deploy the stage *before* syncing the target, or the Gateway caches a spec that is
missing whatever the deployment would have published.

### Result: it works, and the payoff is large

Same tool, before and after the five steps above — `evidence/loyalty-tool-schema-{BEFORE,AFTER}.txt`:

```
before:  "description": "get_loyalty_account"
         "member_id": {"type": "string"}

after:   "description": "Retrieve the Horizon Rewards loyalty account for a customer: tier,
                         current points balance, the cash value of those points in USD, when the
                         points expire, and when they joined. Use this whenever a customer asks
                         about loyalty points, rewards balance, membership tier, or Horizon
                         Rewards status. Requires the member ID, e.g. hz-001."
         "member_id": {"description": "Horizon Rewards member ID in the form hz-NNN,
                                       e.g. hz-001. Lowercase.", "type": "string"}
```

**Write the METHOD description as tool-selection guidance, not as API documentation.** This string is
what the model reads when choosing a tool, so "Use this whenever a customer asks about loyalty points,
rewards balance, membership tier…" earns its place. Same principle as the `get_exchange_rate` docstring
pattern cited for rubric item 13 — the difference is that here it lives in API Gateway rather than in a
Python docstring.

**`basePath` survives.** It is an artifact of the OpenAPI export, offered to the model as an input with
no description, and documentation parts cannot remove it. Not harmful in testing, but it is a
parameter the model can try to fill for no reason — a real argument for static schemas where you
control the parameter list exactly.

### Verify at the level you actually changed

Three independent things can be stale, so check the one you touched rather than only the end result:

```bash
# 1. what API Gateway will export
aws apigateway get-export --region us-east-1 --rest-api-id <id> \
  --stage-name prod --export-type oas30 /tmp/oas.json
python3 -c "
import json; op = json.load(open('/tmp/oas.json'))['paths']['/loyalty/{member_id}']['get']
print('summary:', op.get('summary')); print('params:', op.get('parameters'))
print('responses:', list(op.get('responses', {}).keys()))"

# 2. what the Gateway is serving (see scripts/ or the dump_tools recipe)
# 3. what the model receives — list_tools_sync() output
```

## Secrets: how to get an API key into AgentCore Identity

`agentcore add credential` warns if you pass the value as a flag:

```
Warning: passing secrets as CLI flags exposes them to shell history and the process
table. Prefer interactive mode (run `agentcore add credential` with no secret flags)
for masked entry.
```

### Where the value actually goes

Verified on 0.27.1 after running `add credential`:

| Location | Contents | Git |
|---|---|---|
| `agentcore/agentcore.json` | `{"authorizerType": "ApiKeyCredentialProvider", "name": "..."}` — **name and type only** | tracked ✅ |
| `agentcore/.env.local` | `AGENTCORE_CREDENTIAL_<NAME_UPPER_SNAKE>=<value>` | **gitignored** by `agentcore/.gitignore` ✅ |
| `agentcore/cdk/cdk.out/` | synth output | gitignored by `agentcore/cdk/.gitignore` ✅ |
| AWS | **nothing until `deploy`** — `list-api-key-credential-providers` stays empty | — |

So the scaffold separates declaration from value correctly, and no secret can reach a commit. Two
caveats:

- **`.env.local` is plaintext at mode `0644`.** Interactive entry does not encrypt it, it only avoids
  broadcasting the value elsewhere.
- **The provider is created at `deploy`, not at `add`** — so it is CDK-managed and `agentcore remove`
  will clean it up. Unlike an AgentCore Memory resource created via the SDK.

### Ranking, worst to best

| Approach | Exposure | Resource tracked in `agentcore.json`? |
|---|---|---|
| `--api-key <value>` | `.env.local` **+ shell history + process table** | ✅ |
| **`agentcore add credential`, no flags** | `.env.local` only, masked entry | ✅ |
| AWS Console | `.env.local` not involved; value stays in AWS | ❌ **orphan** |
| Env var from a CI secret store | never on disk locally | ✅ *(inferred — see below)* |

**The Console is not the safe option it looks like.** It protects the value but creates a resource the
project does not know about, so `agentcore remove` cannot clean it up and `agentcore.json` stops
describing the real infrastructure. That is the same orphan problem as the memory resource, which
still requires a manual `delete_memory`.

**Interactive CLI is the right default for hands-on work**: declaration stays in version control, the
value stays out of history.

**For CI/CD**, the naming convention in `.env.local` —
`AGENTCORE_CREDENTIAL_WANDERBOT_LOYALTY_API_KEY` — strongly suggests exporting that variable from a
secret store works with no flag and no file. ⚠️ **Untested inference**, but too deliberate a
convention to be accidental. Verify before relying on it.

### If a key was already exposed via a flag

```bash
# remove the offending line from zsh history
history -d $(history | grep 'add credential' | tail -1 | awk '{print $1}')
```

For a sandbox key that expires with the lab this is cosmetic. For a real key, rotate it — history and
the process table are both readable by anything running as your user.

## Technique: reading `aws ... help` without the bold mangling

`aws <service> help` renders bold via backspace overstriking (the `man` convention), so `S3_VECTORS`
arrives as `SS33__VVEECCTTOORRSS` and greps silently miss it. This caused two wrong conclusions on this
project — "there is no `s3vectors` service" and "S3 Vectors is not a supported storage type", both false.

```bash
aws s3vectors help 2>&1 | col -b | grep -oE '^\s+o [a-z-]+'      # strip overstriking
aws bedrock-agent create-knowledge-base --generate-cli-skeleton   # clean JSON, better for input shapes
```

Prefer `--generate-cli-skeleton` when you want parameter *structure*; use `col -b` when you want prose
or enum lists. Never grep raw `help` output.

## Technique: capture IDs into shell variables, paste as one block

AWS resource creation is usually a chain where each call needs an ID from the previous one — a REST
API alone is ~11 calls threading six IDs. Copying those by hand between commands is the single
biggest source of mistakes, and a mistyped ID fails in a way that looks like a permissions problem.

Instead, capture each ID with `--query` + `--output text` into a shell variable and paste the whole
chain at once:

```bash
REGION=us-east-1
ACCT=905473793005

API_ID=$(aws apigateway create-rest-api --region $REGION \
  --name wanderbot-loyalty-api --query 'id' --output text)

ROOT_ID=$(aws apigateway get-resources --region $REGION --rest-api-id $API_ID \
  --query 'items[?path==`/`].id' --output text)

LOYALTY_ID=$(aws apigateway create-resource --region $REGION --rest-api-id $API_ID \
  --parent-id $ROOT_ID --path-part loyalty --query 'id' --output text)
```

Why this shape works:

- **`--query 'id' --output text`** yields a bare string with no JSON quoting, so `$(...)` captures
  something directly usable. Without `--output text` you get `"abc123"` *including the quotes*, which
  then gets embedded in the next ARN and fails obscurely.
- **`--query` is client-side**, so it costs nothing and works on every `aws` command.
- **`>/dev/null` on calls whose output you don't need** keeps the useful IDs visible amid the noise.
- **`echo` the captured values at the end** so they can be copied into later stages or a config file.

For values that need more than one field out of a single response, parse once rather than calling
twice — an API key returns both its id and its secret value, and the value is only returned at
creation:

```bash
KEY_JSON=$(aws apigateway create-api-key --region $REGION --name wanderbot-loyalty-key --enabled)
KEY_ID=$(echo "$KEY_JSON"    | python3 -c 'import json,sys; print(json.load(sys.stdin)["id"])')
KEY_VALUE=$(echo "$KEY_JSON" | python3 -c 'import json,sys; print(json.load(sys.stdin)["value"])')
```

### Caveats

- **Variables are per-shell.** A new tab or window loses them; re-run the assignments. This bites
  when a long provisioning session gets interrupted.
- **They are not persistence.** Write anything you will need tomorrow to a file — see
  `scripts/gateway_config.json` for the pattern. IDs recovered later cost a `list-*` call and a
  `--query` filter, which is fine, but secrets like an API key value **cannot be recovered** without
  `aws apigateway get-api-key --include-value`.
- **A failed call mid-block leaves later variables empty**, and the commands that follow will run
  with blank arguments. `set -e` at the top of the block stops at the first failure instead of
  cascading.

## S3 buckets for AgentCore: versioning vs lifecycle

The console recommends enabling versioning on any new bucket. For AgentCore, whether that is useful
depends on which direction the service uses the bucket.

| Bucket role | Versioning | Why |
|---|---|---|
| **Written to** — browser session recordings | **low value** | Each session gets a unique key, so overwrites essentially never happen. Versioning guards against a risk that is not present, and doubles storage for anything rewritten |
| **Read from** — `enterprisePolicies`, `certificates`, gateway `toolSchema` S3 assets | **genuinely useful** | These configs accept a `versionId`, so pinning an exact object version is possible — and versioning is the prerequisite for that |

The giveaway is in the `create-browser` input shape, where the same struct appears in both roles:

```json
"recording":          {"enabled": true, "s3Location": {"bucket": "", "prefix": "", "versionId": ""}},
"enterprisePolicies": [{"location": {"s3": {"bucket": "", "prefix": "", "versionId": ""}}, "type": "MANAGED"}]
```

A `versionId` on a **write destination** is meaningless — you cannot write to a specific version. It is
there only because AWS reuses one `S3Location` type for reads and writes. On `enterprisePolicies`, which
*reads* a policy document, pinning a version is exactly the right thing to do.

### What actually matters for a recordings bucket: a lifecycle policy

Session replays are large and accumulate forever. Unbounded growth on a bucket nobody opens is a far
likelier cost problem than a lost object:

```bash
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
  --lifecycle-configuration '{"Rules":[{
    "ID":"expire-recordings","Status":"Enabled",
    "Filter":{"Prefix":"recordings/"},
    "Expiration":{"Days":30}}]}'
```

Also worth knowing: **`create-browser` writes a `recordings/BrowserRecordingTestFile` immediately**, at
creation time, to validate the execution role's S3 write access. Useful free verification that the IAM
policy is right — but it means the bucket is not empty as a test baseline. Check for a *new* object keyed
by session ID, not for the bucket becoming non-empty.

## Inspection

```bash
agentcore status

aws bedrock-agentcore-control list-gateways --region us-east-1 \
  --query 'items[].{name:name,id:gatewayId,status:status}' --output table

aws bedrock-agentcore-control get-gateway --region us-east-1 \
  --gateway-identifier <gateway-id>

aws bedrock-agentcore-control list-gateway-targets --region us-east-1 \
  --gateway-identifier <gateway-id> \
  --query 'items[].{name:name,id:targetId,status:status}' --output table

aws lambda list-functions --region us-east-1 \
  --query "Functions[?starts_with(FunctionName,'wanderbot')].{name:FunctionName,runtime:Runtime}" \
  --output table
```

## Isolating "is the Lambda broken?" from "is the wiring broken?"

`booking_lambda.py` reads the tool name from
`context.client_context.custom.bedrockAgentCoreToolName`. A plain invoke can't set that, so it
lands in the `Unknown tool` branch — which still proves the function deploys and runs:

```bash
aws lambda invoke --region us-east-1 \
  --function-name wanderbot-booking-tools \
  --payload '{"booking_ref":"BK-1001"}' \
  --cli-binary-format raw-in-base64-out /dev/stdout
```

To exercise a specific tool, pass a base64 client context:

```bash
CTX=$(printf '{"custom":{"bedrockAgentCoreToolName":"booking-tools___get_booking"}}' | base64)
aws lambda invoke --region us-east-1 \
  --function-name wanderbot-booking-tools \
  --client-context "$CTX" \
  --payload '{"booking_ref":"BK-1001"}' \
  --cli-binary-format raw-in-base64-out /dev/stdout
```

Logs:

```bash
aws logs tail /aws/lambda/wanderbot-booking-tools --region us-east-1 --since 10m --follow
```

---

## Teardown

Two halves, because the two halves were created by different tools.

**CDK-managed (gateway, targets, runtimes)** — remove from config, then reconcile:

```bash
agentcore remove gateway-target --name booking-tools -y
agentcore remove gateway --name wanderbot-gateway -y
agentcore deploy --yes
```

Prefer scoped `remove` over `remove all`.

**Not CDK-managed (Lambdas, Lambda role)** — manual:

```bash
aws lambda delete-function --region us-east-1 --function-name wanderbot-booking-tools
aws lambda delete-function --region us-east-1 --function-name wanderbot-hotel-tools

aws iam detach-role-policy --role-name wanderbot-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name wanderbot-lambda-role
```

Verify nothing is left:

```bash
aws bedrock-agentcore-control list-gateways --region us-east-1
aws lambda list-functions --region us-east-1 \
  --query "Functions[?starts_with(FunctionName,'wanderbot')].FunctionName"
```
