# Python, read against Swift and Kotlin

Running notes for constructs where Python's *model* differs structurally from Swift/Kotlin — not where
the syntax merely differs. `for` loops and string formatting don't earn an entry.

Grows one entry per exercise. Entries are added when a construct is first hit.

---

## 1. Decorators are functions that run at import — `@tool`, `@app.entrypoint`

*First seen: Code Interpreter exercise.*

```python
@tool
def calculate_trip_cost(code: str, description: str = "") -> str:
    """Execute Python code in an isolated AgentCore sandbox..."""
```

`@tool` is **not** an annotation. It is a function that executes when the module is imported, receives the
function object below it, and returns a replacement object. Strands uses that moment to read the
signature and docstring and build a JSON schema.

| | Mechanism | When |
|---|---|---|
| Python | Decorator: a callable that takes the function and returns a replacement | **Import time**, every run |
| Kotlin | `@Annotation` is inert metadata. Acting on it needs KSP/kapt (codegen) or `kotlin-reflect` | Compile time, or runtime with a reflection dependency |
| Swift | `@attached(...)` macros expand into source; older attributes are compiler directives | Compile time |

The practical consequence isn't stylistic. **Your type hints and docstring *are* the tool contract the
model sees.** A wrong hint is a wrong schema; a vague docstring is a tool the model calls at the wrong
time. In Kotlin you'd hand-write a schema or generate one; here inaccuracy in a comment is a functional
bug.

---

## 2. Type hints are live data, not erased

```python
def calculate_trip_cost(code: str, description: str = "") -> str:
```

Python keeps annotations as objects reachable at runtime (`inspect.signature()`,
`typing.get_type_hints()`), which is the only reason a library can derive a schema from a plain function.

- **Kotlin** erases generics and needs `kotlin-reflect` on the classpath for signature detail.
- **Swift** has only `Mirror`, and it's limited.

Corollary, learned the hard way in this exercise: **annotations are not enforced.** The original file
declared `-> dict` while returning a Strands `AgentResult`. Nothing failed — no interpreter checks it.
In Swift or Kotlin that's a compile error. Here it needs `pyright`/`mypy` to surface, which is why those
tools aren't optional on a real Python project.

---

## 3. Docstrings survive to runtime as `__doc__`

The docstring under `def` is a string literal assigned to `__doc__`, readable by any library. That's how
Strands gets the tool's description.

Swift's `///` and Kotlin's KDoc are **stripped by the compiler**. There is no runtime equivalent at all —
documentation and behaviour are separate concerns there. In Python, for a `@tool`, they're the same thing.

---

## 4. A module body is an implicit initializer

```python
app = BedrockAgentCoreApp()
model = BedrockModel(model_id="us.amazon.nova-2-lite-v1:0")
agent = Agent(model=model, tools=[calculate_trip_cost])
```

These run **once**, on first import; the module object is then cached, so they're effectively singletons.
Nearest analogues: a Kotlin `object`, or Swift `static let` (lazy, `dispatch_once`-backed).

The difference is that Python's version is **implicit** — no keyword marks it. Nothing tells you this file
has initialization side effects; you infer it from statements at column zero. Hence the real bug in the
source exercise: the `Agent` was built *inside* the request handler and rebuilt on every call, which in
Kotlin would have looked obviously wrong because you'd have had to write `object` or not.

The capstone rubric grades `BedrockAgentCoreApp()` being at module level specifically.

---

## 5. `dict.get(k, default)` — where all three agree

```python
user_message = payload.get("prompt") or payload.get("message", "Hello!")
```

Kotlin `?:`, Swift `??`. Genuinely the same idea. Noted only to mark it as *not* worth further thought.

One Python-specific trap: `or` falls through on **any** falsy value, not just `None` — so an empty string
`""` also falls through to the next branch. Kotlin's `?:` and Swift's `??` trigger on null/nil only.

---

## 6. `async def` + `yield` = async generator

```python
async def invoke(payload, context=None):
    async for event in agent.stream_async(user_message):
        yield event
```

A function containing both `async def` and `yield` is an async generator, consumed with `async for`.

| | Equivalent |
|---|---|
| Kotlin | `Flow<T>` — `flow { emit(x) }`, collected with `.collect {}` |
| Swift | `AsyncStream<T>` / `AsyncSequence`, consumed with `for await` |

The concepts line up closely. The difference is **who owns backpressure**: Python's generator protocol is
pull-based by default, so the consumer's pace throttles the producer implicitly. Kotlin's `Flow` is also
cold and pull-driven, but buffering is explicit (`buffer()`, `conflate()`), and Swift's `AsyncStream` makes
you choose a buffering policy at construction.

**Why it matters here:** the source exercise wrote `async def invoke` and then called the *synchronous*
`agent(...)` inside it — an async function that blocks. It "worked," which is the problem. The capstone's
`nest-asyncio` dependency exists to make that nested-loop pattern survivable. Streaming properly means the
dependency isn't needed at all.

---

## 7. Decorator syntax takes an *expression*, not a name

*First seen: `@app.entrypoint`.*

```python
app = BedrockAgentCoreApp()

@app.entrypoint          # attribute access on an object that must already exist
async def invoke(payload, context): ...
```

`@app.entrypoint` is not a name the language knows. It's `app`, then `.entrypoint` on it, then the
result is called with the function below. Python allows any expression after `@` — dotted paths,
calls with arguments (`@tool(name="x")`), even subscripts.

- **Kotlin** annotations are *names* with optional constant arguments. `@app.Entrypoint` is not
  expressible: annotations aren't evaluated, so there's no object to read an attribute from.
- **Swift** attributes and macros are likewise resolved by name at compile time.

Consequence: the decorator can register the function **on that specific instance**, which is why
`app = BedrockAgentCoreApp()` has to appear above it. Ordering matters in a way it doesn't for a
Kotlin annotation.

Common mistakes: `@app_entrypoint` (one identifier — `NameError`), `@app.entryPoint` (Python is
`entrypoint`, one lowercase word).

---

## 8. `async` / `await` are real keywords — but unenforced

Python has had both since 3.5, across four constructs: `async def`, `await`, `async for`, `async with`.

| | Python | Kotlin | Swift |
|---|---|---|---|
| Declare | `async def f()` | `suspend fun f()` | `func f() async` |
| Consume | `await f()` | plain call — the call site *is* the suspension point | `await f()` |
| Keywords | `async`, `await` | `suspend` only. **`async {}` is a library function** (`kotlinx.coroutines`), and there is no `await` — you call `.await()` on a `Deferred` | `async`, `await` |
| Event loop | you supply it (`asyncio.run`, or a framework) | you supply a `CoroutineScope` | built into the runtime |
| Structured concurrency | `asyncio.TaskGroup` (3.11+), bolted on later | `coroutineScope` from the start | `async let` / `TaskGroup` from the start |

**The difference that bites:** Python does not force you to `await`. Calling an async function without
it yields a coroutine object that never executes — at best a `RuntimeWarning`, often silence. Kotlin's
compiler refuses to call `suspend` from non-suspend; Swift's refuses to omit `await`. In Python the
guardrail is `pyright`/`mypy`, not the interpreter.

All three languages *permit* blocking inside an async function — Kotlin wants
`withContext(Dispatchers.IO)`, Swift has no enforcement either. Python is simply where you're least
likely to be warned, which is why the source exercise's `async def invoke` calling the synchronous
`agent(...)` shipped without complaint.

---

## 9. `async` is not `concurrent` — in all three languages

A recurring conflation worth separating, because all three languages invite it:

- **`async`/`suspend` means "this function can yield control at an await point."** That's it. It buys the
  *runtime* the ability to do something else while this call waits on I/O. For a single request in
  isolation it makes nothing faster.
- **Concurrency is a separate, explicit act.** Python: `asyncio.gather`, `asyncio.TaskGroup`,
  `create_task`. Kotlin: `async {}` / `launch` inside a scope — `suspend` alone starts nothing.
  Swift: `async let`, `TaskGroup`, `Task {}`.

Kotlin makes the distinction clearest by naming them differently: `suspend` is the marker, `async {}` is
the concurrency builder. Python reuses the word `async` for the marker, which is precisely where the
confusion comes from.

**Why it mattered here:** the source exercise's `async def invoke` calling the *synchronous* `agent(...)`
produced correct answers in testing. It would have to: one request at a time, blocking the event loop
costs nothing observable. The failure mode is concurrent load — every other in-flight request stalls
behind the blocking call. That is why the bug is invisible locally and expensive in a deployed runtime.

Corollary for the AgentCore Runtime specifically: streaming (`agent.stream_async`) and concurrency are
also independent. Streaming is about *when* bytes reach the caller; the event loop is about *how many
callers* you can serve at once.

---

## 10. Async generators cannot `return` values — Python-only rule

```python
async def invoke(payload, context=None):
    async for event in agent.stream_async(user_message):
        yield event
    # return {"fallback": "message"}  # SyntaxError!
    # yield {"fallback": "message"}   # ✅ This works
```

**In Python:** once a function contains `yield`, it becomes a generator. Generators can only `return` 
without a value (to exit early) or `yield` values. `return value` inside a generator is a `SyntaxError`.

**Kotlin/Swift equivalent patterns:**
- **Kotlin:** `Flow { emit(x) }` — return vs emit behave the same way
- **Swift:** `AsyncStream { continuation.yield(x) }` — return vs yield behave the same way  

**Why Python is different:** generators have a special protocol where `return value` would be ambiguous 
with the iteration termination signal. The value would be accessible via `StopIteration.value`, but 
that's not the normal consumption path.

**Practical impact:** our `async def invoke` with `yield` cannot use `return {"fallback": "..."}` — 
it must be `yield {"fallback": "..."}`. This specific error came up during the Calculator Tool conversion 
and is worth remembering.

---

## 11. `A | B` is a union — and neither Swift nor Kotlin has one

*First seen: Pylance showing `hooks: list[HookProvider | HookCallback[Unknown]]`.*

`|` in a type hint means **either type**. `list[A | B]` is a list whose elements are each an `A` or a
`B`. It is not a mapping — a dict hint has a comma inside the brackets: `dict[str, int]`.

| | "either A or B" |
|---|---|
| Python | `A \| B` (3.10+), or `Union[A, B]` before |
| **Kotlin** | **no such construct** — `sealed class` hierarchy, or `Any` + smart casts |
| **Swift** | **no such construct** — `enum` with associated values, or a shared protocol |
| TypeScript | `A \| B`, identical |

This is a genuine gap in Swift/Kotlin intuition rather than a syntax difference. Kotlin's only
union-shaped feature is nullability: `String?` *is* `String | null`, but with dedicated syntax for that
one case instead of a general mechanism. Python spells it `str | None` and it's the same machinery as
any other union.

Why it matters here: `list[HookProvider | HookCallback]` is Strands telling you it accepts **either** a
class implementing the protocol **or** a bare callback function. Reading the union is how you know both
are allowed.

---

## 12. `Protocol` is *structural* typing — subclassing it is optional

*First seen: `class ShortTermMemoryHookProvider(HookProvider)`, which reads like nothing in Swift/Kotlin.*

Python has **no `interface` keyword**. `class Foo(Bar)` is the same syntax whether `Bar` is a concrete
class, an abstract base, or an interface-like protocol — you cannot tell from the class line, you have
to go look. (`inspect.getsource(Bar)` is how.)

`HookProvider` turned out to be neither a class nor an ABC:

```python
@runtime_checkable
class HookProvider(Protocol):
    def register_hooks(self, registry: "HookRegistry", **kwargs: Any) -> None:
        ...        # Ellipsis is the body — "no implementation"
```

**A `Protocol` is satisfied by shape, not by declaration.** Any class with a matching `register_hooks`
*is* a `HookProvider`, whether or not it says so. `class ShortTermMemoryHookProvider:` with no
parentheses would work identically.

| | How conformance happens |
|---|---|
| **Swift protocol** | **nominal** — you must write `: MyProtocol` |
| **Kotlin interface** | **nominal** — you must write `: MyInterface` |
| **Python `Protocol`** | **structural** — having the method *is* conforming |
| Go interface | structural |
| TypeScript interface | structural |

In Swift you *tell* the compiler you conform and it holds you to it. In Python the type checker
*observes* whether you happen to match — duck typing formalised after the fact (`Protocol` only arrived
in 3.8, long after the culture it describes).

**Still write the base class.** It documents intent, and — precisely because it's optional — it's the
only reason a type checker will warn you when your method drifts out of shape.

---

## 13. `None` is a value, not a type — and nothing is non-nullable

```python
type(None)            # <class 'NoneType'>
None is type(None)()  # True — there is exactly one None object
bool(None)            # False
```

`None` is the single instance of `NoneType`. `Any` is a *type*; `None` is a *value*. Different
categories, so it's neither of the obvious guesses.

| | Absence spelled | A value? | Compiler enforces? |
|---|---|---|---|
| **Python** | `None` | **yes**, one singleton | **no** |
| **Swift** | `nil` | no — `Optional<T>.none`, an enum case | yes |
| **Kotlin** | `null` | yes | yes |

The Swift contrast is the sharp one: `nil` isn't a value sitting in a variable, it's the `.none` case of
`Optional`, and `String` vs `String?` are genuinely different types. **Python has no non-nullable type.**
Every name can hold `None`, always:

```python
def f(x: str) -> str: return x
f(None)     # returns None. No error. None whatsoever.
```

`str | None` is the closest thing to `String?`, and it is purely advisory.

Three consequences used throughout this project:

- **`is None`, not `== None`** — one singleton means identity is the correct test, and the convention.
- **A function with no `return` returns `None`.** That's why the hooks' bare `return` guard clauses work,
  and why `register_hooks` is annotated `-> None`.
- **`None` is falsy** — which is what makes `if not actor_id or not session_id: return` work, and also
  why it catches `""` and `0`. Same trap as entry 5's `or` chain.

**The practical upshot** (learned the same morning we switched Pylance on): Kotlin and Swift put
correctness in the *language*, and you cannot opt out. Python puts it in *tooling you assemble yourself*.
A Python project with a type checker configured is a materially different thing from one without, and
"it runs" carries far less information here than it does in Kotlin.

Worth keeping straight, since the words get used interchangeably:

| Layer | Tool | Catches |
|---|---|---|
| Compile (automatic) | the interpreter | `SyntaxError`, `IndentationError` |
| Type check | Pylance/pyright, mypy | undefined names, use-before-definition, wrong arity, unassigned attributes |
| Lint | ruff, flake8, pylint | unused variables, style, suspicious patterns |
| **Run it** | tests | **data-shape mismatches — e.g. `h["city"]` against a dataset keyed `location`** |

That last row is why parity tests are the deliverable. No static tool in any language knows what keys
are in your JSON.

---

## 14. Python 3.14 made annotations *lazy* — an undefined name is now a deferred error

*First seen: the Gateway exercise, where `-> AsyncGenerator[dict, None]` with no import worked.*

```python
# no `from typing import AsyncGenerator` anywhere in the file
async def invoke(payload: dict, context=None) -> AsyncGenerator[dict, None]:
    ...
```

```
import main            -> OK, no error
get_type_hints(invoke) -> NameError: name 'AsyncGenerator' is not defined
                          File "main.py", line 68, in __annotate__
```

PEP 649, new in **3.14**: instead of evaluating an annotation when the `def` executes, Python stores
a thunk (note the `__annotate__` frame) and evaluates it only when something asks. So a broken
annotation is no longer an import-time failure.

This refines entry 2. Hints are still live runtime data — that is what makes `@tool` schema
generation possible — but they are now *lazily* live. The distinction matters twice:

- **Things do ask.** `typing.get_type_hints()` is exactly how Strands derives a tool's JSON schema
  from its signature. A tool with a broken annotation fails at registration, not at import.
- **It is version-dependent.** On 3.13 and earlier the same line raises immediately. The venvs here
  are 3.14.6 while the exercise Lambdas deploy on 3.12 — so this class of bug can pass locally and
  fail at import in production. That is the worst available failure shape.

| | When is an annotation checked? |
|---|---|
| **Kotlin** | compile time, always, and it is binding |
| **Swift** | compile time, always, and it is binding |
| **Python ≤3.13** | evaluated at `def` time (a name error surfaces), but never *enforced* |
| **Python 3.14+** | **evaluated only on demand**, and still never enforced |

So Python has moved *further* from the Kotlin/Swift model, not closer: annotations are now
lazily-evaluated, unenforced metadata. Which makes a type checker in CI less optional, not more —
`pyright`/`mypy` read them statically and do not care whether the interpreter ever evaluates them.

## 15. Format strings are *data*, so a missing placeholder is a runtime `KeyError`

Seen in the long-term memory exercise, where namespace templates come back from an API call:

```python
namespace = "/summaries/{actorId}/{sessionId}"      # a plain str, fetched at runtime
namespace.format(actorId="alice-001")               # KeyError: 'sessionId'
```

`str.format` resolves placeholders against its keyword arguments **when called**. The template is an
ordinary string that arrived over the network, so nothing could have checked it earlier.

**Swift and Kotlin have no equivalent failure.** Interpolation is part of the syntax, compiled in place:

```kotlin
val ns = "/summaries/$actorId/$sessionId"   // sessionId must be in scope — compile error otherwise
```

```swift
let ns = "/summaries/\(actorId)/\(sessionId)"
```

You cannot interpolate a name that isn't in scope, and you cannot interpolate *into* a runtime string at
all — you'd reach for `replacingOccurrences` or a regex, which fails visibly rather than throwing.
Python's f-strings behave like the Swift/Kotlin case; `.format()` on a variable is the divergent one.

**Why it mattered:** the exercise wraps the call in `except Exception`, so the `KeyError` became a log
line and an empty retrieval. A template shape change three services away turned into "the agent doesn't
remember things."

## 16. A class is a first-class value, used here as a dispatch key

```python
registry.add_callback(MessageAddedEvent, self._retrieve_travel_context)
registry.add_callback(AfterInvocationEvent, self._save_interaction)
```

The first argument is **the class object itself** — not an instance, not a string. Strands stores it as
a dictionary key and looks it up when emitting. Classes are ordinary runtime objects in Python, so this
needs no reflection machinery.

The cost is that nothing relates the two arguments. Both are just objects, so this registers fine:

```python
registry.add_callback(AgentInitializedEvent, MessageAddedEvent)   # two event classes, no handler
```

**Kotlin** would express this with a reified generic, and the handler's parameter type is checked:

```kotlin
inline fun <reified E : Event> HookRegistry.on(noinline handler: (E) -> Unit)
registry.on<MessageAddedEvent> { event -> retrieveTravelContext(event) }
```

**Swift** would use a generic over the event type:

```swift
func addCallback<E: Event>(_ type: E.Type, _ handler: @escaping (E) -> Void)
registry.addCallback(MessageAddedEvent.self, retrieveTravelContext)
```

Note Swift needs `.self` to pass a metatype, and its `E.Type` ties the handler's parameter to the same
`E` — the pairing the Python version leaves to convention. Python can approximate this with
`Callable[[E], None]` and a `type[E]` overload, but only a type checker enforces it; at runtime the two
arguments remain interchangeable objects.

## 17. Duplicate keys in a dict comprehension are silently last-wins

```python
{s["type"]: s["namespaces"][0] for s in strategies}     # two SEMANTIC strategies → one entry
```

No error, no warning. The later item overwrites the earlier one.

**Kotlin's `associate` behaves identically**, so this is not a Python peculiarity in general — the sharp
contrast is **Swift**, which makes you choose:

```swift
Dictionary(uniqueKeysWithValues: pairs)          // TRAPS at runtime on a duplicate key
Dictionary(pairs, uniquingKeysWith: { a, b in b })   // explicit: last wins
```

Swift has no way to build a dictionary from pairs *without* stating which behaviour you want. Python and
Kotlin default to overwrite, and "keys are unique" stays an assumption you carry yourself.

**Why it mattered:** the exercise's `get_namespaces` would silently drop a namespace in any project with
two strategies of the same type — a capstone with separate semantic strategies for policy and trip
history loses one at startup, with no signal.

## 18. `or` coalesces on *truthiness*, so it covers "missing **or** empty"

From the capstone's `get_namespaces`, reading a field that is sometimes absent and sometimes an
empty list:

```python
templates = strategy.get("namespaceTemplates") or strategy.get("namespaces") or []
```

One expression handles three cases: key absent, key present but `[]`, and key present with values.
That works because Python's `or` returns the **first truthy operand** — and `[]`, `{}`, `""`, `0`
and `None` are all falsy.

Contrast the obvious-looking alternative, which is subtly wrong:

```python
templates = strategy.get("namespaceTemplates", strategy.get("namespaces", []))
```

`dict.get`'s default fires only when the key is **missing**. A key present with `[]` returns `[]`
and never consults the fallback — the exact case we needed to handle, since the API populates one
field or the other depending on version.

**Neither Swift nor Kotlin can express this with their coalescing operator**, because both are
strictly *null*-coalescing rather than truthiness-based:

```swift
let templates = strategy.namespaceTemplates ?? strategy.namespaces ?? []
// ?? only fires on nil. An EMPTY array is not nil, so [] short-circuits here
// and the fallback is skipped — a silent wrong answer.
```

```kotlin
val templates = strategy.namespaceTemplates ?: strategy.namespaces ?: emptyList()
// elvis is null-only too. Same bug.
```

Getting the equivalent there means saying "empty" out loud:

```kotlin
val templates = strategy.namespaceTemplates?.takeIf { it.isNotEmpty() }
    ?: strategy.namespaces?.takeIf { it.isNotEmpty() }
    ?: emptyList()
```

```swift
let templates = [strategy.namespaceTemplates, strategy.namespaces]
    .compactMap { $0 }.first { !$0.isEmpty } ?? []
```

**The trade-off, stated fairly.** Python's version is shorter and handles a real API inconsistency in
one line. It is also less precise: `or` cannot distinguish "absent" from "empty" from `0`, so
`count = user.get("count") or 10` silently turns a legitimate `0` into `10`. Swift and Kotlin force
you to name which emptiness you mean, and that verbosity is exactly what stops that bug.

Rule of thumb: reach for `or` when every falsy value should take the fallback — collections you are
about to iterate, strings you are about to print. Use an explicit `is None` check when `0`, `False`
or `""` are legitimate values.

Related: [[13]] — `None` is a value, not a type.

---

## 19. A loop/`if` variable is function-scoped, not block-scoped

Hit while wiring the lesson-1 coordinator: a `cmd_result` line referencing `issue_type` was pasted
*above* the `for issue in issues:` loop that assigns it. Pylance flagged **`issue_type` is possibly
unbound**.

In Python a name assigned *anywhere* in a function belongs to the whole function, but it is only
**bound** when its assignment line actually runs. So a name assigned inside a loop or an `if` block:

- **leaks out** — it is still readable after the block, holding the last value the loop set;
- but is **undefined if the block never ran** — reading it then is an `UnboundLocalError` at runtime,
  which Pylance approximates statically as "possibly unbound."

```python
for issue in issues:
    issue_type = issue.get("issue")   # assigned only if issues is non-empty
print(issue_type)                     # NameError if issues was [] — the loop never bound it
```

Kotlin and Swift are **block-scoped**: a `val`/`let` declared inside a `for` or `if` exists *only*
inside those braces. Referencing it outside isn't "possibly unbound" — it's a hard compile error,
`unresolved reference`, because the name does not exist out there at all. The variable cannot leak,
so the whole class of "used before the branch that defines it" bug is unrepresentable.

```kotlin
for (issue in issues) { val issueType = issue.type }
println(issueType)   // unresolved reference: issueType — won't compile
```

**The divergence that matters:** Python's function scope means placement (what runs before what) is
load-bearing in a way it isn't in Kotlin/Swift, where lexical position alone settles visibility. The
fix is never "declare it higher" as you would to widen a Kotlin scope — it is to put the *use* inside
the same block as the *assignment*.

What caught it: Pylance's "possibly unbound" in the Problems panel — the same read-the-Problems-panel
habit that catches `reportRedeclaration`. The terminal would only have failed at runtime, and only on
the healthy-device path where the loop body never runs.

---

## 20. String interpolation is opt-in per literal (`f"..."`), not always-on

Came up wiring lesson-4 prompts: `f"Generate label for package {package_id}"` vs the same line without
the `f`. Kotlin and Swift interpolate *every* string literal; Python interpolates only literals prefixed
with `f`. A plain Python string treats `{package_id}` as literal characters.

| | Simple variable | Expression | Always on? |
|---|---|---|---|
| **Kotlin** | `"$cans"` | `"${cans.size}"` | yes |
| **Swift** | `"\(cans)"` | `"\(cans.count)"` | yes |
| **Python** | `f"{cans}"` | `f"{len(cans)}"` | **no — needs the `f`** |

All three evaluate arbitrary expressions inside the delimiter, not just names — Python's `{len(cans)}`
is Kotlin's `${cans.size}`. Two Python specifics:

- **The `f` is mandatory.** No prefix → no interpolation → literal braces in the output.
- **Braces are never optional.** Kotlin's bare `"$name"` shorthand has no Python equivalent; it's always
  `f"{name}"`. A `$` in a Python string is just a dollar sign.

**The failure mode is silent, and inverted from the reader's home language.** Forgetting the `f` does not
raise — `"...{package_id}"` is a perfectly valid `str`, so a function typed `prompt: str` accepts it
happily and the model receives the literal text `{package_id}`. A Kotlin/Swift dev's muscle memory
("interpolation just happens") produces a bug that only shows up in the *output*, never as an error. The
tell for when you need `f`: is there a `{value}` to substitute at runtime? If the string is fixed, skip
the `f`; the moment a variable goes in, it's required.

---

## 21. `assert` is not `guard` — it can be compiled out; use `if not x: raise` for real checks

*First seen: food-delivery shared-state exercise (Module 6).*

`assert cond` looks like Swift's `guard cond else { … }` — both sit at the top, both let the type-checker
narrow afterward (`assert state is not None` → Pylance treats `state` as non-`None` below, exactly like
`guard let`). But three structural differences make them **not** interchangeable:

| | Failure handling | Binds a value? | Live in prod? |
|---|---|---|---|
| **Swift `guard`** | forces an `else` that exits scope (compiler-enforced) | yes — `guard let x = …` unwraps + names | always |
| **Swift `precondition`** | traps | no | always (even release) |
| **Python `assert`** | raises `AssertionError`, no branch | no | **NO — stripped under `python -O`** |
| **Python `if not x: raise`** | you choose the exception + message | no (rebinds nothing) | always |

The gotcha with no analog in Swift/Kotlin: **`assert` vanishes when Python runs optimized (`-O`).** So a
value you actually depend on, guarded only by `assert`, is unguarded in production. `assert` is therefore
for *documented invariants and test sanity-checks you accept being compiled out* — not validation.

**Rule:** anything runtime correctness depends on → `if not x: raise`. It survives `-O`, lets you pick the
exception/message, and narrows the type for Pylance identically (a `raise`/`return` inside the `if` narrows
the rest of the function). Reserve `assert` for the narrow niche where you *want* it stripped in prod.

The nearest true `guard` equivalent in Python is `if not x: raise/return` — which is also what the course's
own tool bodies use (`if not order: return …`). Matching it keeps one pattern across the file.

---

## 22. In-place mutating methods return `None`, not the collection

*First seen: ecommerce-checkout saga (Module 7) — `completed_steps.reverse()`.*

Python splits "rearrange this collection" into two forms, and the mutating one returns `None`:

```python
items.reverse()              # in-place — mutates items, returns None
items.sort()                 # in-place — mutates items, returns None
items.append(x)              # in-place — mutates items, returns None
new = list(reversed(items))  # non-mutating — returns a new sequence
new = sorted(items)          # non-mutating — returns a new list
```

**The silent bug:** `x = items.reverse()` sets `x = None` while reversing `items` as a side effect. No
error — `None` is a valid value — so it surfaces later as an `AttributeError`/`TypeError` far from the
cause, or as a function that quietly returns nothing. The whole `list`-mutator family
(`reverse`, `sort`, `append`, `extend`, `insert`, `remove`, `clear`) returns `None` by design.

**Rule:** if you want a *result*, use the non-mutating builtin (`sorted`, `reversed`); if you want to
*change the collection in place*, call the method as a statement and don't assign its return.

| | Mutate in place | Return a new/reordered sequence |
|---|---|---|
| **Python** | `items.reverse()` / `items.sort()` → `None` | `reversed(items)` (iterator) / `sorted(items)` (list) |
| **Kotlin** | `mutableList.reverse()` / `.sort()` | `list.reversed()` / `.sorted()` → new `List` |
| **Swift** | `array.reverse()` / `.sort()` | `array.reversed()` (view) / `.sorted()` → new `Array` |

Kotlin and Swift make the same mutate-vs-return split by name — but their *returning* variants hand back
a usable collection, whereas Python's mutators hand back `None`. The muscle memory to unlearn: in Python
you cannot chain off a mutator (`items.sort().first()` fails), and assigning its result is almost always
a mistake.

## 23. Variables/functions are `snake_case`, not `camelCase`

PEP 8 (Python's official style guide) prescribes `snake_case` for variables and functions, and this is
near-universal in real Python — not optional like a linter preference. `coordinatorModel` runs fine but
reads as "written by someone from Swift/Kotlin"; the idiom is `coordinator_model`.

| | variables / functions | classes | constants |
|---|---|---|---|
| **Python** | `snake_case` | `PascalCase` | `UPPER_SNAKE` |
| **Swift / Kotlin** | `camelCase` | `PascalCase` | `camelCase` / `UPPER_SNAKE` |

Only the variable/function row diverges — classes agree (`PascalCase`). This is the single most visible
"non-native speaker" tell in Python code, because it shows up on nearly every line.

**The rule that actually matters more than PEP 8:** *consistency within the file wins.* The real cost of a
stray `coordinatorModel` isn't that it breaks a rule — it's that it sits next to `retriever_model`,
`system_prompt`, `guardrail_id` and reads as a seam where two authors (or two languages) met. Match the
surrounding code first; reach for PEP 8 only when there's no local precedent. (Caught in the capstone:
a reviewer skimming the file would flag the lone camelCase before any logic.)

## 24. Three concurrency models — and `async/await` is NOT the default one

Swift/Kotlin give you basically one concurrency story (`async`/`await` + `suspend` coroutines) for I/O.
Python has three, and picking wrong is a real cost:

| Python | Mechanism | Swift/Kotlin analog | Use when |
|---|---|---|---|
| `ThreadPoolExecutor` / `threading` | OS threads, preemptive | `Dispatchers.IO` / GCD queue | I/O-bound, **small N** |
| `ProcessPoolExecutor` / `multiprocessing` | separate processes, real parallelism | — (no close analog) | **CPU-bound** |
| `asyncio` + `async`/`await` | one thread, cooperative event loop | **`async/await` / coroutines** | I/O-bound, **large N** |

The traps that have no Swift/Kotlin equivalent:
- **The GIL.** Python threads can't run bytecode in parallel, so threads only help *I/O-bound* work (idle-waiting on the network). CPU-bound → you need *processes*, not threads. (Py 3.13 has an experimental no-GIL build; not default yet.)
- **`asyncio` is all-or-nothing.** It only helps if libraries are async *down to the socket*. `boto3` is **synchronous**, so asyncio+AWS needs `aioboto3`/`aiobotocore`; one blocking call in an async function stalls the whole loop. Swift/Kotlin don't have this "sync library poisons the event loop" problem to the same degree.

**The decision rule:** small N + blocking libraries → **threads** (what the capstone's 3-KB parallel RAG used — correct); large N + async libraries → **asyncio**; CPU-bound → **processes**. So unlike Swift/Kotlin, reaching for `async/await` first is often the *wrong* instinct in Python — `ThreadPoolExecutor` is the pragmatic default for small-N blocking I/O. (See `multi-agent-patterns.md`
parallelization layers and the `lesson-stack-runbook.md` async-provisioning note: polling vs waiter vs event-driven.)

## 25. Pydantic is the typed-data-model layer Swift/Kotlin bake into the language

Swift/Kotlin give you typed structs/data classes with free `Codable`/serialization and compile-time
guarantees. Python's dynamic typing + optional hints don't enforce anything at runtime — so the ecosystem
standardized on **Pydantic**: define a model with typed fields, and it *validates and coerces at runtime*
on construction. It's the de-facto standard (FastAPI, LangChain, the Agents SDKs all lean on it).

Why it matters for data/DB code specifically: the capstone's DynamoDB friction — numbers coming back as
`Decimal`, the `json.dumps(..., default=str)` coercion dance, no schema on the dict you read — is exactly
what a Pydantic model dissolves (declare `price: float`, let it coerce; `.model_dump_json()` serializes
cleanly). It's the closest Python gets to a Swift `struct ... : Codable`.

**The cultural divergence underneath (worth internalizing):** Swift/Kotlin are *framework-first* — reach for
the structured tool early. Python is *stdlib-until-it-hurts* — a dependency has real cost, and the
boto3/`dict` baseline is often enough; you add Pydantic/PynamoDB/SQLAlchemy when the toil *scales*, not
reflexively. Same judgment as #24: the raw tool is right until the repetition grows, then a layer earns
its keep. (If continuing with Python, Pydantic is the single highest-ROI thing to learn next — it retrofits
the type-safety muscle memory you already have from Swift/Kotlin.)
