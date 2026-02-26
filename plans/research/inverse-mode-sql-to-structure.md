# Inverse Mode: SQL to Structure

> **Date:** 2026-02-27
> **Status:** Exploratory idea

## Context

AI coding agents (Copilot, Cursor, etc.) increasingly write SQL for users. The bottleneck is shifting from *writing* SQL to *understanding* SQL that was written for you. A 40-line query with nested LATERAL joins, jsonb_agg, and CASE expressions is hard to read even if you didn't have to write it.

pg_weave currently goes DSL -> SQL. What if we also (or instead) went SQL -> structure?

## The idea

Parse existing PostgreSQL SQL and derive the same structural representation that pg_weave uses internally -- then render it as a readable, indented tree that mirrors the output shape.

Input: any SELECT query (or VIEW definition) that produces nested JSON.

Output: a pg_weave-like structural summary showing:
- Source tables and their roles (root, lookup, collected)
- Output fields and their derivations
- Nesting structure (which arrays contain which objects)
- Join conditions and cardinalities (1:1 vs 1:N)
- Filters at each level
- Ordering of nested arrays

### Example

Given this SQL (from the comparison doc):

```sql
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
        'id', o.id, 'date', o.created_at,
        'total', o.subtotal + o.tax,
        'items', li_agg.items
      ) ORDER BY o.created_at DESC
    ) AS orders,
    count(*) AS order_count
  FROM orders o
  LEFT JOIN LATERAL (
    SELECT jsonb_agg(
      jsonb_build_object('sku', li.sku, 'qty', li.qty,
        'price', li.unit_price * li.qty)
      ORDER BY li.sku ASC
    ) AS items
    FROM line_items li WHERE li.order_id = o.id
  ) li_agg ON true
  WHERE o.customer_id = c.id
) o_agg ON true
```

The tool could produce:

```
customers (root)
  id           = c.id
  name         = c.first_name || ' ' || c.last_name
  country      = co.name                              ← countries (1:1, ON co.id = c.country_id)
  orders[]                                            ← orders (1:N, ON o.customer_id = c.id) ORDER BY created_at DESC
    id         = o.id
    date       = o.created_at
    total      = o.subtotal + o.tax
    items[]                                           ← line_items (1:N, ON li.order_id = o.id) ORDER BY sku ASC
      sku      = li.sku
      qty      = li.qty
      price    = li.unit_price * li.qty
  order_count  = COUNT(orders)
  vip          = true WHEN order_count >= 10
```

## Why this might matter

1. **AI writes SQL, humans review it.** The structural view is a faster way to verify "does this query produce the shape I expect?" than reading nested LATERAL joins.

2. **Onboarding / documentation.** Point the tool at existing VIEWs in a database and auto-generate structural documentation of what each VIEW produces.

3. **Refactoring aid.** See the structure, spot redundancy (duplicate joins, repeated expressions), understand nesting depth before modifying.

4. **pg_weave adoption path.** Users can point it at their existing SQL, see the structural equivalent, and decide whether to adopt pg_weave for future development. Lower barrier than rewriting from scratch.

## What needs to be parsed

| SQL pattern | Structural meaning |
|---|---|
| `FROM table alias` | Root source |
| `LEFT JOIN table ON ...` (no LATERAL) | 1:1 LOOKUP |
| `LEFT JOIN LATERAL (SELECT jsonb_agg(...) ...) ON true` | 1:N COLLECT |
| `jsonb_build_object('key', expr, ...)` | Output fields inside a nested level |
| `jsonb_agg(... ORDER BY ...)` | Nested array with ordering |
| `count(*)` alongside `jsonb_agg` in same LATERAL | Derived count |
| `CASE WHEN ... THEN ... ELSE NULL END` | Conditional field |
| `WHERE` in outer query | Root-level filter |
| `WHERE` inside LATERAL subquery | Join condition for that nesting level |
| `WITH RECURSIVE` | Graph traversal (COLLECT FOLLOW) |
| `jsonb_array_length(...)` | COUNT OF |
| Nested LATERAL inside LATERAL | Multi-level nesting |

## Challenges

### Ambiguity in SQL patterns

Not all LEFT JOINs are LOOKUPs. Not all LATERAL subqueries are COLLECTs. The tool would need heuristics:
- LATERAL + `jsonb_agg` + `ON true` = very likely a COLLECT
- Non-LATERAL LEFT JOIN returning one row = likely a LOOKUP
- But edge cases exist (self-joins, semi-joins, etc.)

### Non-pg_weave SQL

Much SQL doesn't map to the pg_weave model at all -- GROUP BY aggregations, window functions, UNION queries, recursive CTEs for non-tree purposes. The tool should gracefully degrade: show what it can structure, leave the rest as raw SQL blocks.

### Expression complexity

`jsonb_build_object` args are straightforward. But users might use `row_to_json()`, `json_build_object()`, `array_agg(row(...))`, or custom functions. Each needs recognition.

### Parser availability

PostgreSQL's parser is available as a C library (`libpg_query`) with bindings in multiple languages (Rust, Python, Go, Node). The AST is well-documented. This is solvable.

## Scope options

### Minimal: read-only structural viewer

Parse SQL, produce indented structure text. No roundtripping, no code generation. Could be a CLI tool or VS Code extension.

**Effort:** Medium. Parse SQL AST, pattern-match known structures, render tree.

### Medium: bidirectional (view + edit)

Show the structure, let the user modify it (add a field, change ordering), regenerate the SQL. This is effectively full pg_weave -- just with SQL as the input format instead of the DSL.

**Effort:** High. Requires reliable roundtripping.

### Full: pg_weave becomes the visualization layer

Drop the DSL entirely. Users write SQL (or AI writes it), pg_weave parses it and provides structural visualization, documentation, and validation. The DSL was only ever a means to the structural view.

**Effort:** High, but changes the product positioning fundamentally.

## Relationship to current pg_weave

| | Current pg_weave | Inverse mode |
|---|---|---|
| Input | pg_weave DSL | SQL |
| Output | SQL (VIEWs) | Structural visualization |
| Who writes the query | Human writes DSL | AI writes SQL |
| Value proposition | Easier to write nested transforms | Easier to understand nested transforms |
| Parser needed | pg_weave grammar | PostgreSQL SQL grammar |
| Adoption barrier | Learn new DSL | Point at existing SQL |

These aren't mutually exclusive. Both could exist:
- **Write path:** pg_weave DSL -> SQL (for humans who prefer the DSL)
- **Read path:** SQL -> structural view (for understanding AI-generated or legacy SQL)

## Open questions

1. **Is the structural view enough, or do users want to edit through it?** Read-only is much simpler but less compelling.

2. **What percentage of real-world nested-JSON-producing SQL follows recognizable patterns?** If it's high, the tool is broadly useful. If it's low, it's a novelty.

3. **Does this cannibalize the DSL?** If AI + structural viewer is good enough, nobody needs to learn the DSL. That might be fine -- the goal is to help users, not to promote a language.

4. **Could LLMs do this already?** "Explain this SQL as a nested structure" is a plausible prompt. But LLMs hallucinate structure; a deterministic parser doesn't. The tool's value is precision and consistency.

5. **Integration surface:** CLI tool? VS Code extension? PG function (`SELECT pg_weave_explain('view_name')`)? All three?
