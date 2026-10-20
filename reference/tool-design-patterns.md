# Tool design patterns

Decisions that recur when giving an agent tools. Each entry is a *choice* with the evidence that
settled it, not a tutorial.

---

## Plain function vs code-as-string in a sandbox

**Default: write a plain `@tool` Python function.** It's simpler, faster, type-checkable, and your
IDE can see it. Reach for a sandbox only when one of these is true:

| Reason to use a sandbox | Example |
|---|---|
| **The code is untrusted** | the LLM authored it, or a user submitted it — isolate it from your filesystem, network and credentials |
| **The code is dynamic** | assembled at runtime from rules that live in data (a pricing engine reading from a DB), so it cannot be written in advance |
| **You need a provable execution boundary** | fresh sandbox per call, auditable, no state carry-over between customers (`clearContext=True`) |
| **You need capabilities the agent runtime lacks** | heavy numpy/pandas work you don't want in the agent container |

If none apply, the sandbox is buying you a network round-trip and a serialization problem in exchange
for nothing.

### Why it has to be a string at all

The sandbox is a **different process on a different machine**. `executeCode`'s contract is
`{"code": "<text>", "language": "python", "clearContext": True}` — `code` is text. You cannot hand a
remote executor a function object, a closure, or a reference to a local variable; there is no shared
memory to reference. So you marshal logic the way you'd marshal data.

`clearContext=True` compounds it: the sandbox is stateless per call, so the string must be
**fully self-contained** — every import, constant and input value inline. Nothing from your module is
in scope over there.

### Two directions, and only one is safe

| Direction | Who writes the code | Risk |
|---|---|---|
| **LLM authors the code**, tool executes it | the model | the model can emit anything — this is where sandbox isolation is genuinely load-bearing |
| **Tool authors the code**, model supplies typed args | you | the model can only choose values; isolation guards against little |

The AgentCore Code Interpreter exercise used the first; the capstone rubric requires the second.
The second is much safer, which is also why the sandbox does less work there — a useful thing to be
honest about rather than to oversell.

### Minimize the interpolated surface

Building a large f-string means doubling every literal brace (`{{"a": 1}}`) and remembering `!r` on
every string value, or you generate `tier = Gold` — a `NameError` inside the sandbox. Both are easy to
get wrong and neither fails until runtime, remotely.

Confine interpolation to one line instead:

```python
STATIC_ALGORITHM = """
import json
rates = {"standard": 1, "device": 2}     # plain string: no doubling needed
qty   = INPUT["qty"]
...
"""

code = f"INPUT = {json.dumps(inputs)}\n" + STATIC_ALGORITHM
```

One interpolation point, zero doubled braces, and `json.dumps` handles quoting so `!r` is unnecessary.
Same reasoning as parameterized SQL over string concatenation: shrink the region where data and code
are mixed.

### Verify the generated code without deploying

The generated string is code, so compile it locally before trusting a remote run — this caught nothing
in the capstone only because it was checked:

```python
rendered = eval(fstring_source, {}, sample_args)   # or call the builder
compile(rendered, "<generated>", "exec")           # syntax
exec(rendered, {})                                 # behaviour — capture stdout and assert
```

Assert the output against numbers computed **independently from the source data**, not against what
the code happens to produce. In the capstone the expected discount was derived from
`product_catalog.txt` before the tool existed; the agreement is what made it evidence.

### Always pair it with a fallback

A remote sandbox is a dependency that can be unavailable. The fallback should compute a **degraded but
correct** result in-process and say so in its output (a `degraded: true` flag the model can relay),
rather than raising into the customer's face.

Telling detail: if the fallback is trivial to write, the sandbox was not load-bearing for that
computation. That is worth noticing rather than hiding.

---

## Related findings elsewhere in these docs

- **Tool descriptions are functional text, not documentation** — the model has only the description to
  decide *when* to call a tool. `gateway-runbook.md` (`toolOverrides`) and the capstone's
  `starter/gateway_tool_descriptions.py`.
- **`Args:` docstring blocks do not produce per-parameter schema docs** — Strands puts the whole
  docstring in one `description` field. Format constraints must read naturally in the prose.
  `.claude/skills/convert-exercise/SKILL.md`.
- **Required vs optional in a tool schema comes only from the presence of a Python default.**
- **A tool's return format is part of its contract** — meaning carried only by an emoji, colour or
  layout is information the model can silently drop (`conversion-log.md` entry 2).
- **Static vs dynamic tool schemas** — a Lambda target needs a hand-written schema (exact control, junk
  free); an API Gateway target derives one from the OpenAPI export (free, but you inherit artifacts
  like `basePath`). `gateway-runbook.md`.
