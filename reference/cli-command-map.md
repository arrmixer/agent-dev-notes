# Old CLI → new CLI command map

`bedrock-agentcore-starter-toolkit` (deprecated, Python/pip) → `@aws/agentcore` (Node/npm).
Both install an executable named `agentcore`; they cannot coexist.

Rows marked **verified** were run against `@aws/agentcore` 0.27.1 on this machine.

## Lifecycle

| Task | Old | New | Notes |
|---|---|---|---|
| Configure an existing entry point | `agentcore configure -e starter.py -n WanderBot -dt container -rf requirements.txt --disable-memory -er $ER -ecr $ECR_URI --non-interactive` | `agentcore add agent --name WanderBot_CodeInterpreter --type create --build CodeZip --language Python --framework Strands --model-provider Bedrock --memory none --protocol HTTP` | **verified.** No `configure` verb, no `.bedrock_agentcore.yaml`. `-dt container` → `--build Container`; `--disable-memory` → `--memory none`. `-er`/`-ecr` have no equivalent — roles and image repos are CDK-managed |
| Create a project | `agentcore create` | `agentcore create --project-name X --name Y …` | **verified.** Makes its own subdirectory. `--project-name`: ≤23 chars, alphanumeric, leading letter |
| Install deps | `pip install -r requirements.txt` | Scaffold ships `pyproject.toml` + `uv.lock`; the CLI manages a per-agent `.venv` under `app/<Agent>/.venv` | **verified.** Don't hand-create a venv |
| Validate config | *(none)* | `agentcore validate` | **verified** → `Valid` |
| Run locally | *(deploy first, or toolkit `dev`)* | `agentcore dev -r <Agent> --logs --skip-deploy` | **verified.** uvicorn + hot reload. Port `8080` offset by runtime index, so agent #2 is `:8081`. `-r` required once multiple runtimes exist |
| Invoke locally | *(n/a)* | `agentcore dev -r <Agent> "<prompt>" --stream` | **verified.** Needs a dev server already running |
| Deploy | `agentcore deploy` | `agentcore deploy` | Same verb, CDK desired-state engine |
| Preview a deploy | *(none)* | `agentcore deploy --diff` / `--dry-run` | **verified — NOT read-only.** See below |
| Tear down | `agentcore destroy` | `agentcore remove <resource> …` then `agentcore deploy` | Prefer scoped `remove agent --name X -y` over `remove all` |

See **Local development** at the end of this file — `dev` has three launch modes, and shutdown has a gotcha.

## `agentcore add agent` flags

Run five times now, so worth spelling out.

| Flag | Values | Notes |
|---|---|---|
| `--name` | alphanumeric + underscores, max 48 | Ours: `WanderBot_<Exercise>` |
| `--type` | **`create`** \| `byo` \| `import` | **Defaults to `create`**, so passing it is redundant. See below |
| `--build` | **`CodeZip`** \| `Container` | Defaults to CodeZip. Container needs Docker + an image; nothing in these exercises does |
| `--language` | `Python` (create) \| Python/TypeScript/Other (byo) | `create` is Python-only |
| `--framework` | `Strands`, `LangChain_LangGraph`, `GoogleADK`, `OpenAIAgents`, `VercelAI` | |
| `--model-provider` | `Bedrock` | |
| `--memory` | `none` \| `shortTerm` \| `longAndShortTerm` | A **capability level, not a reference.** Declares a *new* CDK-managed memory on `deploy` — **even when the project already has one**, giving you a duplicate. Use `none` when provisioning out-of-band; see *AgentCore Memory* below |
| `--protocol` | `HTTP` \| `MCP` \| `A2A` \| `AGUI` | |

### What `--type` actually selects

| Value | Behaviour | Extra flags |
|---|---|---|
| **`create`** | Scaffolds a fresh agent: venv, `pyproject.toml`, `uv.lock`, and a reference `main.py` that is more complete than any doc example. **What every conversion here uses** | — |
| `byo` | "Bring your own" — registers *existing* code rather than scaffolding | `--code-location <path>`, `--entrypoint <file>` (default `main.py`) |
| `import` | Adopts an existing **Bedrock Agent** — the older pre-AgentCore product. *Not* for importing exercises | `--agent-id`, `--agent-alias-id`, `--region` |

`byo` was an open question in the plan: could it shorten conversions by adopting exercise code in
place? Having seen the flags — **no**. `--code-location` would have to point at `_source/`, which is
immutable by design, and the converted copy gets edited heavily. `create` + copy is the right shape.

## Invocation

| Old | New |
|---|---|
| `agentcore invoke '{"message": "..."}'` | `agentcore invoke "<prompt>"` |
| `agentcore invoke '{"prompt": "...", "session_id": "t1", "customer_id": "CUST-123"}'` | `agentcore invoke --session-id t1 --user-id CUST-123 "<prompt>"` |

**There is no arbitrary-JSON-payload flag.** Identity moved out of the payload and onto flags:
`--session-id` and `--user-id`. `--prompt-file` is the escape hatch for long or structured input.

### 🔴 The two identity flags do NOT arrive the same way

`RequestContext` (`bedrock_agentcore.runtime.context`) has exactly **three** fields:

```
session_id       : str | None
request_headers  : Dict[str, str] | None
request          : Any | None
```

| Flag | Reaches code as | Notes |
|---|---|---|
| `--session-id` | `context.session_id` | a real typed field; `dev` also populates it (`local-dev-session`) |
| `--user-id` | **nothing** | consumed by AgentCore Identity; `is_forwardable_header` rejects it |

### Getting a per-customer id into agent code

**Any header outside the restricted list is forwardable** — `X-Actor-Id`, `X-Customer-Id`, even a bare
`Actor-Id`. Blocked are: the restricted list, `x-amz-*` (SigV4), and `x-amzn-*`.

The `X-Amzn-Bedrock-AgentCore-Runtime-Custom-` prefix is **not a requirement** — the docs describe it as
backward compatibility. It is an *exemption from the `x-amzn-*` ban*, useful only if you already have
such a name. Choosing it for a new header buys nothing and costs 44 characters.

Forwarding is **opt-in per runtime**; any name is dropped unless declared:

```bash
agentcore add agent … --request-header-allowlist X-Actor-Id
agentcore invoke -H "X-Actor-Id: alice-001" "<prompt>"
```

Stored under `runtimes` in `agentcore/agentcore.json`. Limits: 20 headers, 4 KB each, case-insensitive
names, no case-duplicates. `Authorization` is forwardable only with `--authorizer-type CUSTOM_JWT` —
that's the route for *authenticated* identity rather than a caller-asserted label.

| Channel | Works | How |
|---|---|---|
| Any non-restricted header (`X-Actor-Id`) | ✅ | `invoke -H`, plus an allowlist entry |
| Payload key (`actor_id`) | ✅ | **raw API only:** `aws bedrock-agentcore invoke-agent-runtime --payload fileb://p.json` |
| `--user-id` | ❌ | never reaches code |

**`agentcore invoke` sends `{"prompt": "…"}` and nothing more.** `--prompt-file` is read as the prompt
*string*, so a JSON file arrives as literal text — it is not a payload escape hatch. `--payload` on the
raw API is a blob and needs `fileb://`; a plain string fails with `Invalid base64`.

**The generated scaffold gets this wrong.** `agentcore add agent` writes
`user_id = getattr(context, 'user_id', 'default-user')`, which can only ever return `'default-user'` —
`getattr`'s default hides the missing attribute completely. Do not take that line as documentation.

Read the header case-insensitively, since `dict.get` is case-sensitive and header names are not:

```python
def _header(context, name):
    headers = getattr(context, "request_headers", None) or {}
    t = name.lower()
    return next((v for k, v in headers.items() if k.lower() == t), None)
```

⚠️ **Never give an identity field a plausible fallback.** `payload or context or "alice-001"` reads as
defensive but converts "unknown caller" into "the last id someone hardcoded". In this project it served
one customer's memories to a request made as another, and filed the turn under the wrong actor —
`list-actors` never showed the second user at all. Fail closed, or fall back to a sentinel that is
obviously not a customer.

`runtimeSessionId` also has a **33-character minimum**, so session IDs are meant to be generated rather
than hand-written. A hyphen-less UUID is 32, one short.

Payload key also changed: the old CLI's raw JSON meant exercises invented their own keys (`message`);
the new CLI sends **`prompt`**.

## Resources

| Task | Old | New |
|---|---|---|
| Memory | `agentcore memory create <name>` | `agentcore add memory` → `deploy`. **verified:** there is no top-level `memory` verb at all — `agentcore memory --help` falls through to the root command. `add memory` takes `--strategies SEMANTIC,SUMMARIZATION,USER_PREFERENCE,EPISODIC`, `--expiry` (**default 30**, vs the exercises' 7), `--delivery-type kinesis` + `--data-stream-arn` + `--stream-content-level`, and `--indexed-key key:STRING|STRINGLIST|NUMBER` (max 10). The SDK escape hatch is `MemoryClient.create_memory_and_wait(...)` — but a resource made that way is **invisible to `agentcore.json`**, so `remove`/`deploy` will never clean it up and teardown must be a manual `delete_memory` |
| Gateway | `agentcore gateway create-mcp-gateway` | `agentcore add gateway` → `deploy` |
| Gateway target | `agentcore gateway create-mcp-gateway-target` | `agentcore add gateway-target --type api-gateway\|lambda-function-arn\|mcp-server\|open-api-schema\|smithy-model` |
| Knowledge base | *(console / SDK)* | `agentcore add knowledge-base --source s3://… --data-source-type s3` — **verified present.** Exposes no vector-store or embedding choice; see the cost warning in the plan |
| Identity | `agentcore identity setup-*` | `agentcore add credential` |
| Observability | toolkit/CloudWatch workflow | `agentcore logs`, `agentcore traces list`, `agentcore traces get <id>` — **all verified 2026-09-20**, see `observability-runbook.md`. `traces list --runtime X` prints a session-grouped trace list + console deep-link; `traces get <id>` downloads full OTEL gen-AI JSON (model, token counts, TTFT, tool spans) — a superset of the console dashboard. No `stop-session` verb exists; use distinct `--session-id` per invoke |

## AgentCore Identity — `agentcore add credential`

The new CLI's replacement for the old `agentcore identity setup-*` verbs. Creates a credential
provider that a **gateway target** uses for *outbound* auth when calling the backend it fronts.

```bash
agentcore add credential --name <name> --type api-key --api-key <value>
```

| Flag | Notes |
|---|---|
| `--name` | Referenced later by `add gateway-target --credential-name` |
| `--type` | `api-key` (**default**) or `oauth` |
| `--api-key` | The key value, for `--type api-key` |
| `--discovery-url`, `--client-id`, `--client-secret`, `--scopes` | OAuth only |

Then link it when creating the target:

```bash
agentcore add gateway-target --name loyalty-api --gateway wanderbot-gateway \
  --type api-gateway --rest-api-id <id> --stage <stage> \
  --outbound-auth api-key --credential-name <name>
```

### Valid `--outbound-auth` depends on the target type — *in the CLI*

From `agentcore add gateway-target --help`. **This is a CLI-level restriction, not a documented
service one** — see the caveat below:

| Target type | Valid `--outbound-auth` |
|---|---|
| `mcp-server` | `oauth` or `none` |
| `lambda-function-arn` | `oauth` or `none` — the CLI rejects `api-key` |
| `http-runtime` | `oauth` or `none` |
| **`api-gateway`** | **`api-key` or `none`** |
| `open-api-schema` | `oauth` or `api-key` — **required**, no `none` |
| `smithy-model` | `none` only (uses the gateway IAM role) |
| `passthrough` | `gateway-iam-role`, `oauth`, or `jwt-passthrough` |

### ⚠️ The service API is more permissive than the CLI

*(Corrected. An earlier version of this file asserted a Lambda target "cannot carry an API key". That
overstated a CLI rule as a service rule.)*

Checked against the AgentCore API itself — `aws bedrock-agentcore-control create-gateway-target help`
and the `CreateGatewayTarget` API reference:

- `credentialProviderType` is a **flat enum**: `GATEWAY_IAM_ROLE | OAUTH | API_KEY |
  CALLER_IAM_CREDENTIALS | JWT_PASSTHROUGH`.
- `credentialProviderConfigurations` is **`Required: No`**, "Fixed number of 1 item" — so a target may
  carry no credential at all, or exactly one.
- `targetConfiguration` and `credentialProvider` are each tagged unions, and **no cross-constraint
  between them is documented anywhere in the API reference.**

So on paper the service would accept `API_KEY` with a Lambda target configuration. Whether it
*enforces* a pairing server-side is **unverified** — the prose devguide page for target configuration
renders empty (JS-only), so there is no authoritative statement either way. Only an actual
`create-gateway-target` call would settle it.

**Practical takeaway, correctly scoped:**

- **The CLI will not let you attach an API key to a `lambda-function-arn` target.** That is a real
  constraint on the path we are using, whatever the API permits.
- Yesterday's `booking-tools` target passed no credential at all — valid, since the field is optional.
  It authenticates as the gateway's IAM role, which CDK grants `lambda:InvokeFunction` on.
- **Via the CLI, API-key auth is reachable only through `api-gateway` or `open-api-schema` targets.**
  So a key-protected backend must be fronted by an API Gateway REST API (or described by an OpenAPI
  schema) — which is exactly the shape the loyalty exercise takes.

Terminology note: "outbound" is gateway → backend. It is unrelated to `add gateway
--authorizer-type`, which governs inbound agent → gateway auth and is `NONE` in this project.

## 🔴 Naming rules CONTRADICT between resource types

Every AgentCore resource type validates its name differently, and two of them are exact opposites.
None can be inferred from another. Both mistakes below were made on this project.

| Resource | Pattern | Dashes | Underscores | Leading digit |
|---|---|---|---|---|
| **Gateway target** | `^([0-9a-zA-Z][-]?){1,100}$` | ✅ **required style** | ❌ rejected | ✅ allowed |
| **Browser** | `[a-zA-Z][a-zA-Z0-9_]{0,47}` | ❌ **rejected** | ✅ allowed | ❌ rejected |
| **Memory** | `[a-zA-Z][a-zA-Z0-9_]{0,47}` | ❌ **rejected** | ✅ allowed | ❌ rejected |
| Runtime (`add agent --name`) | alphanumeric + underscores, max 48 | ❌ | ✅ | — |
| Project (`create --project-name`) | ≤23 chars, alphanumeric, leading letter | ❌ | — | ❌ |

Observed errors:

```
Property value [booking_tools] does not match pattern: ^([0-9a-zA-Z][-]?){1,100}$
Value 'wanderbot-browser' at 'name' failed to satisfy constraint:
  Member must satisfy regular expression pattern: [a-zA-Z][a-zA-Z0-9_]{0,47}
Value at 'name' failed to satisfy constraint:            # CreateMemory, same pattern
  Member must satisfy regular expression pattern: [a-zA-Z][a-zA-Z0-9_]{0,47}
```

**Memory is confusing because the ID contains a dash you did not type.** `CreateMemory` rejects
`WanderBot-LongTerm` but returns an ID like `WanderBot_LongTerm-WWZDcxENNZ` — the service appends
`-<10 chars>` itself. So a memory *ID* seen in code looks like it violates the *name* rule. Pass the
name; never reconstruct an ID by hand.

So `booking-tools` is required for a target and `wanderbot_browser` is required for a browser —
swapping them fails both times. **Read the error's regex rather than reasoning from another
resource type.** Gateway targets fail at CDK change-set validation (before anything is created);
browsers fail at the API call.

## Two AWS services, not one — resources vs sessions

AgentCore is split across two CLI services, and a command missing from one is usually in the other.

| Service | Owns | Examples |
|---|---|---|
| `aws bedrock-agentcore-control` | **resources** (long-lived, provisioned) | `create-browser`, `create-gateway`, `create-gateway-target`, `create-api-key-credential-provider`, `list-gateways` |
| `aws bedrock-agentcore` | **sessions and invocations** (ephemeral) | `start-browser-session`, `list-browser-sessions`, `get-browser-session`, `invoke-browser`, `start-code-interpreter-session`, `list-sessions` |

Same shape as `bedrock` vs `bedrock-runtime`. Looking for session operations in the control plane
returns `Found invalid choice`, which reads as "this feature has no CLI support" when it simply lives
next door.

Practical consequence for custom resources: with the **default** browser, AWS manages session
lifecycle. Once you `create-browser` your own, stuck sessions are yours to find and stop —
`list-browser-sessions` / `stop-browser-session`.

## Not AgentCore resources

Lambda functions and API Gateway REST APIs are plain AWS in both generations — console or AWS SDK.
AgentCore Gateway *targets* point at them once they exist.

---

## Local development (`agentcore dev`)

All verified against 0.27.1 on this machine.

### Three launch modes — chosen at start, not switchable while running

| Mode | Command | Interactive? |
|---|---|---|
| Browser chat UI + inspector | `agentcore dev -r <Agent> --skip-deploy` | yes |
| Terminal TUI | `agentcore dev -r <Agent> --skip-deploy -b` | yes, no browser |
| Headless, logs to stdout | `agentcore dev -r <Agent> --skip-deploy --logs` | **no** — one command per prompt |

`--logs` is marked `[non-interactive]`, which is why it forces
`agentcore dev -r <Agent> "<prompt>" --stream` as a separate command for each test. Fine for scripted
parity runs, tedious by hand — use `-b` when iterating.

`--skip-deploy` is worth making a habit: without it, `dev` attempts to deploy project resources first.

### `--stream` is display-only

`--stream` controls whether the **client** prints tokens as they arrive or buffers until the response is
complete. It does not change what the server does, and it has nothing to do with concurrency despite how
the name reads.

The server streams either way: the entrypoint is `async for event in agent.stream_async(...)` with
`yield event`, so the runtime emits an event stream on every invocation. Worth being clear on, because a
streaming-path failure will occur with or without the flag — the Claude Sonnet `ConverseStream` denial we
hit was not caused by passing `--stream`.

Practical reason to use it when testing: a slow or hung call is *visible* as tokens appear, instead of a
silent terminal you can't distinguish from a freeze.

### Ports are offset by runtime index

Base is `8080`, plus the runtime's position in `agentcore.json`. In this project:

| Runtime | Port |
|---|---|
| `Scratch` | 8080 |
| `WanderBot_CodeInterpreter` | 8081 |
| `WanderBot_FunctionCalling` | 8082 |

Override with `-p`. `-r` becomes mandatory once more than one runtime exists.

### `dev` cannot set session or actor identity

`dev` exposes only `-p/--port`, `-r/--runtime` and `--tool`. **`--session-id` and `--user-id` are
`invoke`-only.** So anything reading identity off `context` is unreachable locally and falls back to
whatever default the code supplies.

Consequence for memory work: same-session recall is testable locally (constant defaults are enough),
but **cross-session behaviour is not** — that needs a deploy. Also note `-b` (TUI) mode does **not**
capture `app.logger` output into the dev log, unlike `--logs`, so a `dir(context)` probe added to
settle the attribute name produced nothing.

### Hot reload

The server watches `app/<Agent>/` only. Editing `main.py` reloads automatically — a restart is needed
only to change launch mode, not to test code changes.

### Stopping — and the leak to watch for

`Ctrl+C` in the foreground terminal is the normal path. **But killing a wrapper shell does not always
reap the children**: observed once where the shell was terminated yet the `node agentcore dev` process
and two uvicorn workers kept holding `:8082`.

```bash
lsof -nP -iTCP:8082 -sTCP:LISTEN   # who holds the port
pkill -f 'agentcore dev'           # kill all dev servers
```

A stale server on the port is a nasty failure mode — you get responses, just not from the code you
think you're running.

### The server is plain HTTP

Useful for scripting a test battery without the CLI:

```bash
curl -s localhost:8082/invocations -H 'Content-Type: application/json' \
  -d '{"prompt":"Find flights from BCN to FCO on 2026-03-20"}'
```

### ⚠️ Counting tool calls from the dev log: the off-by-one

The dev log writes markers in this order:

```
[SSE] ...toolUse frames for turn N...
[SYSTEM] → <prompt for turn N>          <- written AFTER the stream
[RESPONSE] <answer for turn N>
```

So **`toolUse` frames appearing *before* a `[SYSTEM] →` line belong to that prompt**, not the previous
one. Segmenting on `[SYSTEM] →` and counting frames after it attributes every turn's calls to the
wrong turn, and makes the final turn look like it made none. This produced a false "zero tool calls"
finding in conversion log entry 4 that had to be retracted.

Don't count frames. **Extract the arguments**, which tells you far more anyway:

```bash
python3 - "<logfile>" <<'EOF'
import sys, re
lines = open(sys.argv[1]).read().splitlines()
turn, buf = 1, ""
for ln in lines:
    m = re.search(r'"toolUse": \{"input": "(.*?)"\}\}', ln)
    if m:
        buf += m.group(1)
    if '[SYSTEM] \u2192' in ln:
        print(f"turn {turn}: {ln.split(chr(0x2192),1)[1].strip()}")
        print(f"   tool input -> {buf.encode().decode('unicode_escape') if buf else '(NO TOOL CALL)'}\n")
        turn += 1; buf = ""
EOF
```

Tool *arguments* are the real evidence for memory and context tests: a value in the input that never
appeared in that turn's prompt can only have come from recall. Absence of a tool call proves much
less, and is the easier thing to measure wrong.

### Logs on disk

The startup banner prints `Log: agentcore/.cli/logs/dev/dev-<timestamp>.log`. Reading that file is how
tool execution gets *proved* rather than assumed — it captured the Code Interpreter's model-authored
Python. `--logs` appears to additionally tee to stdout rather than being what creates the file, but that
is inferred from the banner, not confirmed.

## AgentCore Memory — three provisioning routes

`agentcore memory create` (old toolkit) and `agentcore memory browse` (README bonus) **do not exist**;
there is no top-level `memory` verb. Replacements:

| Need | Route |
|---|---|
| strategies, CDK-tracked, don't care about namespaces | `agentcore add memory --strategies SEMANTIC,USER_PREFERENCE --expiry <days>` |
| explicit namespace templates | `aws bedrock-agentcore-control create-memory --memory-strategies '<json>'` |
| **both** | create with `aws`, then `agentcore import memory --arn <arn> --name <local>` |

`add memory`'s own flags are in **Resources** above. New here: the API's `--event-expiry-duration` is
required, min 3, max 365, and is an **integer of days** despite help text claiming "ISO 8601 duration"
(`P15D` fails).

### Two vocabularies for one concept

`aws … create-memory --memory-strategies` is a tagged union: `semanticMemoryStrategy`,
`summaryMemoryStrategy`, `userPreferenceMemoryStrategy`, `customMemoryStrategy`,
`episodicMemoryStrategy`. `agentcore add memory --strategies` uses the enum spellings. `get-memory`
reads back the **enum**, so code doing `s["type"]` sees `SEMANTIC` — camelCase in, SCREAMING_SNAKE out.

Sending `namespaces` populates **both** `namespaces` and `namespaceTemplates`.

### `add agent --memory` is a capability level, not a reference

`none` | `shortTerm` | `longAndShortTerm`. There is no flag for "use this memory", and
`longAndShortTerm` **declares a new memory in `agentcore.json`** rather than reusing the project's — a
duplicate that materialises on the next deploy. Either `add agent` first and import after, or pass
`--memory none`. `agentcore remove memory --name <name>` backs it out; config-only, nothing in AWS.

### `import memory` runs two CloudFormation phases

Phase 1 creates companion IAM; Phase 2 performs the CFN import and attaches the execution role. State
read between phases is misleading — `memoryExecutionRoleArn` reads `null` and no
`AWS::BedrockAgentCore::Memory` appears in the stack until Phase 2 lands. **Wait for
`IMPORT_COMPLETE`.** Afterwards CloudFormation owns the resource, so stack deletion is destructive.

### How a deployed agent gets the memory ID

An **injected env var**, `MEMORY_<CONFIG_NAME_UPPERCASED>_ID` — derived from the name in
`agentcore.json`, so `wanderbot_longTermMemory` gives `MEMORY_WANDERBOT_LONGTERMMEMORY_ID`. Read it with
`os.environ.get(<name>, "<literal>")`: `dev` injects nothing, so the fallback is what makes local runs
work.

For identity, see **Invocation** above. The code-side accessors are
`getattr(context, "user_id", …)` and `getattr(context, "session_id", …)` — confirmed in the
`--memory longAndShortTerm` scaffold. Neither flag exists on `dev`, so both pin to fallbacks locally.

### Inspecting what memory actually stored

```bash
aws bedrock-agentcore list-actors                 --memory-id <id>
aws bedrock-agentcore list-memory-records         --memory-id <id> --namespace <ns>
aws bedrock-agentcore list-events                 --memory-id <id> --actor-id <a> --session-id <s>
aws bedrock-agentcore list-memory-extraction-jobs --memory-id <id>   # watch async extraction
```

## Which runtime is that CDK asset?

Every deploy re-stages **all** runtimes, so an eight-runtime project uploads ~400 MB per command. To
decode the hashes the CLI prints:

```bash
python3 -c "
import json,re
t=json.load(open('agentcore/cdk/cdk.out/AgentCore-<Project>-default.template.json'))
for lid,r in t['Resources'].items():
    for h in re.findall(r'([0-9a-f]{64})\.zip', json.dumps(r)):
        print(h[:12], lid)
"
```

The bootstrap bucket's lifecycle policy only expires **noncurrent** versions. Asset keys are content
hashes, so every object is permanently current and nothing is ever reclaimed.

## ⚠️ Both CLIs write a build cache that will break a GitHub push

Each CLI drops its build output **next to your source**, not in a temp dir, and both are large enough to
matter. Add them to `.gitignore` *before the first commit*, not after:

| CLI | Path to ignore | Size observed |
|---|---|---|
| Old toolkit (`agentcore launch`) | `.bedrock_agentcore/` | `<Agent>/dependencies.zip` at **102 MB** |
| New CLI (`agentcore deploy`) | `agentcore/cdk/cdk.out/` | ~50 MB per runtime, re-staged every deploy |

The old toolkit's zip is the dangerous one: **102 MB exceeds GitHub's hard 100 MB per-file limit**, so the
push is rejected outright rather than merely being slow.

```
remote: error: File starter/.bedrock_agentcore/CustomerSupportAgent/dependencies.zip
        is 102.10 MB; this exceeds GitHub's file size limit of 100.00 MB
remote: error: GH001: Large files detected.
```

**Why it bites:** the failure is delayed and lands far from its cause. `git add -A` accepts the file
without complaint, the commit succeeds, and rejection only happens at `push` — by which point the blob is
already in history, so deleting the file in a *new* commit does not help. A 115 MB `.git` for a project
whose source is under 1 MB is the tell.

Recovery depends entirely on whether the commit left your machine:

```bash
# Find the offender: biggest blobs in the current tree
git ls-tree -r -l HEAD | sort -k4 -rn | head

# Unpushed — amend, no force-push, files stay on disk
git rm -r --cached --quiet <cache-dir>
git commit --amend --no-edit

# Reclaim the space the blob still occupies in the pack
git reflog expire --expire=now --all && git gc --prune=now
```

If it **was** pushed, this becomes a history rewrite (`git filter-repo`) plus a force-push — which is why
the ignore rule belongs in place before the first deploy, not after the first rejected push.

Generalises past AgentCore: any deploy tool that vendors dependencies into a local build directory has
this shape. Check what a new deploy command wrote into your working tree before committing.

## ⚠️ `deploy --diff` is not read-only

Documented as *"Show CDK diff without deploying"*, but it writes before it reads:

```
STEP: Set up API key providers
Token vault encrypted with KMS key: arn:aws:kms:...:key/63238cbc-...
Updated API key provider: wanderbot-loyalty-api-key
```

**Credential providers are managed outside CDK**, so the CLI syncs them eagerly in every mode. The key
value comes from the environment, not from `agentcore.json` (which stores only `name` and
`authorizerType`) — so if the environment is missing or stale when you run a *diff*, you have silently
overwritten a live credential. Treat `--diff` and `--dry-run` as "no CloudFormation changes", not as
"no changes".

Corollary: keep `.env` out of git. The value is read from the environment on every invocation, so the
file is a real credential, not a template.

Synth also bundles **all** runtimes' assets before it can compute a diff, so `--diff` on a
nine-runtime project is nearly as slow as a deploy and re-uploads the same ~430 MB.

## Two customer-managed KMS keys, ~$1/month each

| Key alias / id | Created by |
|---|---|
| `alias/cdk-hnb659fds-assets-key` | CDK bootstrap, i.e. the first `deploy` |
| `63238cbc…` *"AgentCore Identity encryption key"* | `agentcore add credential` — the token vault |

These are the only fixed monthly charges in the project; everything else is per-use. *(Correction: an
earlier note named bootstrap as the sole source.)*

## Reading the dev log depends on which mode wrote it

The two `dev` modes produce **different log formats**, and a recipe for one finds nothing in the other.

| Mode | Tool calls appear as | Arguments visible? |
|---|---|---|
| `--logs` | plain lines: `[20:33:06] Tool #1: search_flights` | ❌ name only |
| SSE/stream capture | `"toolUse":{"input":"…"}` deltas to reassemble | ✅ |

So `--logs` proves *which* tool ran, never *with what*. Where a test depends on argument carry-over —
proving memory recall, for instance — `--logs` cannot supply the evidence.

⚠️ **Never segment a dev log on the `User:` / `[SYSTEM] →` marker.** In `--logs`, `User:` is written when
a request *arrives* while `Tool #N` flushes with the *previous* turn's response, so a tool call appears
directly beneath the next prompt. In the TUI-mode logs the marker arrives *after* its own turn instead.
Both produce an off-by-one, in opposite directions. **Pair tool calls to prompts by content, not by
position** — this has produced one retracted finding (conversion log #4) and one near-miss (#10).

⚠️ **The log file lags the terminal.** A turn visible in the terminal may have no `Tool #N` line yet.
Wait, or re-send the prompt, before concluding a tool didn't run.
