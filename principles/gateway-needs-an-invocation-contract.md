# A tool gateway is only as dynamic as its invocation layer

> **Rule:** A "tool gateway" has three layers — **registration → discovery → invocation** — and is only as
> dynamic as its *weakest* one. Registration and discovery are easy to make dynamic; **invocation** is the
> hard part. True runtime-dynamic tools require a *standard invocation contract* (MCP) where **the schema IS
> the binding**. Without it you hardcode a per-tool binding — and that's a registry with a nice name, not a gateway.

*(Terse: `.claude/rules/gateway-needs-an-invocation-contract.md`.)*

## What it means

It's tempting to call any tool registry a "gateway." But the value word is *dynamic invocation*: can the agent
*call* a newly-registered tool with no new code? Three readings, increasing dynamism:
- **Naive sim** — a hand-written `@tool` binding per backend (dynamic registry, static invocation).
- **Generic proxy** — one `@tool` that forwards any registered call (mostly dynamic; a single-app imitation).
- **Real MCP Gateway** — fully dynamic: the protocol standardizes how a tool advertises its
  name/description/input-schema and how a call + result are framed, so the client needs *no* bespoke per-tool
  code to discover **or** invoke. The schema is the binding.

## Evidence (L11 analytics-gateway sim)

A `LambdaGateway` was dynamic in registration + discovery but **static in invocation**: the agent
*discovered* a newly-registered `stock_price` tool (it appeared in the system prompt) but had no `@tool`
binding to actually call it — so it mis-fired `get_weather` three times and gave up. The seam was exactly at
the invocation layer.

## The test

**"Is it really a dynamic gateway?"** → Add a backend. Can the agent *invoke* it with zero new code? If
invocation still needs a new per-tool binding, it's a registry, not a gateway. The thing that earns the word
"gateway" is a standard invocation contract (MCP), not the registry.

## Why it's Tier-1

Applies to any tool/plugin/gateway/MCP architecture. The general lesson: dynamism claims must be checked at
the *weakest* layer, and "discovery" is cheap theater without matching dynamic invocation. MCP is the
cross-vendor version of the generic proxy; no standard interface → back to hardcoding a binding per backend.

## Source
`reference/multi-agent-patterns.md` (§gateway) + `gateway-runbook.md`, L11 analytics-gateway.
