# ADR 020: WITH Clause — Optional JSONB Input Shape Validation

**Status:** Proposed  
**Date:** 2026-02-26

## Context

JSONB columns are opaque to the compiler — `pg_catalog` knows a column is `jsonb` but nothing about its internal structure. Any nested path is accepted and type errors only surface at PostgreSQL execution time. A `WITH` clause on `FROM` could declare expected JSONB structure so the compiler can validate paths and infer types at compile time.

## Decision

Introduce an optional `WITH` clause on `FROM`:

```sql
FROM orders AS o
  WITH data { items [{ price numeric, qty integer }], notes text }
{
  SET total = SUM(MAP o.data.items AS item -> item.price * item.qty)
}
```

- `WITH` declares nested JSONB structure for compile-time path/type validation.
- Without `WITH`, JSONB access remains flexible (current default behavior).
- `WITH` is input-side only — it does not affect output typing (`INTO` handles that).

## Rationale

- Catches misspelled field names and container mismatches before query execution.
- Auto-casts JSONB text extractions to declared types — without `WITH`, authors must manually cast every JSONB field (e.g. `item.price::numeric`); with `WITH`, the compiler generates casts from the declared shape.
- Fully optional — no impact on users who prefer schemaless JSONB access with explicit casts.

## Consequences

- Adds grammar rules (`with_clause`, `json_shape`, `json_field`) to the parser.
- Compiler must validate referenced JSONB paths against declared shape when present.
- Does not change output behavior or interact with `INTO`.
