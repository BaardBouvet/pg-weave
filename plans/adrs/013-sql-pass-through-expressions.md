# ADR 013: SQL Pass-Through Expressions

**Status:** Accepted  
**Date:** 2026-02-22

## Context

The DSL needs an expression language for SET and LET values. Options range from inventing a custom expression syntax, to curating an allowlist of functions, to passing expressions through to PostgreSQL unchanged.

## Decision

Expressions inside `SET` and `LET` are **pass-through SQL**. Any valid PG scalar expression is accepted and emitted directly into the generated VIEW. The DSL adds structure (`FROM`, `COLLECT`, `MAP`, `FLATMAP`, `WHEN`); the expression part is just SQL.

```
-- All of these are valid SET expressions:
SET name       = c.first_name || ' ' || c.last_name,
SET total      = SUM(MAP o.items AS item -> item.price * item.qty),
SET tier       = CASE WHEN total > 1000 THEN 'premium' ELSE 'standard' END,
SET created    = NOW(),
SET normalized = LOWER(TRIM(c.email)),
SET fallback   = COALESCE(c.nickname, c.first_name),
SET rounded    = ROUND(o.amount, 2),
SET typed      = o.data->>'quantity'::integer
```

The parser treats the expression as an opaque token sequence between `=` and the next `,`, `WHEN` suffix, or `}`. DSL-specific constructs (`MAP`, `COLLECT`, `LOOKUP`, `FLATMAP`) are recognized within expressions; everything else passes through.

### What the DSL adds on top of SQL

| Construct | DSL or SQL? | Purpose |
|-----------|-------------|---------|
| `SET field = expr` | DSL | Output field assignment |
| `LET name = expr` | DSL | Scoped temporary variable |
| `WHEN condition` (suffix) | DSL | Conditional field inclusion |
| `MAP array AS alias { ... }` | DSL | Array element transform |
| `MAP array AS alias -> expr` | DSL | Inline array transform |
| `COLLECT dataset AS alias ON ... { ... }` | DSL | Nested array aggregation |
| `LOOKUP dataset ON ...` | DSL | 1:1 join |
| `FLATMAP array AS alias { ... }` | DSL | Row explosion |
| `CASE WHEN ... END` | SQL | Conditional value — just passes through |
| `COALESCE(...)` | SQL | Null handling — just passes through |
| `SUM(...)`, `COUNT(...)`, etc. | SQL | Aggregates — just pass through |
| `||`, `*`, `+`, `-`, etc. | SQL | Operators — just pass through |
| `::type` | SQL | Type cast — just passes through |
| Any PG function | SQL | `UPPER()`, `ROUND()`, `NOW()`, etc. |

### Controlling output column types

Because `::type` passes through to SQL, you control the type of any output column directly in the `SET` expression — no separate output-declaration syntax is needed:

```
-- Ensure numeric output even when source is text/JSONB
SET price    = (item->>'price')::numeric,
SET quantity = (item->>'qty')::integer,
SET label    = id::text,
SET created  = (data->>'ts')::timestamptz
```

This replaces any need for a dedicated `RETURNS { ... }` block: cast at the assignment site and PG enforces the type when the VIEW is created.

### Sorting existing arrays (SQL pass-through)

MAP and COLLECT support `ORDER BY` for sorting their output. To sort an existing array *without* transforming it, use SQL pass-through with a subquery:

```
-- Sort a scalar PG array
SET sorted_tags = (SELECT array_agg(t ORDER BY t) FROM unnest(p.tags) AS t)

-- Sort a JSONB array by a nested field
SET sorted_items = (SELECT jsonb_agg(elem ORDER BY (elem->>'price')::numeric DESC)
                    FROM jsonb_array_elements(o.data->'items') AS elem)

-- Reverse a JSONB array
SET reversed = (SELECT jsonb_agg(elem ORDER BY ordinality DESC)
                FROM jsonb_array_elements(o.data->'items') WITH ORDINALITY AS elem)
```

Alternatively, use MAP with ORDER BY — this is the DSL-native approach:

```
-- Pre-sort: order input elements by a source field, then transform
SET sorted_items = MAP o.data.items AS item ORDER BY item.price DESC {
  SET name  = item.name,
  SET price = item.price
}

-- Post-sort: transform first, then order by a computed field
SET sorted_items = MAP o.data.items AS item {
  SET name   = item.name,
  SET margin = item.price - item.cost
} ORDER BY margin DESC

-- Post-sort with LET: sort by a value that doesn't appear in the output
SET items = MAP o.data.items AS item {
  LET discount = LOOKUP discounts ON sku = item.sku,
  SET name  = item.name,
  SET price = item.price
} ORDER BY discount.rate DESC

-- Sort without changing shape: identity transform + pre-sort
SET sorted_items = MAP o.data.items AS item ORDER BY item.price DESC -> item
```

`ORDER BY` position determines when sorting happens:
- **Before the block** (`ORDER BY ... { }`) — sorts input elements before transformation. Can only reference source fields.
- **After the block** (`{ } ORDER BY ...`) — sorts output elements after transformation. Can reference computed SET fields.

The inline arrow form (`-> item`) passes each element through unchanged, making it a pure sort.

This is standard SQL — subquery variants pass through to the generated VIEW unchanged. MAP with ORDER BY is preferred when you want sorting with DSL-level readability.

### Non-deterministic functions

Functions like `NOW()`, `RANDOM()`, `uuid_generate_v4()` make VIEWs volatile and break IVM. This is the user's responsibility — pg_weave does not detect or block volatile expressions. The IVM extension itself will reject incompatible VIEWs at materialization time.

### Identifier quoting

SQL double-quote syntax works for identifiers that are reserved words, contain special characters, or have spaces — both column names and table/dataset names:

```
SET "order" = o.id,              -- reserved word as column name
SET total = o."order-total",     -- hyphen in source column

FROM "my-orders" AS o {          -- hyphen in table name
  LET c = LOOKUP "customer data" ON id = o.customer_id,
  ...
}
```

Quoted identifiers pass through to generated SQL unchanged.

## Rationale

- **Zero learning curve** — SQL users already know the expression syntax
- **Full PG power** — no artificial restrictions; any function, operator, or cast works
- **Simpler parser** — expressions are opaque token sequences, not a custom AST
- **Composable** — DSL constructs (`MAP`, `COLLECT`) can appear inside SQL expressions naturally
- **No maintenance burden** — new PG functions work automatically, no allowlist updates

## Consequences

- Expression errors surface at VIEW creation time (PG reports them), not at DSL parse time
- No compile-time type checking of SQL expressions — that's PG's job
- Non-deterministic expressions are the user's problem — documented, not enforced
- Parser must handle nested parentheses and SQL keywords inside expressions without confusing them with DSL keywords
