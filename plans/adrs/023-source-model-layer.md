# ADR 023: MODEL Layer — Rejected in Favour of Weave Chaining

**Status:** Rejected  
**Date:** 2026-02-26

## Context

Multiple weaves targeting the same tables repeat identical LOOKUPs and LET bindings. A dedicated MODEL concept (table + KEY + relationships + shared LETs) was explored to eliminate this repetition and enable ON-free COLLECT via relationship inference. See the [Malloy gap analysis](../research/malloy-gap-analysis.md) for background.

## Decision

Reject a separate MODEL concept. Use materialised weaves instead.

A "base" weave projects lookup results and computed values into a VIEW. Downstream weaves reference that VIEW as their FROM source — the joined/computed columns are already present as regular columns.

```sql
-- Base weave: materialises customer with resolved lookups
FROM customers AS c {
  SET *,
  LET co = LOOKUP countries ON id = c.country_id,
  LET r  = LOOKUP regions   ON id = co.region_id,
  SET country_name = co.name,
  SET region_name  = r.name,
  SET full_name    = c.first_name || ' ' || c.last_name
}
-- creates VIEW: customer_base

-- Downstream weave: references the materialised VIEW
FROM customer_base AS c {
  SET id      = c.id,
  SET name    = c.full_name,
  SET country = c.country_name,
  SET orders  = COLLECT order_base AS o ON o.customer_id = c.id {
    SET id    = o.id,
    SET total = o.total
  } ORDER BY total DESC
}
```

## Rationale

- One concept — weaves are the only abstraction. No MODEL, no KEY, no registry.
- Materialised base weaves are regular VIEWs — fully inspectable, standard PG tooling applies.
- IVM handles VIEW-on-VIEW chains; no special compiler support needed.
- ON stays explicit — no relationship inference, no implicit graph, no hidden magic.
- The repetition problem is solved by referencing a base VIEW instead of repeating LOOKUPs.

## Consequences

- Slightly more VIEWs in the schema (one per "base" layer).
- COLLECT ON remains explicit — users write the join condition each time.
- If relationship inference becomes valuable later, a MODEL layer can be revisited without breaking existing weaves.
- No module/import system yet — models and weaves live in the same compilation unit.
