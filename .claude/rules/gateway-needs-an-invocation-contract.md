# A gateway is only as dynamic as its invocation layer

A "tool gateway" has three layers — registration → discovery → invocation — and is only as dynamic as the
weakest. Registration/discovery are easy to make dynamic; invocation is the hard part. Runtime-dynamic tools
need a standard invocation contract (MCP), where the schema IS the binding — the client needs no per-tool
code to discover OR invoke.

Test: add a backend; can the agent INVOKE it with zero new code? If invocation still needs a new per-tool
binding, it's a registry with a nice name, not a gateway. Discovery without matching dynamic invocation is
theater — check dynamism claims at the weakest layer.
