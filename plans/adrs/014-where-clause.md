# ADR 014: WHERE Clause — Inline and Block-Trailing

**Status:** Accepted  
**Date:** 2026-02-22

## Context

The DSL needs a way to filter rows at various levels — input rows before transformation, collected items, mapped elements, and output rows after computation. Two positions are useful:

- **Inline WHERE** — on the source line, filtering rows before entering the block
- **Block-trailing WHERE** — at the end of a `{ }` block, filtering after `LET` computations

Both forms serve different readability goals: inline WHERE reads like a SQL `WHERE` clause; block-trailing WHERE makes it clear the filter uses computed values.

## Decision

Support `WHERE` in both positions at every block level: top-level weave, COLLECT, MAP, and FLATMAP.

### Inline WHERE — filter source rows

Appears after the source declaration, before the block opens:

```
-- Top-level: filter input rows
FROM orders AS o WHERE o.status = 'active' {
  SET order_id = o.id,
  SET total    = o.amount
}

-- COLLECT: filter which rows to collect
FROM customers AS c {
  SET name = c.name,
  SET active_orders = COLLECT orders AS o ON o.customer_id = c.id WHERE o.status = 'active' {
    SET order_id = o.id,
    SET total    = o.amount
  }
}
```

### Block-trailing WHERE — filter after computation

Appears at the end of the `{ }` block, after all SET/LET statements. Can reference `LET` bindings.

```
-- Top-level: filter output rows using computed value
FROM orders AS o {
  LET total = SUM(MAP o.data.items AS item -> item.price * item.qty),

  SET order_id = o.id,
  SET total    = total,

  WHERE total > 100
}

-- COLLECT: filter collected items using computed value
FROM customers AS c {
  SET name = c.name,
  SET big_orders = COLLECT orders AS o ON o.customer_id = c.id {
    LET line_total = o.price * o.qty,

    SET order_id = o.id,
    SET total    = line_total,

    WHERE line_total > 100
  }
}

-- MAP: filter mapped elements
FROM orders AS o {
  SET expensive_items = MAP o.data.items AS item {
    SET name  = item.name,
    SET price = item.price,

    WHERE item.price > 50
  }
}
```

### Both forms combined

```
-- Inline filters input; trailing filters output after computation
FROM orders AS o WHERE o.status = 'active' {
  LET total = SUM(MAP o.data.items AS item -> item.price * item.qty),

  SET order_id = o.id,
  SET total    = total,

  WHERE total > 100
}
```

### Compilation

| Form | Compiles to |
|------|------------|
| Inline WHERE (top-level) | `WHERE` clause on the source table |
| Inline WHERE (COLLECT/MAP) | `WHERE` inside the `LATERAL` subquery |
| Block-trailing WHERE (top-level) | Wrapping subquery/CTE with outer `WHERE` |
| Block-trailing WHERE (COLLECT/MAP) | `WHERE` inside the `LATERAL` subquery (after computed columns) |

## Rationale

- **Inline WHERE** reads naturally for simple source filters — familiar SQL pattern
- **Block-trailing WHERE** enables filtering on computed `LET` values — not possible with inline
- **Both forms at every level** — consistent, no special rules about where WHERE is allowed
- **Pass-through SQL** — the condition expression is pass-through SQL like all other expressions

## Consequences

- Parser must handle WHERE in two positions per block — resolved by: inline WHERE appears between source and `{`, trailing WHERE appears after the last SET/LET inside `{ }`
- Both forms can be combined — inline narrows input, trailing narrows output
- Block-trailing WHERE at top level requires a wrapping subquery/CTE in generated SQL
