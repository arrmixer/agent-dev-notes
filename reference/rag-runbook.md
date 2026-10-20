# RAG runbook — Bedrock Knowledge Base on S3 Vectors

Console→CLI conversion of the M15 RAG exercise. Written stage by stage as each is verified.

Prereqs: `AWS_PROFILE=default` (sandbox `905473793005`), `us-east-1`.
Credential failure signatures: see `gateway-runbook.md`.

---

## ⚠️ Cost guardrail — S3 Vectors, never OpenSearch Serverless

The capstone specifies OpenSearch Serverless; **this exercise must not create one.** OpenSearch
Serverless is the only resource in this project that bills an idle capacity floor.

- **Do NOT use `agentcore add knowledge-base`.** It exposes no vector-store choice and its default name
  is `kb-quick-start-xxxxx` — the same "quick create" wording that provisions OpenSearch in the console.
- `aws opensearchserverless list-collections` is **denied** to the sandbox `voclabs` role, so the check
  has to go through Bedrock instead:

```bash
aws bedrock-agent list-knowledge-bases --region us-east-1 \
  --query 'knowledgeBaseSummaries[].[name,knowledgeBaseId]' --output text

aws bedrock-agent get-knowledge-base --region us-east-1 --knowledge-base-id <id> \
  --query 'knowledgeBase.storageConfiguration.type' --output text     # want S3_VECTORS
```

Run it after creating the KB. Cost Explorer lags ~24h, so it is not a safety net.

---

## What the console hides

"Vector store: Amazon S3 Vectors" in the console silently creates two resources. Via CLI they are
explicit, and S3 Vectors is a **separate service** (`aws s3vectors`) with its own bucket type — not a
regular S3 bucket.

Full chain: **six** resources plus a role.

| # | Resource | Command |
|---|---|---|
| 0 | docs bucket (regular S3) | `aws s3api create-bucket` + `aws s3 cp` |
| 1 | **vector bucket** | `aws s3vectors create-vector-bucket` |
| 2 | **vector index** | `aws s3vectors create-index` |
| 3 | Bedrock service role | `aws iam create-role` + `put-role-policy` |
| 4 | knowledge base | `aws bedrock-agent create-knowledge-base` (`s3VectorsConfiguration`) |
| 5 | data source | `aws bedrock-agent create-data-source` |
| 6 | ingestion | `aws bedrock-agent start-ingestion-job` |

`storageConfiguration.type` accepts: `OPENSEARCH_SERVERLESS`, `PINECONE`, `REDIS_ENTERPRISE_CLOUD`,
`RDS`, `MONGO_DB_ATLAS`, `NEPTUNE_ANALYTICS`, `S3_VECTORS`. `s3VectorsConfiguration` takes
`vectorBucketArn`, `indexArn`, `indexName`.

---

## Stage 0 — documents to S3 (done)

```bash
REGION=us-east-1
ACCT=905473793005
KB_BUCKET=wanderbot-kb-${ACCT}

aws s3api create-bucket --bucket $KB_BUCKET --region $REGION
aws s3api put-public-access-block --bucket $KB_BUCKET \
  --public-access-block-configuration \
  'BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true'

aws s3 cp _source/rag/datasets/travel_policies.txt     s3://${KB_BUCKET}/docs/
aws s3 cp _source/rag/datasets/destination_guides.txt  s3://${KB_BUCKET}/docs/
```

Verified: `travel_policies.txt` 9.0 KiB, `destination_guides.txt` 16.8 KiB.

**Deliberately no lifecycle policy** on this bucket — these are source documents, unlike
`wanderbot-browser-recordings-*` which has a 30-day expiry. Do not reuse that bucket for KB sources;
the documents would silently vanish after 30 days.

**Prefix `docs/` matters** — it scopes the data source, so unrelated objects in the bucket are not
ingested.

---

## 🔴 Stage 1 BLOCKED via CLI — `s3vectors` is denied to the sandbox role

**The CLI can do this; the sandbox role cannot.** Both calls fail with a plain `AccessDenied` (no
explicit deny, not `voc-cancel-cred` — a genuine permission gap):

```
s3vectors:CreateVectorBucket  -> AccessDenied "no identity-based policy allows"
s3vectors:ListVectorBuckets   -> AccessDenied "no identity-based policy allows"
```

Meanwhile `bedrock-agent:ListKnowledgeBases` **is** allowed. So the restriction is specific to the
`s3vectors` service, not to RAG generally.

**This is the first exercise where console→CLI conversion is blocked by sandbox permissions rather than
by CLI capability.** Verified that the tooling supports the full chain: `aws s3vectors` exists with
`create-vector-bucket` / `create-index`, and `create-knowledge-base` accepts `S3_VECTORS` with an
`s3VectorsConfiguration`. In an account with normal permissions the CLI path below would work as
written.

**Why the exercise says "use the console":** the KB wizard's *quick create a new vector store* option
provisions the vector bucket through a **Bedrock service role**, not the caller's role — sidestepping
the `s3vectors` permission the caller lacks.

**Adopted plan:** console for the vector store + KB; CLI for verification and everything after.

### The CLI commands, for an account that permits them

Kept because they are correct and were verified against the API shapes — only the sandbox blocks them.

```bash
VEC_BUCKET=wanderbot-kb-vectors-${ACCT}

aws s3vectors create-vector-bucket --vector-bucket-name $VEC_BUCKET --region $REGION

aws s3vectors create-index --region $REGION \
  --vector-bucket-name $VEC_BUCKET \
  --index-name wanderbot-kb-index \
  --data-type float32 \
  --dimension 1024 \
  --distance-metric cosine
```

### Notes that still apply, whichever route creates the index

**`dimension 1024`** must match the embedding model. Titan Text Embeddings v2 outputs 1024 by default
(also supports 512 and 256); the KB's `embeddingModelConfiguration.dimensions` is set to the same value
so they match by construction.

**`distance-metric cosine`**, not the skeleton's default `euclidean` — Titan v2 embeddings are
normalised, so cosine is the correct similarity measure for text.

⚠️ A dimension mismatch does **not** fail here. Everything succeeds until
`start-ingestion-job`, which then fails on mismatched vector dimensions. Check `get-index` output before
building anything on top.

---

## Stage 2 (console) — the KB is `MANAGED`, which hides the vector store

The console wizard produced a **`MANAGED`** knowledge base, not the `VECTOR` type the CLI path builds.
They are materially different:

| | CLI path (`create-knowledge-base`) | Console quick-create (what we got) |
|---|---|---|
| `knowledgeBaseConfiguration.type` | `VECTOR` | **`MANAGED`** |
| `storageConfiguration` | explicit — `s3VectorsConfiguration` with bucket + index ARNs | **absent entirely** |
| Vector store | you create and own it | AWS creates and manages it |
| Verifiable storage type via API | ✅ `storageConfiguration.type` | ❌ **not exposed** |

What `get-knowledge-base` returns for a MANAGED KB:

```json
{"type": "MANAGED",
 "managedKnowledgeBaseConfiguration": {
   "embeddingModelType": "CUSTOM",
   "embeddingModelArn": "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0",
   "embeddingModelConfiguration": {"bedrockEmbeddingModelConfiguration":
     {"dimensions": 1024, "embeddingDataType": "FLOAT32"}}}}
```

Titan v2 at 1024 dimensions FLOAT32 — matching what the CLI path would have set explicitly.

### ⚠️ Consequence for the cost guardrail

**The documented guardrail check does not work on a MANAGED KB.**
`storageConfiguration.type` is `null`, and both alternative routes are denied to the sandbox role
(`aoss:ListCollections`, `s3vectors:ListVectorBuckets`). So there is **no API-based way** to confirm the
vector store is S3 Vectors rather than OpenSearch Serverless.

Available verification, in order of directness:

1. **The console's KB detail page** — checked: it reports *Knowledge base type: **Managed vector
   store***. That confirms the MANAGED type but **does not name the underlying store**, so it restates
   the API rather than answering the question. No console route to the storage type was found.
   *(Inference, not proof: a "managed" tier that silently provisioned a resource billing an idle
   capacity floor would be an odd product decision, and S3 Vectors is the store that makes such a tier
   viable. Treat as likely, not verified.)*
2. **Cost Explorer**, filtered by service, ~24h later — an OpenSearch Serverless line item would be
   unmistakable. Lagging, so a detector rather than a guard.

For the capstone, this argues for the **CLI `VECTOR` path** over console quick-create: an explicit
`s3VectorsConfiguration` is self-documenting and auditable, where MANAGED is opaque.

## Stage 3 — verify retrieval BEFORE writing any code

Ingestion `COMPLETE`: 2 scanned, 2 indexed, 0 failed. Then prove retrieval independently of the agent,
so any later failure is unambiguously code:

```bash
aws bedrock-agent-runtime retrieve --region us-east-1 \
  --knowledge-base-id IWDUDLVM4Y \
  --retrieval-query '{"text":"What is the refund policy for Economy Flex?"}'
```

Result: **5 chunks**, top hit `score=1.0000` and it is the correct section —
`### 2.1 Cancellation Timeframes and Fees`, containing the 85% / 70% / 50% / no-refund table.

This is worth doing every time. It splits the exercise cleanly in two: infrastructure is proven, so a
wrong answer afterwards can only come from the tool wiring, the system prompt, or the model. Compare the
Browser exercise, where retrieval and agent behaviour could not be separated.

Also note `retrieve` returns **5 results by default** and each carries a `score` — useful for judging
whether a poor answer is bad retrieval or bad generation.

### 🔴 S3 Vectors (MANAGED) KBs need `managedSearchConfiguration`, not `vectorSearchConfiguration`

*Hit on cd14764 Lesson 8 (clinical-literature RAG).* The moment you pass an explicit
`retrievalConfiguration`, the key depends on the KB's vector store:

| KB vector store | KB type | retrieve() config key |
|---|---|---|
| OpenSearch Serverless | VECTOR | `vectorSearchConfiguration` |
| **S3 Vectors** (the cost-guardrail choice) | **MANAGED** | **`managedSearchConfiguration`** |

Using the wrong key fails with:

```
ValidationException: Incompatible configuration: vectorSearchConfiguration is not
supported for managed knowledge bases. Use managedSearchConfiguration instead.
```

Working call against an S3 Vectors KB:

```bash
aws bedrock-agent-runtime retrieve --region us-west-2 \
  --knowledge-base-id <KB_ID> \
  --retrieval-query '{"text":"warfarin drug interactions"}' \
  --retrieval-configuration '{"managedSearchConfiguration":{"numberOfResults":10}}'
```

The response shape is identical, so parsing code is unaffected — only the request key changes. This is
**API drift**: lesson/sample code written for OpenSearch-backed KBs hardcodes `vectorSearchConfiguration`
and silently predates S3 Vectors. The failure hides behind the agent too — the retriever LLM just reports
"a tool configuration error" and gives up, returning 0 passages, which looks like a code-TODO bug until
you reproduce the raw `retrieve()` call. **Always prove retrieval from the CLI first** (this stage) so a
0-passage result is unambiguously the API call, not your wiring. Watch for this on the capstone if it uses
S3 Vectors.

**Useful IDs for this KB:**

| | |
|---|---|
| Knowledge base ID | `IWDUDLVM4Y` |
| Data source ID | `9UDDNYXTDJ` |
| Docs bucket | `s3://wanderbot-kb-905473793005/docs/` |

## Full CLI chain — for a real account (NOT runnable in this sandbox)

Reference for real-world use. **Not executed here** — `s3vectors:*` is denied to the sandbox role, so
stages 1–2 fail and we used the console instead. Shapes verified against `--generate-cli-skeleton`;
the calls themselves are unexercised, so treat them as a starting point rather than proven.

This builds a **`VECTOR`** knowledge base with an explicit `s3VectorsConfiguration` — auditable, unlike
the console's opaque `MANAGED` type.

```bash
REGION=us-east-1
ACCT=$(aws sts get-caller-identity --query Account --output text)
KB_BUCKET=wanderbot-kb-${ACCT}
VEC_BUCKET=wanderbot-kb-vectors-${ACCT}
INDEX=wanderbot-kb-index
TITAN="arn:aws:bedrock:${REGION}::foundation-model/amazon.titan-embed-text-v2:0"
```

### 1–2. Vector bucket + index

```bash
aws s3vectors create-vector-bucket --vector-bucket-name $VEC_BUCKET --region $REGION

aws s3vectors create-index --region $REGION \
  --vector-bucket-name $VEC_BUCKET --index-name $INDEX \
  --data-type float32 --dimension 1024 --distance-metric cosine

VEC_ARN=$(aws s3vectors get-vector-bucket --region $REGION \
  --vector-bucket-name $VEC_BUCKET --query 'vectorBucket.vectorBucketArn' --output text)
IDX_ARN=$(aws s3vectors get-index --region $REGION \
  --vector-bucket-name $VEC_BUCKET --index-name $INDEX \
  --query 'index.indexArn' --output text)
```

### 3. IAM role Bedrock assumes

Three distinct permissions on three different resources — the likeliest place for this to go wrong.

```bash
ROLE_ARN=$(aws iam create-role --role-name wanderbot-kb-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{
    "Effect":"Allow","Principal":{"Service":"bedrock.amazonaws.com"},
    "Action":"sts:AssumeRole"}]}' \
  --query 'Role.Arn' --output text)

aws iam put-role-policy --role-name wanderbot-kb-role --policy-name kb-access \
  --policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[
      {\"Effect\":\"Allow\",\"Action\":[\"s3:GetObject\"],
       \"Resource\":\"arn:aws:s3:::${KB_BUCKET}/*\"},
      {\"Effect\":\"Allow\",\"Action\":[\"s3:ListBucket\"],
       \"Resource\":\"arn:aws:s3:::${KB_BUCKET}\"},
      {\"Effect\":\"Allow\",\"Action\":[\"s3vectors:*\"],
       \"Resource\":[\"${VEC_ARN}\",\"${IDX_ARN}\"]},
      {\"Effect\":\"Allow\",\"Action\":[\"bedrock:InvokeModel\"],
       \"Resource\":\"${TITAN}\"}
    ]}"

sleep 10   # IAM propagation
```

Note the bucket/object ARN split again — `GetObject` on `bucket/*`, `ListBucket` on `bucket`.
`bedrock:InvokeModel` on Titan is needed because **Bedrock embeds the documents on your behalf**.

### 4. Knowledge base

```bash
KB_ID=$(aws bedrock-agent create-knowledge-base --region $REGION \
  --name wanderbot-kb --role-arn "$ROLE_ARN" \
  --knowledge-base-configuration "{
    \"type\":\"VECTOR\",
    \"vectorKnowledgeBaseConfiguration\":{
      \"embeddingModelArn\":\"${TITAN}\",
      \"embeddingModelConfiguration\":{\"bedrockEmbeddingModelConfiguration\":{
        \"dimensions\":1024,\"embeddingDataType\":\"FLOAT32\"}}}}" \
  --storage-configuration "{
    \"type\":\"S3_VECTORS\",
    \"s3VectorsConfiguration\":{
      \"vectorBucketArn\":\"${VEC_ARN}\",
      \"indexArn\":\"${IDX_ARN}\",
      \"indexName\":\"${INDEX}\"}}" \
  --query 'knowledgeBase.knowledgeBaseId' --output text)
echo "KB_ID=$KB_ID"
```

`dimensions: 1024` here **must** equal the index's `--dimension`. Set both from one variable in anger.

### 5. Data source

```bash
DS_ID=$(aws bedrock-agent create-data-source --region $REGION \
  --knowledge-base-id "$KB_ID" --name wanderbot-docs \
  --data-source-configuration "{
    \"type\":\"S3\",
    \"s3Configuration\":{
      \"bucketArn\":\"arn:aws:s3:::${KB_BUCKET}\",
      \"inclusionPrefixes\":[\"docs/\"]}}" \
  --query 'dataSource.dataSourceId' --output text)
```

`inclusionPrefixes` is how the `docs/` scoping is expressed — without it the whole bucket is ingested.

### 6. Ingest, then verify

```bash
JOB=$(aws bedrock-agent start-ingestion-job --region $REGION \
  --knowledge-base-id "$KB_ID" --data-source-id "$DS_ID" \
  --query 'ingestionJob.ingestionJobId' --output text)

aws bedrock-agent get-ingestion-job --region $REGION \
  --knowledge-base-id "$KB_ID" --data-source-id "$DS_ID" --ingestion-job-id "$JOB" \
  --query 'ingestionJob.{status:status,stats:statistics}' --output json

# the cost guardrail — works on a VECTOR KB, unlike MANAGED
aws bedrock-agent get-knowledge-base --region $REGION --knowledge-base-id "$KB_ID" \
  --query 'knowledgeBase.storageConfiguration.type' --output text     # want S3_VECTORS

# and prove retrieval before touching agent code
aws bedrock-agent-runtime retrieve --region $REGION --knowledge-base-id "$KB_ID" \
  --retrieval-query '{"text":"What is the refund policy for Economy Flex?"}'
```

### Teardown

```bash
aws bedrock-agent delete-data-source --region $REGION --knowledge-base-id $KB_ID --data-source-id $DS_ID
aws bedrock-agent delete-knowledge-base --region $REGION --knowledge-base-id $KB_ID
aws s3vectors delete-index --region $REGION --vector-bucket-name $VEC_BUCKET --index-name $INDEX
aws s3vectors delete-vector-bucket --region $REGION --vector-bucket-name $VEC_BUCKET
aws iam delete-role-policy --role-name wanderbot-kb-role --policy-name kb-access
aws iam delete-role --role-name wanderbot-kb-role
aws s3 rm s3://${KB_BUCKET} --recursive && aws s3api delete-bucket --bucket ${KB_BUCKET}
```

**A console-created `MANAGED` KB has no vector bucket you can delete** — AWS owns it. Deleting the KB is
the whole teardown, which is convenient but means you cannot inspect or audit what it created.

## ✅ Cost guardrail RESOLVED — no OpenSearch, but a KMS key was found

Cost Explorer, 3-day window covering KB creation (2026-09-15 → 17):

```
  $0.1389  AWS Key Management Service          <- 90% of spend
  $0.0079  Amazon Bedrock
  $0.0050  Amazon Bedrock Managed Knowledge Base
  $0.0012  AmazonCloudWatch
  $0.0007  Amazon Simple Storage Service
  TOTAL    $0.1537
```

**No OpenSearch Serverless line item.** The guardrail is satisfied. Note also the dedicated
**`Amazon Bedrock Managed Knowledge Base`** category — a MANAGED KB bills as its own service rather
than passing OpenSearch charges through, which is itself evidence it is not OpenSearch-backed.

### ⚠️ The real standing charge came from CDK bootstrap, not AgentCore

KMS dominates the bill, and the cause is `alias/cdk-hnb659fds-assets-key` — a **customer-managed KMS
key created by CDK bootstrap**, i.e. by `agentcore deploy --yes` auto-bootstrapping. Customer-managed
keys cost ~**$1/month regardless of use**.

*(Correction: when reviewing the CDK template before the first deploy, this runbook's sibling
`gateway-runbook.md` recorded "nothing with a standing hourly charge." That check looked for OpenSearch
Serverless, NAT gateways, ECR images and provisioned concurrency — and missed that bootstrap itself
provisions a KMS key. The conclusion was right about the template and wrong about the deploy.)*

| | |
|---|---|
| Shape | fixed monthly, not usage-based — the only idle floor in this project |
| Magnitude | ~$1/month; trivial here, but it is the shape we spent effort avoiding |
| Source | CDK bootstrap, once per account+region |
| Removal | only by deleting the `CDKToolkit` stack, which breaks all future deploys |

**Generalisable lesson:** auditing the *application* template is not the same as auditing the *deploy*.
Bootstrap creates resources that never appear in your stack, and `--yes` performs it silently.

## Reading `aws ... help`

`aws <service> help` uses backspace overstriking for bold, so `S3_VECTORS` arrives as
`SS33__VVEECCTTOORRSS` and greps silently miss it. This produced two false conclusions here — "there is
no `s3vectors` service" and "S3 Vectors is not a supported storage type".

```bash
aws s3vectors help 2>&1 | col -b | grep ...          # strip overstriking
aws bedrock-agent create-knowledge-base --generate-cli-skeleton   # clean JSON for input shapes
```
