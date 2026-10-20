# Observability runbook — AgentCore traces + CloudWatch

Console→CLI conversion of the observability exercise (`_source/observability/`, `app/WanderBot_Observer/`).
Written stage by stage as each was verified 2026-09-20.

Prereqs: `AWS_PROFILE=default` (sandbox `905473793005`), `us-east-1`.
This is the one exercise that **cannot be done locally** — traces exist only for AgentCore-invoked runtimes.

---

## 🔴 Deploy blocked by slow uploads — the CLI pre-seed workaround

Two deploys failed at the **asset publish** step, both times on the one ~40 MB runtime bundle:

```
CDK_ASSETS_E5279: fail: @smithy/node-http-handler - the request socket did not respond
S3.AbortMultipartUpload  (repeated)
```

Small files (the 92 KB CloudFormation template) uploaded fine; the large sustained multipart transfer
died. Diagnosis with a raw timed upload:

```bash
dd if=/dev/urandom of=/tmp/t.bin bs=1m count=40
time aws s3 cp /tmp/t.bin s3://cdk-hnb659fds-assets-905473793005-us-east-1/DIAG-DELETE.bin --region us-east-1
# → succeeded, but ~2.5 MiB/s and DEGRADING (5 → 1 MiB/s)
```

**Root cause:** a slow link (later attributed to a VPN) + CDK's `@smithy` uploader has a socket
timeout that fires when a part transfers too slowly. The **AWS CLI transfer manager tolerates the same
slow link** (adaptive retries, per-part handling) where CDK's uploader gives up. So it is not a dead
network — it is slow-link + an over-aggressive CDK socket timeout.

### The workaround — pre-seed the asset, then deploy

CDK skips any asset already in the bucket (it does a `headObject` existence check). So upload the
staged bundle yourself with the CLI that works, then let CDK find it present:

```bash
H=<full-64-hex-hash>          # from the synthesized template, see below
aws s3 cp "agentcore/cdk/cdk.out/asset.$H.zip" \
  "s3://cdk-hnb659fds-assets-905473793005-us-east-1/$H.zip" \
  --region us-east-1 --cli-read-timeout 0     # 0 = no client-side read timeout, for a slow link
```

Find the hash for a given runtime from the synthesized template:

```bash
python3 -c "
import json,re
t=json.load(open('agentcore/cdk/cdk.out/AgentCore-AgentCoreCourse-default.template.json'))
for lid,r in t['Resources'].items():
    if r.get('Type')=='AWS::BedrockAgentCore::Runtime' and 'Observer' in lid:
        print(*set(re.findall(r'([0-9a-f]{64})\.zip', json.dumps(r))))
"
```

Then re-run `agentcore deploy`: publish finds all assets present, skips every upload, and proceeds
straight to the CloudFormation update. Confirmed: **0 socket failures, Observer `READY`**.

**For the capstone:** test the deploy with the VPN OFF first. If the VPN is the throttle, normal CDK
deploys work without this dance. The pre-seed is the fallback when you can't change the link.

⚠️ Only the big bundle needs pre-seeding — the other 9 assets were already in the bucket from prior
deploys, so this was a one-asset workaround. On a fresh account you would pre-seed each large bundle.

### Nothing is at risk while publish fails

Publish runs **before** the CloudFormation update, so a failed publish leaves the stack untouched (no
half-applied state). The partial multipart pieces are reaped by the bucket's
`AbortIncompleteMultipartUpload` lifecycle rule. Safe to `Ctrl+C` and retry at any point.

---

## The observability activation mechanism

Observability is **out of the box** — there is no flag on `agentcore deploy`, and we never disabled it.
Two layers:

- **Library:** `aws-opentelemetry-distro` ships in every scaffold's `pyproject.toml` (so the exercise's
  headline "add one dependency" is already done at scaffold time). It provides an
  `opentelemetry/instrumentation/auto_instrumentation/sitecustomize.py`.
- **Platform:** the managed AgentCore Runtime activates it — sets `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true`,
  the exporter endpoints, and puts the distro's sitecustomize on the path. Confirmed live in deployed
  runtime logs: `telemetry.auto.version: 0.20.0-aws`, `otel-rt-logs` stream, `aws.log.group.names`.

So on the CodeZip path: **you provide the library, the managed runtime activates and exports.** No
Dockerfile, no wrapper, no config. *(Inference not yet confirmed: whether the platform wraps the
entrypoint with `opentelemetry-instrument` or injects `PYTHONPATH` directly. Both give identical
results; `agentcore exec` into a container would settle it.)*

---

## Three CloudWatch surfaces — don't confuse them

| Surface | Log group(s) | Tied to agents? | Needs an invocation? |
|---|---|---|---|
| **GenAI Observability dashboard** | reads `aws/spans` (X-Ray, indexed) | yes — session-grouped | **yes** — empty until invoked |
| **Per-runtime logs** (`app.logger`) | `/aws/bedrock-agentcore/runtimes/<id>-DEFAULT` | yes — directly | no — exist on deploy |
| Lambda / model logs | `/aws/lambda/*`, `vendedlogs/bedrock` | adjacent, not the runtime | — |

The dashboard being empty while logs exist is the intended split: **metrics + raw logs are free on
deploy; traces (the dashboard) require an AgentCore invocation.** The README's hint — "Sessions view
empty → make sure you invoked via AgentCore" — is exactly this.

### Two prerequisites for the dashboard to populate

1. **CloudWatch Transaction Search must be `ACTIVE`** (indexes X-Ray spans into `aws/spans`, which the
   dashboard queries). Check: `aws xray get-trace-segment-destination` → want `Destination: CloudWatchLogs, Status: ACTIVE`.
2. **Propagation lag is real — 2-3 min minimum, up to ~10.** X-Ray gets spans fast (`get-trace-summaries`
   shows them within seconds); the indexed dashboard view trails. `agentcore traces list` prints the
   caveat itself: *"Traces may take 2-3 minutes to appear in CloudWatch."* Don't diagnose "empty
   dashboard" until several minutes have passed.

### ⚠️ Replaced runtimes orphan their log groups

`bedrock-agentcore/runtimes/*` had **14 log groups for 9 runtimes**. A runtime that is *replaced* (not
updated) gets a new runtime ID + new log group; the old one is **not** garbage-collected and bills for
stored logs indefinitely. The log-group count encodes deployment history — 5 of our runtimes were
replaced at some point (2 groups each), 4 never were (1 each). The live group is the one whose ID
matches `list-agent-runtimes`. Deleting the stale ones is safe.

---

## Reading traces from the CLI — `agentcore traces` beats raw X-Ray

The first-party path is cleaner than the X-Ray plumbing and gives a **superset** of the console.

```bash
agentcore traces list --runtime WanderBot_Observer          # session-grouped list + console deep-link
agentcore traces get <traceId> --runtime WanderBot_Observer # downloads full trace JSON
```

`traces list` shows Trace ID + Timestamp + **Session ID** — the session grouping the dashboard's
"Sessions" view shows, straight in the terminal. `traces get` writes `agentcore/.cli/traces/<runtime>-<id>.json`
carrying the full OpenTelemetry gen-AI semantic conventions:

| Attribute | Example |
|---|---|
| `gen_ai.request.model` | `us.amazon.nova-2-lite-v1:0` |
| `aws.genai.span_kind` | AGENT / LLM / TOOL |
| `gen_ai.agent.tools` | `["search_flights","search_hotels"]` |
| `aws.genai.token_count_total` | 1222, 1786 per LLM call; 5290 whole invocation |
| `gen_ai.server.time_to_first_token` | 444 ms |
| `gen_ai.server.request.duration` | 575 ms, 1085 ms |

**So the console→CLI conversion fully holds — the CLI is richer and machine-readable.**

### What the span tree reveals

```
AgentCoreCourse_WanderBot_Observer.DEFAULT    1682ms
  invoke_agent Strands Agents                 1677ms
    execute_event_loop_cycle                   583ms   ← cycle 1: model decides to call tool
      chat → BedrockRuntime                    579ms
      execute_tool search_hotels                 1ms   ← the tool itself is trivial
    execute_event_loop_cycle                  1092ms   ← cycle 2: model writes answer from tool result
      chat → BedrockRuntime                   1089ms
```

Two lessons: **the tool is 1 ms, the model is ~99% of latency** (optimize the model round-trips, not the
tool); and **the two `execute_event_loop_cycle` spans are the ReAct loop made visible** — decide, act,
observe, respond. Token total 5290, not the README's illustrative "512 in / 128 out", because the
tool's JSON result is fed back into the model on cycle 2.

### Raw X-Ray, if you need it

```bash
NOW=$(date -u +%s); START=$((NOW-1800))
aws xray get-trace-summaries --region us-east-1 --start-time $START --end-time $NOW
aws xray batch-get-traces --region us-east-1 --trace-ids <id>   # segments carry .Document JSON strings
```

⚠️ `aws ... --output text` with a scalar `--query` appends a stray `None`/`0` line here (a pager/botocore
quirk), which corrupts an ID captured into a shell var — `batch-get-traces` then rejects
`[id\nNone]` as >35 chars. Take the first line (`| head -1`) or use `agentcore traces` instead.

---

## Metric filter on `ERROR` + alarm (the project README's requirement)

The payoff for replacing `logging.basicConfig` with `app.logger` across all 10 exercises: deployed
runtimes emit structured JSON to their CloudWatch log group, so a **JSON metric filter** can count
error-level events without false positives.

```bash
LG="/aws/bedrock-agentcore/runtimes/AgentCoreCourse_WanderBot_Observer-qvXBr14rbF-DEFAULT"

aws logs put-metric-filter --region us-east-1 \
  --log-group-name "$LG" \
  --filter-name WanderBotObserverErrors \
  --filter-pattern '{ $.level = "ERROR" }' \
  --metric-transformations metricName=ObserverErrorCount,metricNamespace=WanderBot/Observability,metricValue=1,defaultValue=0

aws cloudwatch put-metric-alarm --region us-east-1 \
  --alarm-name WanderBotObserverErrorRate \
  --namespace WanderBot/Observability --metric-name ObserverErrorCount \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 5 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching
```

`--period 300 --threshold 5 --comparison-operator GreaterThanThreshold` is literally ">5 errors per
5 min". `notBreaching` because most 5-min windows have no invocations at all (no data ≠ problem).

### Why the JSON pattern, not `"ERROR"` — and prove it

A text pattern `"ERROR"` matches the word anywhere, including a user's *"I got an error booking"*
message. `{ $.level = "ERROR" }` keys on the structured field, which the message text can't reach.
Prove the discrimination without forcing a real failure or polluting the log group:

```bash
aws logs test-metric-filter --region us-east-1 \
  --filter-pattern '{ $.level = "ERROR" }' \
  --log-event-messages \
    '{"level":"ERROR","message":"Failed to retrieve","logger":"bedrock_agentcore.app"}' \
    '{"level":"INFO","message":"User: find flights","logger":"bedrock_agentcore.app"}' \
    '{"level":"INFO","message":"I got an error booking my flight","logger":"bedrock_agentcore.app"}' \
  --query 'matches[].eventNumber' --output text
# → 1   (only the real ERROR; the INFO line and the user-says-"error" line do NOT match)
```

`test-metric-filter` is the "prove it, don't infer it" tool for filters — it validates the pattern
against sample events with no live errors needed. A freshly created filter that has never matched
anything is otherwise unproven.

Initial alarm state is `INSUFFICIENT_DATA` — correct when no errors have occurred; with `notBreaching`
that is benign, not a fault.

### ⚠️ Capstone refinement (noted, not applied here)

Two residual false-positive sources for the capstone's alarm:

1. **OTEL SDK export errors** log at ERROR level (`Failed to export span batch`) — infrastructure
   noise, not your app failing. Scope the pattern to your own logger:
   `{ $.level = "ERROR" && $.logger = "bedrock_agentcore.app" }`.
2. **One metric filter attaches to one log group.** For many runtimes you need a filter per group, or a
   cross-group Logs Insights / subscription. The capstone's single runtime avoids this.

---

## Summary — the console→CLI conversion holds end to end

| Console step | CLI equivalent |
|---|---|
| GenAI dashboard → Sessions | `agentcore traces list --runtime X` |
| Click a trace → span tree | `agentcore traces get <id>` (JSON: model, tokens, TTFT, tool spans) |
| Read model/token/latency | gen-AI attributes in the downloaded trace |
| Metric filter + alarm (console wizard) | `aws logs put-metric-filter` + `aws cloudwatch put-metric-alarm` |
| — (no console equivalent) | `aws logs test-metric-filter` to prove the pattern |

`agentcore stop-session` (README's session break) does not exist — distinct `--session-id` values per
`agentcore invoke` replace it; the concept of "breaking" a sticky session is gone.
