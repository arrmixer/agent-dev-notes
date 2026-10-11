# Raw tool until the toil scales

Reach for the raw/stdlib tool first; add an abstraction layer (ORM, framework, async model) only when the
repetitive toil actually scales. The signal to add a layer is the same manual friction repeating across many
sites — not aesthetics, not "a framework exists." A dependency has real cost; pay it when it earns its keep.

Both failure modes are real: premature abstraction (carry/fight a dependency you didn't need) and too-late
abstraction (copy-pasted toil rots everywhere). The tell is measured repetition. Right tool for N=3 is often
the wrong tool for N=30 — name the N where you'd switch, don't default to either end.
