# ADR 009: FP-Aligned Keywords — LOOKUP, COLLECT, MAP, FLATMAP

**Status:** Accepted  
**Date:** 2026-02-22

## Context

The DSL previously used two keywords for data operations: `NEST` (aggregate
nested structures, lookups, array mapping, graph walks) and `EMIT` (explode
arrays into rows). Having `NEST` cover four distinct operations made it hard
to tell at a glance what a statement does.

## Decision

Split `NEST` and `EMIT` into four keywords aligned with functional programming
conventions:

| Keyword | Replaces | FP concept | What it does | SQL compilation |
|---------|----------|------------|-------------|-----------------|
| **LOOKUP** | `NEST FIRST ... ON` | find | Fetch one related entity (1:1) | `LEFT JOIN LATERAL ... LIMIT 1` |
| **COLLECT** | `NEST ... ON` | groupJoin | Gather related entities into nested array | `LEFT JOIN LATERAL` + `jsonb_agg` / `array_agg` (mode-dependent) |
| **MAP** | `NEST array AS ...` | map | Transform each array element | `jsonb_build_object` / row expression + aggregate (mode-dependent) |
| **FLATMAP** | `EMIT ... AS ...` | flatMap | Explode array into output rows (1→N) | `LATERAL jsonb_array_elements` |

Both MAP and COLLECT support `ORDER BY expr [ASC|DESC]` in two positions:
- **Before the block** — pre-sort: orders input elements by a source field
- **After the block** — post-sort: orders output by a computed field

Pre-sort compiles to ordering inside the aggregate (`jsonb_agg(... ORDER BY expr)` / `array_agg(... ORDER BY expr)`). Post-sort wraps in a subquery to sort by the computed value.

### Examples

```
-- LOOKUP: enrich orders with customer name
FROM orders AS o {
  LET c = LOOKUP customers ON _id = o.customer_id,

  SET customer_name = c.name,
  SET line_items = MAP o.items AS item {
      SET product = item.name,
      SET total   = item.price * item.quantity
  },
  SET order_total = SUM(MAP o.items AS item -> item.price)
}
```

```
-- COLLECT: nest orders onto customers (ORDER BY controls nested array order)
FROM customers AS c {
  SET name = c.name,
  SET orders = COLLECT orders AS o ON o.customer_id = c._id ORDER BY o.amount DESC {
      SET order_id = o._id,
      SET total    = o.amount
  }
}
```

```
-- FLATMAP: explode items into rows
FROM orders AS o {
  FLATMAP o.items AS item {
    SET _id     = o._id || '-' || item.sku,
    SET product = item.name
  }
}
```

Graph walks use COLLECT with FOLLOW:

```
SET chain = COLLECT customers AS ref FOLLOW ref.referred_by DEPTH 5 { ... }
SET tree  = COLLECT RECURSIVE employees AS r FOLLOW r.manager_id DEPTH 8 { ... }
```

Post-sort can reference LET bindings computed inside the block:

```
-- Sort MAP output by a LET value that doesn't appear in the output
SET items = MAP o.items AS item {
  LET discount = LOOKUP discounts ON sku = item.sku,
  SET name  = item.name,
  SET price = item.price
} ORDER BY discount.rate DESC
```

Pre-sort (`ORDER BY` before the block) can only reference source fields; post-sort (`ORDER BY` after the block) can reference any SET field, including those derived from LETs.

## Rationale

- **Self-documenting** — each keyword tells you exactly what happens: LOOKUP finds one, COLLECT gathers many, MAP transforms in place, FLATMAP explodes
- **FP familiarity** — developers who know map/flatMap/collect recognize the semantics instantly
- **No ambiguity** — previously `NEST` covered 4 operations; now each has its own name
- **4 keywords vs 1** — slightly more to learn, but each is simpler to understand than overloaded `NEST`
- **COLLECT for graphs** — graph walks produce a collected array of visited nodes, so COLLECT + FOLLOW reads naturally

## Consequences

- Four keywords to learn instead of one — mitigated by each being self-explanatory
- Parser needs four AST node types instead of one with variants
- Migration from earlier `NEST`/`EMIT` examples requires renaming
