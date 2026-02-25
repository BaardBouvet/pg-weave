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

### Additional obvious candidates (from DTL list-function patterns)

Good near-term candidates:

- `NTH index OF expr` — element-at with explicit index semantics
- `DISTINCT OF expr` — deduplicated list while preserving first occurrence order

Set-algebra candidates (strong fit, likely v1.2 scope):

- `INTERSECTS left WITH right` — boolean overlap test
- `INTERSECTION OF left WITH right` — common values, preserving `left` order
- `DIFFERENCE OF left WITH right` — values from `left` not present in `right`, preserving `left` order
- `UNION OF left WITH right` — first-seen unique values from `left` then `right`

Likely defer (higher semantic/typing complexity):

- `FILTER expr BY predicate`
- `ANY OF expr SATISFIES predicate` / `ALL OF expr SATISFIES predicate`
- `SLICE expr FROM start [TO end] [STEP stride]`
- `SUM OF expr`, `MIN OF expr`, `MAX OF expr` (requires strong element-typing and coercion rules)
- Full set-containment helpers (`SUBSET OF`, `SUPERSET OF`) until null/equality semantics are formalized

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
- Set helpers require explicit equality/null rules and order-preservation guarantees

Potential SQL lowering hints for the next review pass:

- `IS EMPTY expr` / `IS NOT EMPTY expr` → length-check lowering (`jsonb_array_length` vs `cardinality`)
- `FIRST OF expr` / `LAST OF expr` → first/last element extraction by type-aware lowering
- `NTH index OF expr` → index-based extraction with out-of-range → `null`
- `DISTINCT OF expr` → deduplication preserving stable order
- `INTERSECTS left WITH right` → overlap test by value equality
- `INTERSECTION` / `DIFFERENCE` / `UNION` → ordered set-style operations with deterministic output order
