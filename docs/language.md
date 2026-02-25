# Language Reference

pg_weave uses a SQL-inspired DSL for declarative transforms.

## Core blocks

- `FROM source AS alias { ... }` — entry point for a weave
- `SET field = expr` — emit output fields
- `LET name = expr` — local computed values (reusable in `SET` / `WHERE`)
- `WHERE expr` — inline or block-trailing filtering

### `FROM`

Starts a weave from a source relation and binds alias `o`.

```sql
FROM orders AS o {
	-- output one field from the source row
	SET id = o.id
}
```

### `SET`

Emits output fields using literals and SQL expressions.

```sql
FROM orders AS o {
	-- copy a source field
	SET id          = o.id,
	-- compute a new display field
	SET total_label = 'USD ' || o.amount::text
}
```

### `SET ... WHEN ...`

Conditionally emits a field only when the predicate is true.

```sql
FROM orders AS o {
	SET id = o.id,
	SET discount_code = o.discount_code WHEN o.discount_code IS NOT NULL
}
```

### `LET`

Computes a reusable intermediate value before emitting fields.

Ordering rule in a block:

- `SET` and `LET` can be interleaved.
- A `LET` binding is in scope from its declaration onward in that block.
- Referencing a `LET` name before its declaration is invalid.
- `SET` names are output fields, not reusable expression bindings; use `LET` for references.

```sql
FROM orders AS o {
	-- reusable local value (not emitted by itself)
	LET total = o.subtotal + o.tax,
	SET id    = o.id,
	-- emit computed value
	SET total = total
}
```

### `WHERE` (inline and trailing)

Uses inline filtering on input rows and trailing filtering on computed values.

```sql
FROM orders AS o WHERE o.status = 'active' {
	-- compute first, then filter by computed value
	LET total = o.subtotal + o.tax,
	SET id    = o.id,
	SET total = total,
	WHERE total > 100
}
```

### `WITH` (optional JSONB shape)

Declares JSONB field shape for compile-time validation/casting hints.

```sql
FROM orders AS o
	WITH data { items [{ price numeric, qty integer }] }
{
	-- `item.price` and `item.qty` are validated by WITH shape
	SET total = SUM(MAP o.data.items AS item -> item.price * item.qty)
}
```

Without `WITH`, JSONB paths are flexible and many mistakes are only caught later by PostgreSQL at execution time.

For full schema/type behavior (input validation, output typing, JSONB vs typed nested outputs),
see [Schema and Typing Guide](schema.md).

## Data operations

- `LOOKUP dataset ON ...` — fetch one related entity (1:1)
- `COLLECT dataset AS alias ON ... { ... }` — nested 1:N collection
- `MAP array AS alias { ... }` — element-wise array transformation
- `FLATMAP array AS alias { ... }` — explode array elements into output rows

### `LOOKUP`

Fetches one related entity (1:1) and reuses it via a local alias.

```sql
FROM orders AS o {
	-- resolve customer row for this order
	LET c = LOOKUP customers ON id = o.customer_id,
	SET order_id   = o.id,
	SET customer_name = c.name
}
```

### Multi-hop `LOOKUP` (fixed-depth joins)

Chain multiple `LOOKUP` bindings when you know hop count ahead of time.

```sql
FROM orders AS o {
	LET customer = LOOKUP customers ON id = o.customer_id,
	LET country  = LOOKUP countries ON id = customer.country_id,
	LET region  = LOOKUP regions ON id = country.region_id,

	SET order_id      = o.id,
	SET customer_name = customer.name,
	SET country_name  = country.name,
	SET region_name   = region.name
}
```

### `COLLECT`

Builds a nested array from related rows (1:N).

```sql
FROM customers AS c {
	SET id = c.id,
	-- collect all orders for each customer
	SET orders = COLLECT orders AS o ON o.customer_id = c.id {
		SET id     = o.id,
		SET amount = o.amount
	-- order the nested array
	} ORDER BY amount DESC
}
```

Add `INTO type[]` to target a PG composite type instead of the JSONB default:

```sql
SET orders = COLLECT orders AS o ON o.customer_id = c.id
	INTO order_summary[] {
	SET id     = o.id,
	SET amount = o.amount
}
```

Full typing rules are documented in [Schema and Typing Guide](schema.md).

### `MAP`

Transforms each element of an array into a new value.

```sql
FROM orders AS o {
	-- compute per-item totals from JSON array elements
	SET item_totals = MAP o.data.items AS item -> item.price * item.qty
}
```

### `FLATMAP`

Explodes an array into multiple output rows.

```sql
FROM orders AS o {
	-- one output row per item
	FLATMAP o.data.items AS item {
		SET order_id = o.id,
		SET sku      = item.sku,
		SET qty      = item.qty
	}
}
```

### Multiple `FLATMAP` blocks

Use multiple `FLATMAP` blocks when you want to emit rows from different arrays in one weave.

```sql
FROM customers AS c {
	FLATMAP c.emails AS email {
		SET customer_id = c.id,
		SET type  = 'email',
		SET value = email
	},
	FLATMAP c.phones AS phone {
		SET customer_id = c.id,
		SET type  = 'phone',
		SET value = phone
	}
}
```

## Expression model

Expressions in `SET`, `LET`, and `WHERE` are pass-through PostgreSQL SQL expressions.

### `COUNT OF array_expr`

Use `COUNT OF ...` when you want array length without caring whether the value is JSONB or a typed PostgreSQL array.

```sql
FROM person AS p {
	SET orders = COLLECT orders AS o ON o.cust_id = p.id {
		SET id = o.id
	},
	LET order_count = COUNT OF orders,
	SET order_count = order_count
}
```

Lowering behavior:

- JSONB array value → `jsonb_array_length(...)`
- Typed PG array value → `cardinality(...)`

## Formal grammar

See [Grammar Specification (EBNF)](grammar.md).
