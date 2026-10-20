---
name: bedrock-kb-setup
description: >-
  Set up Amazon Bedrock Knowledge Bases backed by S3 Vectors for RAG, and retrieve from them in code. Covers
  the self-managed vs managed distinction (which decides vectorSearchConfiguration vs managedSearchConfiguration
  — getting it wrong fails as a fake "tool config error" with zero passages), Titan Embed v2 / 1024-dim / S3
  prefixes, the console wizard steps (use the EXISTING vector store, not quick-create), syncing the data source,
  and verifying retrieval from the CLI before blaming your code. Use when creating/debugging Bedrock KBs or RAG retrieval.
---

# Bedrock Knowledge Bases on S3 Vectors

S3 Vectors is the cheap managed vector store (no OpenSearch cluster). Titan Embed Text v2 does the embeddings.
`bedrock-agent-runtime.retrieve()` is the idiomatic grounding call.

## The gotcha that eats hours: self-managed vs managed
The `retrievalConfiguration` key depends on KB type:
- **Self-managed KB** (you point it at an existing S3 Vectors bucket/index) → `vectorSearchConfiguration`.
- **Managed KB** (Bedrock creates/owns the store) → `managedSearchConfiguration`.

Using the wrong one fails as a misleading **"tool config error" with 0 passages** — looks like your code, isn't.
Match the key to how the KB was created. (Capstone used self-managed → `vectorSearchConfiguration`; an earlier
lesson used managed → `managedSearchConfiguration`.)

## Console creation (per KB; one bucket, differ by prefix + index)
- Data source: your S3 **policy-docs** bucket, prefix per domain (`policies/returns/`, `.../shipping/`, `.../warranty/`).
- Embeddings: `amazon.titan-embed-text-v2:0`.
- Vector store: **S3 Vectors → "Use an existing vector bucket"** (the one your CloudFormation stack made) with
  the matching index (`returns-policy-index`, …). **Do NOT "Quick create a new vector store."**
- **Self-managed**, not a Managed KB.
- After creating each KB, click **Sync** on its data source (and confirm "Last sync time" / status Available).

## Vector store specifics & cost (portable)

- **Index config must match the embedding model exactly:** Titan Embed v2 → **1024 dimensions**, distance
  **cosine** (not euclidean), type **FLOAT32**. A dimension mismatch doesn't warn — it **fails at ingestion/sync**.
- **Why S3 Vectors over OpenSearch:** S3 Vectors has ~**zero idle cost** (object storage), whereas an
  OpenSearch Serverless collection bills continuously whether queried or not. For a course/low-traffic
  project that idle delta dominates — pick S3 Vectors.
- **Standing charges to expect even when idle:** the KMS key + CDK bootstrap leave a small (~$1/mo) baseline;
  not zero, but negligible next to OpenSearch.
- **Don't `agentcore add knowledge-base`** — that path creates/owns a *managed* store (and pulls you back
  into `managedSearchConfiguration`). Create the KB in the console against your existing S3 Vectors bucket.

## Order dependency
The KBs need the stack's vector bucket + indexes AND the seeded policy docs — so: **stack → seed → create KBs**.
Confirm the seed's S3 prefix matches the KB wizard prefix (`policies/returns/` etc.) or the KB ingests nothing.

## Verify retrieval BEFORE blaming code
From the CLI/a scratch script, call `retrieve()` on each KB id with a known query and confirm non-empty,
non-error passages. If the KBs return passages here but your agent gets none, the bug is in your wiring, not the KB.

## Parallel multi-agent RAG
One retriever sub-agent per KB, run concurrently (ThreadPoolExecutor for small N) → coordinator synthesizes.
See principle `parallelize-gate-decompose-aggregate`.
