# Schema and Typing Guide

pg_weave separates input validation from nested output typing.

## Optional typing principle

Output typing is opt-in. You can ship weaves without defining output types first, and add types later when contracts need to be stricter.

## Input shape (`WITH`)

`WITH` is optional and only describes JSONB input structure on `FROM`.

```sql
FROM orders AS o
  WITH data { items [{ price numeric, qty integer }], notes text }
{
  SET total = SUM(MAP o.data.items AS item -> item.price * item.qty)
}
```

- Use `WITH` for compile-time JSON path/type validation.
- `WITH` does not define output field types.

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

- Start without `WITH` and without `INTO`.
- Add `WITH` when JSONB input safety matters.
- Add `INTO` only when downstream consumers need typed nested arrays.
