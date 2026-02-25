# Schema and Typing Guide

Output typing in pg_weave is optional by design: start schemaless, then opt into stricter nested types only when needed.

## Nested output typing (`INTO`)

- No `INTO` on `COLLECT`/`MAP` → nested output defaults to `jsonb`.
- `INTO some_type[]` → nested output becomes a typed PG array.

```sql
CREATE TYPE order_summary AS (id text, amount numeric);

FROM customers AS c {
  SET id = c.id,
  SET orders = COLLECT orders AS o ON o.customer_id = c.id
    INTO order_summary[] {
    SET id = o.id,
    SET amount = o.amount
  }
}
```

For scalar arrays (for example `integer[]`), plain SQL is already simple and explicit:

```sql
FROM customers AS c {
  SET id = c.id,
  SET order_amounts = ARRAY(
    SELECT o.amount::integer
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.amount
  )
}
```

SQL generation summary:

- No `INTO` → JSON construction + `jsonb_agg(...)`
- `INTO some_type[]` → typed row construction + `array_agg(ROW(...)::some_type)`

## Practical default

- Start without `INTO`.
- Add `INTO` only when downstream consumers need typed nested arrays.
