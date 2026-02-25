# ADR 018: Polymorphic Collection Helpers (Beyond COUNT OF)

**Status:** Proposed  
**Date:** 2026-02-25

## Context

`COUNT OF expr` (ADR 017) hides JSONB-vs-typed-array differences for a common operation. Similar friction appears for emptiness checks and element access, where SQL differs by physical representation.

## Decision

Evaluate a minimal helper set for v1.1, with storage-neutral semantics:

- `IS EMPTY expr`
- `IS NOT EMPTY expr`
- `FIRST OF expr`
- `LAST OF expr`

`COUNT OF expr` remains the established baseline from ADR 017.

## Rationale

- Keeps DSL intent-focused instead of exposing backend-specific function/operator choices
- Avoids repetitive `COUNT OF expr = 0` patterns
- Reduces accidental coupling to JSONB internals in user-facing transforms
- Lets authors write one expression style regardless of array storage model

## Consequences

- Compiler must lower each helper by resolved type (JSONB array vs typed PG array)
- Helper semantics must be defined for empty arrays (especially `FIRST OF` / `LAST OF`)
- Error messages must clearly explain non-array misuse
- This ADR is intentionally scoped to read-only helpers; quantifiers (`ANY OF` / `ALL OF`) and reducers are deferred
