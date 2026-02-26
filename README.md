# pg_weave

## Vision

pg_weave is a small, SQL-inspired language for reshaping PostgreSQL data -- turning flat rows, JSONB documents, and related tables into nested, structured outputs.

No modeling step. No schema declarations. No new stack to learn. If you know SQL, you can start immediately.

Write transforms that read like intent, then compile them into standard PostgreSQL `VIEW`s that slot into your existing tooling.

- Point at tables and reshape directly -- no upfront setup
- SQL expressions and functions pass through unchanged
- Nested outputs default to JSONB; use `INTO` only when typed composites are required
- PostgreSQL extension entrypoint: `pg_weave(...)` compiles transforms into standard `VIEW`s

## Annotated Example

Given source rows from `person` and `orders`:

```json
{
  "id": "1",
  "name": "John Smith",
  "age": 25
}
```

```json
[
  { "id": "100", "cust_id": "1", "amount": 320 },
  { "id": "200", "cust_id": "1", "amount": 500 }
]
```

Define a weave:

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

This produces rows shaped like:

```json
{
  "id": "1",
  "type": "customer",
  "name": "JOHN SMITH",
  "orders": [
    { "id": "100", "amount": 320 },
    { "id": "200", "amount": 500 }
  ],
  "order_count": 2
}
```

## Documentation

- [Language reference](docs/language.md)
- [Grammar specification (EBNF)](docs/grammar.md)
- [Schema and typing guide](docs/schema.md)
