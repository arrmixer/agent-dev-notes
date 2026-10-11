# Parallelize: gate → decompose → aggregate

Parallelism is a decision with a precondition, not a free speedup. Gate first: are the subtasks independent
(no shared mutable state, no ordering dependency), side-effect-free or idempotent, and worth the overhead at
this N? If any "no," don't parallelize. Then decompose along the independent axis, then aggregate with the
merge that split implies (dedup / rank / top-K / labeled concat).

Naive parallelism on dependent or side-effecting work races and corrupts silently. For small-N blocking I/O
use a thread pool; reserve asyncio for large-N with async-native libraries.
