# ADR 010: Schema-Driven Data Model — Catalog Introspection

**Status:** Accepted  
**Date:** 2026-02-22

## Context

pg_weave should work with any existing PG table — regular typed columns, JSONB columns, PG array columns, composite types, or any mix. The compiler needs to know each table's shape to generate correct SQL. However, requiring authors to manually declare schemas for tables that already exist in PG is redundant.

## Decision

The PG extension auto-introspects `pg_catalog` at compile time. No declarations needed for column names, types, arrays, or composite types. JSONB columns are treated as flexible — any nested path is valid and type errors surface at PostgreSQL execution time.

Output typing uses `INTO` on `COLLECT`/`MAP` (ADR 016). `INTO` names a PG composite type — the compiler resolves it from `pg_catalog`. Without `INTO`, nested outputs default to JSONB.

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
| Inside JSONB column | JSONB array | `jsonb_array_elements()` | `(elem.value->>'field')` |
| Top-level column | Scalar array (`text[]`) | `unnest()` | scalar |

The compiler auto-detects array type from introspected or declared schema.

## Rationale

- **Zero friction** — table columns introspected from catalog; no declarations needed
- **Works with existing tables** — pg_weave reads any PG table, no special conventions required
- **Hybrid tables** — mix typed columns + JSONB + PG arrays + composite types freely
- **Automatic maintenance** — DDL event trigger keeps VIEWs in sync with source table and custom type changes

## Consequences

- PG extension depends on catalog access at compile time — source table must exist before weave is compiled
- DDL event trigger adds a dependency on PG's event trigger infrastructure
- JSONB columns cannot validate nested field names or infer types — errors surface at query time
- Compiler needs a `SchemaProvider` trait
