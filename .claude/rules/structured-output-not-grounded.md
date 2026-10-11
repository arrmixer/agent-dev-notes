# Structured output ≠ grounded output

A schema-valid output is not a true one. A schema (JSON Schema, Pydantic, fixed sections) guarantees shape,
not that the content is supported by evidence. A model handed a template fills every slot — including the
one with no backing — because the structure demands a value there.

Validate grounding separately from the schema: require evidence per claim (citations / source IDs / the
tool result each field came from), and instruct the model to say "not specified" rather than backfill. Test
with an input where part of the schema has no supporting evidence — a correct system leaves it empty, a
broken one fabricates to fill the shape.
