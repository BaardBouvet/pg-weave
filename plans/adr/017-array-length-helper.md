# ADR 017: Polymorphic Array-Length Helper

**Status:** Accepted  
**Date:** 2026-02-25

## Context

Examples that compute array length currently expose PostgreSQL-specific functions (`jsonb_array_length` vs `cardinality`) depending on whether the value is JSONB or a typed PG array. This leaks storage details into the DSL surface and hurts readability.

## Decision

Introduce a DSL-level array-length helper with polymorphic lowering.
This ADR covers array-length semantics only; general syntax-style policy is defined in ADR 008.

Working form in docs/examples:
- `COUNT(array_expr)`

Compiler lowering:
- JSONB array expression → `jsonb_array_length(...)`
- Typed PG array expression → `cardinality(...)`

Naming options were evaluated (`SIZE`, `LENGTH`, `COUNT`), and the accepted form is `COUNT(array_expr)`.

## Rationale

- Hides backend-specific function differences from authors
- Keeps examples readable and intent-first
- `COUNT(...)` reads much better in real transforms than backend-specific alternatives (`jsonb_array_length`, `cardinality`) and than less natural DSL names
- Reduces accidental coupling to JSONB internals
- Supports mixed-schema projects consistently

Naming considerations:
- `COUNT` aligns with "number of elements" language and reads naturally in examples
- In side-by-side examples, `COUNT(orders)` is immediately understandable to SQL users and keeps the DSL visually clean
- `LENGTH` may be confused with string length semantics
- `COUNT` can be confused with aggregate row counting, so parser/grammar must explicitly describe the array-helper form

## Consequences

- Parser/compiler must recognize `COUNT(...)` helper form and lower by resolved type
- Type resolution errors should report whether value is non-array
- Parser disambiguation rules are required in grammar/docs to distinguish helper form from SQL aggregate form
