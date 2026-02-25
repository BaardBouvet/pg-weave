# ADR 016: Nested Output Defaults — JSONB Without Output Schema

**Status:** Accepted  
**Date:** 2026-02-25

## Context

pg_weave should work out-of-the-box for schemaless data with minimal declarations.
`COLLECT` and `MAP` produce nested outputs, which can be represented as JSONB or typed arrays.
The DSL should avoid extra inline syntax noise for common cases.

## Decision

- If output schema is **not explicitly defined**, nested outputs from `COLLECT`/`MAP` default to **JSONB**.
- If output schema **is defined**, nested output mode is inferred from declared field types:
  - `jsonb` fields use JSON aggregation
  - `some_type[]` fields use typed aggregation
- Scalar/top-level `SET` fields continue to use expression/type inference rules from ADR 010.
- `WITH` remains input-side JSONB validation/casting only (ADR 010); it is not replaced by this ADR.

## Rationale

- Keeps schemaless-first UX: definitions run without upfront output contracts.
- Keeps common DSL syntax concise: no mandatory inline output-mode modifiers.
- Allows stronger typing when teams opt into explicit output schema.
- Preserves separation of concerns: input validation (`WITH`) vs output contract (`SCHEMA`/`TYPE`).

## Consequences

- Bare `COLLECT`/`MAP` nested outputs compile to JSONB by default.
- Declared output schema can switch specific nested fields to typed arrays.
- Compiler must choose aggregation strategy per target field type (`jsonb_agg` vs typed `array_agg`).
