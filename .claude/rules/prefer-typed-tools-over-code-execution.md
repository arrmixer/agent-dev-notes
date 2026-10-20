# Prefer typed tools; justify a code-execution sandbox

Default to a plain, typed `@tool` function. Add a code-execution sandbox / remote executor only when you can
name why: untrusted code (LLM/user-authored), dynamic code (assembled from runtime data), a provable
per-call isolation boundary, or capabilities the runtime lacks. Otherwise it's latency + serialization for
nothing. Prefer tool-authors-code-with-typed-args over LLM-authors-code (the model supplies values, not
arbitrary code).

When you must run generated code: isolate inputs at ONE interpolation point (`f"INPUT={json.dumps(x)}\n" +
STATIC_CODE`, like parameterized SQL); verify locally first, asserting against values derived independently
from the source data (not from the code's own output); and pair it with an in-process fallback that returns
a degraded-but-correct result flagged `degraded: true`. If that fallback is trivial, the sandbox wasn't load-bearing.
