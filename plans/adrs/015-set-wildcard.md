# ADR 015: SET * — Wildcard Column Copy

**Status:** Proposed (needs review)  
**Date:** 2026-02-22

## Context

When a source table has 50+ columns and you only need to transform 3, listing every column with `SET` is painful. SQL solves this with `SELECT *`. The DSL needs an equivalent.

## Decision

`SET *` copies all columns from the `FROM` source. It is expanded at compile time using the introspected (or declared) schema — the generated SQL always uses explicit column references, never `SELECT *`.

### Escaping literal `*` field names

`*` is reserved as the wildcard token in `SET *`. To target a literal field named `*`, quote it:

```
FROM orders AS o {
  SET *,
  SET "*" = 'special'
}
```

This implies `SET` targets accept quoted identifiers in addition to bare identifiers.

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

### Analysis: `SET *` inside `MAP` / `COLLECT` blocks

This was considered and deferred for now.

Why it is tricky:
- In nested blocks, `SET *` would have to refer to the current element/object, not the outer `FROM` row.
- Many `MAP`/`COLLECT` inputs are JSONB with unknown keys at compile time.
- Unknown keys conflict with deterministic schema inference and typed `INTO` outputs.

If introduced later, a consistent rule would be:
1. `SET *` in a nested block means “copy all fields from the current block input object”.
2. For typed/composite inputs, compile-time expansion is allowed.
3. For JSONB-object inputs, allow only schemaless output (no `INTO`), lowered as object merge where explicit `SET` wins.
4. For scalar inputs, raise a compile-time error (`SET *` has no object fields to copy).

Current stance: keep `SET *` top-level only in this ADR. If nested wildcard copying proves necessary, address it in a dedicated follow-up ADR with explicit JSONB and typing constraints.

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

## Optional extension: pattern-based `SET` selection

Potential future syntax:
- DTL-like wildcard patterns: `SET * MATCH (meta_*, *_tmp)`
- Regex patterns: `SET * MATCHES ('^meta_.*$', '.*_tmp$')`

Pros:
- Reduces boilerplate on wide schemas with naming conventions
- Mirrors DTL familiarity for migration users
- Keeps projection intent declarative

Cons:
- Pattern semantics become dialect-specific (glob vs regex behavior)
- Harder to reason about than explicit `EXCEPT (...)`
- Greater risk of accidental column capture when schema evolves
- More compile-time validation and error-reporting complexity

Current stance: keep core `SET *` + `EXCEPT (...)` as baseline; revisit pattern matching only if real-world migrations show repeated pain.
