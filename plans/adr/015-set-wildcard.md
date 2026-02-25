# ADR 015: SET * — Wildcard Column Copy

**Status:** Proposed (needs review)  
**Date:** 2026-02-22

## Context

When a source table has 50+ columns and you only need to transform 3, listing every column with `SET` is painful. SQL solves this with `SELECT *`. The DSL needs an equivalent.

## Decision

`SET *` copies all columns from the `FROM` source. It is expanded at compile time using the introspected (or declared) schema — the generated SQL always uses explicit column references, never `SELECT *`.

### Basic usage

```
-- Copy all columns, override status
FROM orders AS o {
  SET *,
  SET status = 'processed'
}
```

### EXCEPT — exclude columns

```
-- Copy all except internal columns
FROM orders AS o {
  SET * EXCEPT (internal_notes, debug_flag),
  SET status = 'processed'
}
```

### Semantics (order-independent)

`SET *` is declarative — explicit `SET` always wins, regardless of position:

```
-- These two produce identical output
FROM orders AS o {
  SET *,
  SET status = 'processed'
}

FROM orders AS o {
  SET status = 'processed',
  SET *
}
```

Resolution rules:
1. Start with all columns from the `FROM` source
2. Remove any listed in `EXCEPT (...)`
3. Replace any that have an explicit `SET` (explicit wins)
4. Add any new `SET` fields not in the source

### Compilation

`SET *` is purely a compile-time expansion. The compiler knows the source columns from `pg_catalog` (PG extension) or `SCHEMA` (dbt plugin mode) and emits explicit column references:

```
-- DSL
FROM orders AS o {
  SET * EXCEPT (internal_notes),
  SET status = 'processed'
}

-- Generated SQL (orders has: id, customer_id, status, amount, internal_notes)
CREATE VIEW enriched_orders AS
SELECT
  o.id,
  o.customer_id,
  'processed' AS status,
  o.amount
FROM orders o;
```

### Scope

- `SET *` always refers to the `FROM` source — not LOOKUP aliases
- To copy fields from a LOOKUP source, use explicit `SET` statements
- Works at top-level only — not inside COLLECT, MAP, or FLATMAP blocks (those construct new objects, not passthrough)

## Rationale

- **Mirrors `SELECT *`** — SQL users recognize it instantly
- **No new keyword** — reuses `SET` with wildcard, consistent with the language
- **Order-independent** — explicit `SET` always wins; no positional override rules
- **Compile-time expansion** — generated SQL is always explicit columns, better for IVM and debugging
- **DDL-safe** — event trigger re-expands when source columns change

## Consequences

- Compiler must resolve `SET *` against the schema before SQL generation
- `SET *` on a table with JSONB columns copies the column as-is (not its inner fields)
- Adding a column to the source table automatically includes it in the VIEW (via DDL trigger re-expansion)
- `EXCEPT` field names are validated against the schema — typos are compile-time errors
