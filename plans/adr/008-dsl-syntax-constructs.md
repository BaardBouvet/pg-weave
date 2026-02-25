# ADR 008: DSL Syntax Constructs — SET, LET, WHEN

**Status:** Accepted  
**Date:** 2026-02-22

## Context

The DSL needs a consistent, readable syntax for field assignment, temporary
variables, and conditional field inclusion. Early sketches used mixed styles
(colon inside nested blocks, bare `=` at top level). We want one unified style
across all block levels.

Expressions inside SET and LET are pass-through SQL — any valid PG scalar
expression works (see [ADR 013](013-sql-pass-through-expressions.md)).

## Decision

Three constructs are adopted:

Syntax style rule: avoid `@`-prefixed DSL annotations in core language constructs.

### SET — field assignment
Explicit keyword for every field assignment — in weave blocks, COLLECT blocks, and
inline object literals. One rule: **every field starts with `SET`**.

```
SET order_total = SUM(MAP o.items AS item -> item.price),
SET customer    = c.first_name || ' ' || c.last_name
```

Inside inline object literals, one SET per line:

```
SET line_items = MAP o.items AS item {
    SET product = item.name,
    SET total   = item.price * item.quantity
}
```

### LET — temporary variable
Declares a named value scoped to the enclosing `{ }` block. Not emitted to
output. Usable anywhere SET is usable — weave, COLLECT, MAP, FLATMAP, and inline object
literals.

```
LET total = SUM(MAP o.items AS item -> item.price * item.quantity),
LET name  = c.first_name || ' ' || c.last_name
```

Scoped to any block level:

```
-- LET scoped to any block level
FROM customers AS c {
  LET full_name = c.first_name || ' ' || c.last_name,

  SET name = full_name,
  SET orders = COLLECT orders AS o ON o.customer_id = c._id {
    LET line_total = o.price * o.quantity,

    SET order_id = o._id,
    SET total    = line_total,
    SET margin   = line_total * 0.15
  }
}
```

### WHEN — conditional field inclusion
Suffix on SET that controls whether the field is included in the output.
When the condition is false the field is omitted entirely (not set to null).
The condition after WHEN is a pass-through SQL boolean expression.

```
SET loyalty_bonus = total * 0.02 WHEN c.is_member = true
```

### Combined example

```
-- Combined example: lookup, computed fields, conditionals
FROM orders AS o {
  LET c = LOOKUP customers ON _id = o.customer_id,

  LET total = SUM(MAP o.items AS item -> item.price * item.quantity),
  LET customer_name = c.first_name || ' ' || c.last_name

  SET order_total   = total,
  SET customer      = customer_name,
  SET tier          = CASE WHEN total > 10000 THEN 'wholesale'
                           WHEN total > 1000  THEN 'premium'
                           ELSE 'standard'
                      END,                    -- SQL pass-through expression
  SET free_shipping = total > 500,
  SET loyalty_bonus = total * 0.02 WHEN c.is_member = true
}
```

## Rationale

- **SET everywhere** — explicit keyword avoids ambiguity with SQL `=` (comparison) and reads clearly at any depth, including inside inline object literals; no colons in the language
- **LET** — familiar from many languages; clearly distinct from SET (temp vs output); avoids confusion with SQL `WITH` (CTE)
- **WHEN suffix** — lightweight way to conditionally omit a field; omits rather than nulls, which matches JSON best practices
- **No `@` annotation style** — the DSL favors word-level constructs over symbol prefixes because it reads closer to SQL and natural language in real examples
- **No CASE WHEN construct** — `CASE WHEN ... END` is just a SQL expression, passed through like any other; no special DSL handling needed

## Consequences

- Parser must distinguish trailing `WHEN` (conditional inclusion) from `CASE WHEN` inside SQL expressions — resolved by position: `WHEN` after the complete value expression is always the suffix form; `CASE WHEN` appears inside the expression
- Avoiding `@`-style DSL markers makes SQL-expression boundary detection more parser-sensitive — expression parsing must rely on context/lookahead around DSL keywords and delimiters
- LET ordering matters — a LET can reference earlier LETs but not later ones
- LET is block-scoped — visible within its `{ }` block and nested blocks inside it, but not outside
- WHEN suffix means output shape can vary per row — consumers must handle optional fields
