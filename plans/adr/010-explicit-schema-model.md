# ADR 010: Schema-Driven Data Model — Introspection + WITH Clause

**Status:** Accepted  
**Date:** 2026-02-22

## Context

pg_weave should work with any existing PG table — regular typed columns, JSONB columns, PG array columns, composite types, or any mix. The compiler needs to know each table's shape to generate correct SQL. However, requiring authors to manually declare schemas for tables that already exist in PG is redundant.

## Decision

How the compiler learns table structure depends on runtime context:

- **PG extension** — auto-introspects `pg_catalog` at compile time. No declarations needed for column names, types, arrays, or composite types. Authors optionally add a `WITH` clause on `FROM` to declare nested structure inside JSONB columns.
- **dbt plugin** (pg_weave-dbt) — no catalog access. Authors provide full `SCHEMA` declarations as a catalog substitute.

`WITH` in this ADR is strictly for **input JSONB shape description** on `FROM`. Nested output typing is handled separately in ADR 016.

Output typing uses `INTO` on `COLLECT`/`MAP` (ADR 016). `INTO` names a PG composite type — the compiler resolves it from `pg_catalog` (PG extension) or project declarations (dbt plugin). Without `INTO`, nested outputs default to JSONB.

Important: `INTO` is **not** a replacement for `WITH` input validation. `INTO` describes what a nested block emits; `WITH` describes and validates JSONB fields read from inputs (including fields used only in `LET`, `WHERE`, `ON`, or `ORDER BY` and not present in output).

### WITH clause — optional JSONB validation (PG extension)

By default, JSONB columns are bare — any path is valid, no compile-time checking. The optional `WITH` clause on `FROM` adds field validation and auto-casting:

```
-- No WITH: any JSONB path accepted, no validation
FROM orders AS o {
  SET total = SUM(MAP o.data.items AS item -> item.price * item.qty)
}

-- WITH: compiler validates field names and infers types
FROM orders AS o
  WITH data { items [{ price numeric, qty integer }], notes text }
{
  SET total = SUM(MAP o.data.items AS item -> item.price * item.qty)
}
```

Both produce identical SQL. `WITH` is entirely optional — omit it for rapid prototyping, add it when you want compile-time safety.

- Multiple JSONB columns: `WITH data { ... }, metadata { ... }`

### SCHEMA declaration — full catalog (dbt plugin)

```
SCHEMA orders {
  order_id    integer PRIMARY KEY,
  customer_id text,
  status      text,
  data        jsonb {
    items     [{ sku text, price numeric, qty integer }],
    notes     text
  },
  metadata    jsonb
}
```

### Output typing — INTO clause

Nested `COLLECT`/`MAP` outputs can target a PG composite type via `INTO`:

```
-- PG composite type (standard DDL, managed outside the weave)
CREATE TYPE order_summary AS (id text, amount numeric);

-- Weave references the type via INTO
FROM customers AS c {
  SET id = c.id,
  SET orders = COLLECT orders AS o ON o.customer_id = c.id
    INTO order_summary[] {
    SET id = o.id,
    SET amount = o.amount
  }
}
```

Without `INTO`, the nested output defaults to JSONB (ADR 016).

`SCHEMA` remains available in dbt plugin mode as a catalog substitute for **input** table shapes.

### DDL change detection

The PG extension registers an `EVENT TRIGGER` on `ddl_command_end` for schema-affecting DDL, including `ALTER TABLE` and `ALTER TYPE` (for custom/composite types).

When a relevant object changes:
1. Resolve impacted relations/types (changed table directly, or tables/columns using the changed type)
2. Look up which weaves reference those relations or `INTO` target types
3. Re-introspect column/type metadata from `pg_catalog` (names, scalar/composite kind, array/composite element details)
4. Re-generate and replace the VIEW SQL
5. Raise a WARNING if the new schema/type contract is incompatible with the weave definition

### Field access

First segment after alias is always a real column. Subsequent segments into JSONB columns become `->` / `->>`:
- `o.status` → `o.status` (regular column)
- `o.data.notes` → `o.data->>'notes'` (JSONB path)
- `o.metadata.anything` → `o.metadata->>'anything'` (bare JSONB, unvalidated)

### Array types

| Source | PG type | Iterate | Field access |
|--------|---------|---------|-------------|
| Top-level column | Composite type array | `unnest()` | `(elem).field` |
| Inside `jsonb / WITH` | JSONB array | `jsonb_array_elements()` | `(elem.value->>'field')` |
| Top-level column | Scalar array (`text[]`) | `unnest()` | scalar |

The compiler auto-detects array type from introspected or declared schema.

## Rationale

- **Zero friction in PG extension** — table columns introspected; author only declares JSONB detail when wanted
- **Works with existing tables** — pg_weave reads any PG table, no special conventions required
- **Progressive detail** — start with no `WITH` (bare JSONB), add structure later for validation
- **Hybrid tables** — mix typed columns + JSONB + PG arrays + composite types freely
- **Automatic maintenance** — DDL event trigger keeps VIEWs in sync with source table and custom type changes
- **dbt portability** — explicit `SCHEMA` makes definitions self-contained outside the database

## Consequences

- PG extension depends on catalog access at compile time — source table must exist before WEAVE is compiled
- DDL event trigger adds a dependency on PG's event trigger infrastructure
- dbt plugin mode requires more upfront work (full SCHEMA) but definitions are portable
- Bare JSONB columns cannot validate field names or infer types — errors surface at query time
- Compiler needs a `SchemaProvider` trait with two implementations (catalog vs declared)
