# ADR 016: Nested Output Defaults — JSONB Without Output Schema

**Status:** Accepted  
**Date:** 2026-02-25

## Context

pg_weave should work out-of-the-box for schemaless data with minimal declarations.
`COLLECT` and `MAP` produce nested outputs, which can be represented as JSONB or typed arrays.
The DSL should avoid extra inline syntax noise for common cases.

## Decision

- `COLLECT`/`MAP` without `INTO` → nested output defaults to **JSONB**.
- `COLLECT`/`MAP` with `INTO type[]` → nested output uses **typed array aggregation** against the named PG composite type.
- The `INTO` target is a standard PG composite type (`CREATE TYPE ... AS (...)`).
- The compiler resolves `INTO` targets from `pg_catalog` (PG extension) or project declarations (dbt plugin).
- Scalar/top-level `SET` fields continue to use expression/type inference rules from ADR 010.

## Rationale

- Keeps schemaless-first UX: definitions run without upfront output contracts.
- `INTO` is local to the point of use — output mode is visible where it matters.
- Reuses PG composite types instead of inventing a parallel type system.
- Composite types are sharable across weaves without redeclaring.
- Preserves separation of concerns: catalog introspection for input vs `INTO` for output typing.

## Consequences

- Bare `COLLECT`/`MAP` nested outputs compile to JSONB by default.
- `INTO type[]` switches a specific nested block to typed array aggregation.
- Compiler must choose aggregation strategy based on `INTO` presence (`jsonb_agg` vs typed `array_agg`).

Compiler-lowering quick view:

- No `INTO` → JSON construction + `jsonb_agg(...)`
- `INTO some_type[]` → typed row construction + `array_agg(ROW(...)::some_type)`
