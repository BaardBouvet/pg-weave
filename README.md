# pg_weave

## Vision

pg_weave is a fresh, SQL-inspired language for shaping complex PostgreSQL data into clear, reusable models.

Write transforms that read like intent, not plumbing — then compile them into standard PostgreSQL `VIEW`s that fit naturally into your existing stack.

pg_weave is not a replacement for SQL. It keeps SQL at the center and makes the repetitive nested-shaping parts dramatically easier.

- Works with real-world table shapes: typed columns, JSONB, arrays, and mixed schemas
- SQL pass-through by default for expressions and functions
- Optional output typing: start schemaless, add `INTO` typed nested outputs when needed
- Primary runtime: PostgreSQL extension (`pg_weave(...)`)
- Output: deterministic SQL suitable for normal PG tooling and materialization flows

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
