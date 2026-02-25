# ADR 017: Polymorphic Array-Length Helper

**Status:** Accepted  
**Date:** 2026-02-25

## Context

Examples that compute array length currently expose PostgreSQL-specific functions (`jsonb_array_length` vs `cardinality`) depending on whether the value is JSONB or a typed PG array. This leaks storage details into the DSL surface and hurts readability.

## Decision

Introduce a DSL-level array-length helper with polymorphic lowering.
This ADR covers array-length semantics only; general syntax-style policy is defined in ADR 008.

Working form in docs/examples:
- `COUNT OF array_expr`

Compiler lowering:
- JSONB array expression → `jsonb_array_length(...)`
- Typed PG array expression → `cardinality(...)`

Naming options were evaluated (`SIZE`, `LENGTH`, `COUNT`) and expression-shape alternatives. The accepted form is `COUNT OF array_expr`.

## Rationale

- Hides backend-specific function differences from authors
- Keeps examples readable and intent-first
- `COUNT OF ...` reads naturally in real transforms while avoiding SQL function-call ambiguity
- Reduces accidental coupling to JSONB internals
- Supports mixed-schema projects consistently

Naming considerations:
- `COUNT` aligns with "number of elements" language and reads naturally in examples
- In side-by-side examples, `COUNT OF orders` is immediately understandable and visually close to other DSL keyword forms
- `LENGTH` may be confused with string length semantics
- `COUNT OF ...` avoids overlap with SQL aggregate `COUNT(...)` call syntax

## Consequences

- Parser/compiler must recognize `COUNT OF ...` helper form and lower by resolved type
- Type resolution errors should report whether value is non-array
