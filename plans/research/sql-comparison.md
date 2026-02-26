# Why pg_weave: Side-by-Side Comparison

> **Purpose:** Show the readability gap between pg_weave and hand-written SQL as query complexity grows.

## Simple example

Two tables: `person`, `orders`. Enrich each person with their sorted orders and a derived order count.

### pg_weave

```sql
SELECT pg_weave('person_with_orders', $$
-- Persons enriched with sorted orders and derived order count
FROM person AS p {
  SET id = p.id,
  -- regular SQL literals/functions work directly
  SET type = 'customer',
  SET name = upper(p.name),

  SET orders = COLLECT orders AS o ON o.cust_id = p.id {
    SET id     = o.id,
    SET amount = o.amount
  } ORDER BY amount ASC,

  LET order_count = COUNT OF orders,
  SET order_count = order_count,

  WHERE order_count > 1
}
$$);
```

### Equivalent SQL

```sql
CREATE VIEW person_with_orders AS
SELECT
  p.id,
  'customer' AS type,
  upper(p.name) AS name,
  o_agg.orders,
  o_agg.order_count
FROM person p
LEFT JOIN LATERAL (
  SELECT
    jsonb_agg(
      jsonb_build_object('id', o.id, 'amount', o.amount)
      ORDER BY o.amount ASC
    ) AS orders,
    count(*) AS order_count
  FROM orders o
  WHERE o.cust_id = p.id
) o_agg ON true
WHERE o_agg.order_count > 1;
```

### BigQuery Pipe Syntax

```sql
FROM person AS p
|> EXTEND
    ARRAY(
      SELECT AS STRUCT o.id, o.amount
      FROM orders AS o
      WHERE o.cust_id = p.id
      ORDER BY o.amount ASC
    ) AS orders
|> EXTEND ARRAY_LENGTH(orders) AS order_count
|> WHERE order_count > 1
|> SELECT
    p.id,
    'customer' AS type,
    UPPER(p.name) AS name,
    orders,
    order_count;
```

### Malloy

```malloy
-- Source definition (required upfront)
source: person_source is duckdb.table('person') extend {
  join_many: orders is duckdb.table('orders')
    on orders.cust_id = id
  measure: order_count is count(orders.id)
}

-- Query
run: person_source -> {
  group_by:
    id
    type is 'customer'
    name is upper(name)
  aggregate: order_count
  nest: orders is {
    select: orders.id, orders.amount
    order_by: amount asc
  }
} -> {
  where: order_count > 1
}
```

At this level all four are readable. Pipe syntax has a nice top-down flow. BigQuery's `ARRAY(SELECT AS STRUCT ...)` is cleaner than PostgreSQL's `jsonb_agg(jsonb_build_object(...))`. Malloy's query is the most concise, but requires the source definition upfront. The difference becomes clear when complexity grows.

---

## Complex example

Four tables: `customers`, `countries`, `orders`, `line_items`.

Output: each customer with their country name (LOOKUP), nested orders each containing nested line items (two-level COLLECT), an order count, and a conditional VIP flag.

## pg_weave

```sql
SELECT pg_weave('customer_overview', $$
FROM customers AS c {
  LET co = LOOKUP countries ON id = c.country_id,

  SET id           = c.id,
  SET name         = c.first_name || ' ' || c.last_name,
  SET country      = co.name,

  SET orders = COLLECT orders AS o ON o.customer_id = c.id {
    SET id     = o.id,
    SET date   = o.created_at,
    SET total  = o.subtotal + o.tax,

    SET items = COLLECT line_items AS li ON li.order_id = o.id {
      SET sku   = li.sku,
      SET qty   = li.qty,
      SET price = li.unit_price * li.qty
    } ORDER BY sku ASC
  } ORDER BY date DESC,

  LET order_count = COUNT OF orders,
  SET order_count = order_count,
  SET vip         = true WHEN order_count >= 10
}
$$);
```

**~25 lines.** Reads top-down: source, lookups, flat fields, nested collections, derived values.

## Equivalent SQL

```sql
CREATE VIEW customer_overview AS
SELECT
  c.id,
  c.first_name || ' ' || c.last_name AS name,
  co.name AS country,
  o_agg.orders,
  o_agg.order_count,
  CASE WHEN o_agg.order_count >= 10 THEN true ELSE NULL END AS vip
FROM customers c
LEFT JOIN countries co ON co.id = c.country_id
LEFT JOIN LATERAL (
  SELECT
    jsonb_agg(
      jsonb_build_object(
        'id',    o.id,
        'date',  o.created_at,
        'total', o.subtotal + o.tax,
        'items', li_agg.items
      )
      ORDER BY o.created_at DESC
    ) AS orders,
    count(*) AS order_count
  FROM orders o
  LEFT JOIN LATERAL (
    SELECT
      jsonb_agg(
        jsonb_build_object(
          'sku',   li.sku,
          'qty',   li.qty,
          'price', li.unit_price * li.qty
        )
        ORDER BY li.sku ASC
      ) AS items
    FROM line_items li
    WHERE li.order_id = o.id
  ) li_agg ON true
  WHERE o.customer_id = c.id
) o_agg ON true;
```

**~40 lines.** The structure is buried in nested LATERAL joins, repeated `jsonb_build_object` calls, and manual aggregate threading. Adding a third nesting level or another LOOKUP makes this significantly worse.

## BigQuery Pipe Syntax

```sql
FROM customers AS c
|> LEFT JOIN countries AS co ON co.id = c.country_id
|> EXTEND
    ARRAY(
      SELECT AS STRUCT
        o.id,
        o.created_at AS date,
        o.subtotal + o.tax AS total,
        ARRAY(
          SELECT AS STRUCT li.sku, li.qty, li.unit_price * li.qty AS price
          FROM line_items AS li
          WHERE li.order_id = o.id
          ORDER BY li.sku ASC
        ) AS items
      FROM orders AS o
      WHERE o.customer_id = c.id
      ORDER BY o.created_at DESC
    ) AS orders
|> EXTEND ARRAY_LENGTH(orders) AS order_count
|> EXTEND IF(order_count >= 10, TRUE, NULL) AS vip
|> SELECT
    c.id,
    c.first_name || ' ' || c.last_name AS name,
    co.name AS country,
    orders,
    order_count,
    vip;
```

**~30 lines.** The top-level flow reads top-down thanks to the pipe operators. However, the nested `ARRAY(SELECT AS STRUCT ...)` subqueries inside `EXTEND` are still standard SQL -- pipe operators don't reach inside them. Adding a third nesting level adds another nested `ARRAY()` subquery, just like standard SQL. The real readability win of pipe syntax is for flat transformations, not nesting.

## Malloy

```malloy
-- Source definition (required upfront)
source: customer_source is duckdb.table('customers') extend {
  join_one: countries is duckdb.table('countries')
    on countries.id = country_id
  join_many: orders is duckdb.table('orders')
    on orders.customer_id = id extend {
      join_many: line_items is duckdb.table('line_items')
        on line_items.order_id = id
    }
  dimension: full_name is concat(first_name, ' ', last_name)
  measure: order_count is count(orders.id)
}

-- Query
run: customer_source -> {
  group_by:
    id
    name is full_name
    country is countries.name
  aggregate: order_count
  nest: orders is {
    group_by:
      orders.id
      date is orders.created_at
      total is orders.subtotal + orders.tax
    nest: items is {
      select:
        orders.line_items.sku
        orders.line_items.qty
        price is orders.line_items.unit_price * orders.line_items.qty
      order_by: sku asc
    }
    order_by: date desc
  }
} -> {
  where: order_count > 1
  calculate: vip is pick true when order_count >= 10
}
```

**~35 lines** (12 source + 23 query). The query itself is the cleanest of all four -- `nest:` blocks mirror output shape just like pg_weave's `COLLECT`, and path traversal (`orders.line_items.sku`) removes the need for ON clauses inside the query. The cost is the mandatory source model: every join, dimension, and measure must be declared before you query. For a single query this is overhead; across many queries over the same tables it pays back.

## Observations

| | pg_weave | SQL | BigQuery Pipe | Malloy |
|---|---|---|---|---|
| Line count | ~25 | ~40 | ~30 | ~35 (12 source + 23 query) |
| Nesting visible in structure | Yes -- block indentation mirrors output shape | No -- LATERAL nesting is inverted (deepest subquery first) | Partial -- top-level is piped, nesting is standard SQL inside `ARRAY()` | Yes -- `nest:` blocks mirror output shape |
| Adding a field | One `SET` line | Add to `jsonb_build_object` args | Add to `SELECT AS STRUCT` inside `ARRAY()` | One line in `group_by:`/`select:` |
| Adding another nesting level | Add a `COLLECT` block | Another `LEFT JOIN LATERAL` + `jsonb_agg` + `jsonb_build_object` | Another nested `ARRAY(SELECT AS STRUCT ...)` | Another `nest:` block |
| LOOKUP (1:1 join) | `LET co = LOOKUP ... ON` | `LEFT JOIN ... ON` | `\|> LEFT JOIN ... ON` | Pre-declared `join_one:` in source |
| Conditional field | `SET vip = true WHEN ...` | `CASE WHEN ... END` | `IF(expr, TRUE, NULL)` | `pick true when ...` |
| Ordering nested arrays | `ORDER BY` at block end | `ORDER BY` inside `jsonb_agg()` | `ORDER BY` inside `ARRAY()` | `order_by:` at block end |
| Upfront model required | No | No | No | Yes -- `source:` definition |
| Output | PostgreSQL VIEW | PostgreSQL VIEW | Query result (BigQuery) | Query result (multi-DB) |

The gap grows with every additional nesting level, LOOKUP, or conditional field.

**Pipe syntax verdict:** genuinely cleaner than standard SQL for flat pipelines (filter, extend, aggregate, join). For *nested output structures*, the pipe operators don't help -- you still write standard SQL subqueries inside `ARRAY()`. pg_weave's advantage over pipe syntax is specifically in the nesting.

**Malloy verdict:** the strongest alternative for nested output. `nest:` blocks are structurally equivalent to pg_weave's `COLLECT`, and path traversal through pre-declared joins eliminates ON clauses inside queries. The trade-offs are: (1) a mandatory source model must be written and maintained before any query, (2) Malloy is a separate language and compiler -- not embedded in SQL, (3) output is query results, not materialized VIEWs that other SQL can reference. pg_weave targets a different niche: self-contained transforms that compile to PostgreSQL VIEWs with no modelling step and no toolchain beyond a PG extension.
