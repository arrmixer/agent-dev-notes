# A tool description is functional text, not docs

Write a tool's description for the model's routing decision — WHEN to call it and WHAT it returns — not as
human documentation. It's the only thing the model reads to choose the tool; a vague description causes
mis-selection, and that's a description bug, not a code bug.

Framework quirk to check: some SDKs (e.g. Strands) collapse the whole docstring into one description field,
so an `Args:` block gives the model no per-parameter schema — put format constraints in natural prose, and
convey optional-ness via Python defaults. The return format is part of the contract (downstream consumes it):
never carry meaning only in emoji/color/layout that can be silently stripped.
