# Gap Analysis: Sesam Hub vs pg_weave

> **Date:** 2026-02-22  
> **Status:** Under discussion

## Critical Gaps

### 1. Namespaces / Property Collision Prevention

Sesam Hub's Namespaced Identifiers (`~:crm:email`, `~:erp:email`) prevent property name collisions when combining data from multiple systems. In pg_weave, alias-based disambiguation (`c.email` vs `e.email`) handles this naturally at the transform level.

Namespace collision is primarily a concern when *merging* identities from multiple sources — that's pgmerge's domain, not pg_weave's.

**Resolution:** Not applicable to pg_weave. Alias-based access provides natural disambiguation. Namespace collision prevention belongs in pgmerge.

### 2. Copy / Passthrough Semantics

Sesam Hub's `["copy", "*"]` copies all properties, then you modify a few. `SET *` copies all source columns at compile time, with optional `EXCEPT` and explicit `SET` overrides (order-independent).

**Resolution:** `SET *` with `EXCEPT`. See [ADR 015](adr/015-set-wildcard.md).

### 3. Soft-Delete Filtering (`_deleted`)

Sesam Hub explicitly filters `_deleted: true` entities from hops and joins. In pg_weave, this is handled with a `WHERE` clause — inline on `FROM` or `COLLECT`, or as a block-trailing filter.

```
FROM orders AS o WHERE NOT o.deleted {
  SET orders = COLLECT order_items AS oi ON oi.order_id = o.id WHERE NOT oi.deleted {
    SET product = oi.name
  }
}
```

**Resolution:** Defer to user — use `WHERE` clauses. No built-in convention needed.

### 4. Mid-Transform Filter / Discard

DTL's `["filter"]` conditionally drops entities mid-transform. pg_weave supports `WHERE` in two positions: inline (on source, before block) and block-trailing (at end of `{ }`, after LET computations). Both work at every level — top-level weave, COLLECT, MAP, FLATMAP.

**Resolution:** Inline + block-trailing WHERE. See [ADR 014](adr/014-where-clause.md).

### 5. Non-Deterministic Functions (`now`, `uuid`)

Using non-deterministic functions like `now()` makes a VIEW VOLATILE, breaking IVM. This is the user's responsibility — pg_weave passes expressions through to SQL without detection or blocking. The IVM extension itself will reject incompatible VIEWs.

**Resolution:** Defer to user. See [ADR 013](adr/013-sql-pass-through-expressions.md).

## Moderate Gaps

### 6. Union / Multi-Source Combinators

Sesam has `union_datasets` (UNION ALL with optional ID prefixing) and `merge_datasets` (FULL OUTER JOIN by `_id`). These are standard SQL operations — users can create regular SQL VIEWs for unions/joins and use those as the `FROM` source in a weave.

**Resolution:** Use regular SQL VIEWs. No DSL support needed.

### 7. Chained Transforms

Sesam DTL supports multi-stage transform passes (each stage's output feeds the next). In pg_weave, each weave compiles to a VIEW — chaining is natural: one weave's output VIEW becomes another weave's `FROM` source.

**Resolution:** Chain weaves via VIEW references. No special syntax needed.

### 8. Conditional Expressions (`if` / `case`)

`CASE WHEN ... THEN ... ELSE ... END` is a SQL expression and passes through to the generated VIEW unchanged. No special DSL handling needed.

**Resolution:** SQL pass-through. See [ADR 013](adr/013-sql-pass-through-expressions.md).

### 9. Explain / Validate Tooling

The PG extension provides equivalent functions:
- `pg_weave_explain(definition text)` — show generated SQL without executing
- `pg_weave_validate(definition text)` — structural + semantic validation

**Resolution:** Built into the PG extension API.

## Minor Gaps

| Feature | Notes |
|---------|-------|
| `default` (set if not present) | Expressible as `COALESCE(field, default)` |
| `rename` | Expressible as `SET new = old` |
| `key-values`, `keys`, `values` | PG has `jsonb_each()`, `jsonb_object_keys()` |
| Set operations on arrays | PG has array operators; need DSL exposure |
| `flatten` (recursive array flatten) | Niche but useful |
| `sorted` on arbitrary arrays | `ORDER BY` in COLLECT covers most cases |
| `on_duplicate_id` handling | DISTINCT ON for N:1 mappings |
| Embedded source (inline static data) | Useful for tests/lookups — `VALUES` equivalent |
| Glob patterns (`copy "order_*"`) | Not needed with explicit SET model |

## Not Applicable

These Sesam Hub features don't apply to a PG extension model:

| Feature | Why N/A |
|---------|---------|
| HTTP/REST/Kafka sources | Data must be in PG; use FDW or external ETL |
| Pump scheduling | IVM replaces scheduled execution |
| Multi-tenancy | PG schemas handle isolation |
| Circuit breakers / retry | VIEWs don't fail per-row |
| JWT/OAuth | PG has native auth |
| Entity batching | VIEWs operate set-at-a-time |
| Webhook endpoints | Use PG `LISTEN`/`NOTIFY` |

## Resolution Tracker

| # | Gap | Resolution | ADR |
|---|-----|------------|-----|
| 1 | Namespaces | N/A — pgmerge concern | |
| 2 | Copy semantics | `SET *` with `EXCEPT` | [015](adr/015-set-wildcard.md) |
| 3 | Soft-delete | Defer to user — use WHERE | |
| 4 | Mid-transform filter | Inline + trailing WHERE | [014](adr/014-where-clause.md) |
| 5 | Non-deterministic functions | Defer to user | [013](adr/013-sql-pass-through-expressions.md) |
| 6 | Union sources | Regular SQL VIEWs | |
| 7 | Chained transforms | Chain weaves via VIEWs | |
| 8 | Conditional expressions | SQL pass-through | [013](adr/013-sql-pass-through-expressions.md) |
| 9 | Explain/validate | PG extension functions | |
