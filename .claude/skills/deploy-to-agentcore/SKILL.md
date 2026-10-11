---
name: deploy-to-agentcore
description: >-
  Deploy a Strands / multi-agent Python system to Amazon Bedrock AgentCore Runtime using the agentcore npm
  CLI (stage -> configure -> deploy). Covers direct-code (CodeZip) packaging, the CDK stack it synthesizes,
  attaching a Bedrock Guardrail via environment variables (NOT a guardrailConfiguration block), serverProtocol
  HTTP (not MCP) for a standard runtime, reading back the deployed runtime ARN, and the stack->seed->KBs->deploy
  order of operations. Use when deploying/redeploying an AgentCore Runtime, debugging a deploy, or wiring the
  CLI into a project.
---

# Deploy to AgentCore Runtime (agentcore CLI)

AgentCore does not serialize Python objects — it packages your *code* and runs it as an HTTP server. Deploy
is a CLI pipeline, not a boto3 call. (Deploy with the CLI; **invoke** the deployed runtime with boto3
`invoke_agent_runtime` — different jobs.)

## Prerequisites
- Node 20+, `uv`, and the CLI: `npm install -g @aws/agentcore@<pinned-version>` (pin it — tested on 0.30.0).
- If an older Python `bedrock-agentcore-starter-toolkit` is installed, uninstall it (same `agentcore` command shadows).
- AWS creds in `~/.aws` (NOT `.env` — `load_dotenv()` lets stale `.env` keys override good ones → `InvalidClientTokenId`).

## The pipeline (wrap the CLI in a helper like `agentcore_cli.py`)
1. **Stage** (`stage_runtime_code`): copy your `src/*.py` + `config.py` + a `pyproject.toml` of runtime deps into
   `build/runtime/`, plus a marker file telling the entrypoint to serve HTTP. This is what gets zipped (`build: CodeZip`).
2. **Configure** (`configure_runtime`): write `agentcore/agentcore.json` — `network_mode='PUBLIC'`,
   `protocol='HTTP'`, `execution_role_arn=<your CF role>` (so the CLI won't mint its own), and **env vars**
   (region, KB IDs, guardrail id/version, log group). Empty values are skipped — blank KB IDs pre-Task-5 are fine.
3. **Deploy** (`deploy` → `agentcore deploy -y`): CLI downloads arm64/Py3.12 wheels with uv, zips, and
   creates/updates the runtime via a CDK stack `AgentCore-<project>-default`. **First run also bootstraps CDK
   (`CDKToolkit`) — 5–10 silent minutes; not a hang.** Re-running is safe (unchanged = left alone).
4. **Read back** (`deployed_runtime_arn`): from `agentcore/.cli/deployed-state.json`, fallback to a name lookup.

## Two things the reviewer confirmed are correct here
- **Guardrail attaches via env vars**, not a `guardrailConfiguration` block: pass `GUARDRAIL_ID`/`VERSION` as
  runtime env; the agent graph reads them and applies to every model. No per-agent change.
- **`serverProtocol: HTTP`**, not MCP. MCP is for tool gateways / A2A, not a standard runtime here.

## Order of operations (whole project)
infra CloudFormation stack → seed data → create KBs (console) → add KB IDs to `.env` → `deploy` (creates
guardrail, deploys runtime, memory, observability) → copy printed GUARDRAIL_ID/VERSION/RUNTIME_ARN into `.env`.

## Gotchas
- The `deploy` arg must reach the script — a dropped arg just prints the usage banner (not a bug).
- Deploy needs the infra stack up first (it reads role ARN / account from CF exports).
- Redeploy after creating KBs to inject their IDs; editing local `.env` alone doesn't update the deployed runtime.
