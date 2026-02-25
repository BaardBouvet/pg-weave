# ADR 012: SQL-Style Comments

**Status:** Accepted  
**Date:** 2026-02-22

## Context

As the DSL drops the `WEAVE name` keyword (name is supplied externally via `pg_weave()`), definitions start directly with `FROM`. A comment above the `FROM` line is the natural place to describe what a weave does. The DSL needs a comment syntax.

## Decision

Adopt SQL's `--` single-line comment syntax. Everything from `--` to end of line is ignored by the parser.

```
-- Enrich orders with customer name and line item totals
FROM orders AS o {
  LET c = LOOKUP customers ON id = o.customer_id,

  SET customer_name = c.name,       -- full name from customer record
  SET order_total   = o.amount
}
```

- `--` only (no `/* */` block comments) — keeps the parser simple
- Comments are allowed anywhere a newline is valid
- Inline comments after expressions are supported

## Rationale

- **SQL familiarity** — `--` is universally recognized by SQL users
- **Self-documenting weaves** — without a name keyword in the body, a leading comment is the primary way to describe intent
- **Zero ambiguity** — `--` has no overlap with any DSL operator or expression syntax
- **Parser simplicity** — single-line only; no nesting or balancing concerns

## Consequences

- Lexer strips `--` to EOL before parsing — trivial to implement
- No multi-line block comments — long explanations use multiple `--` lines
- Comments are not preserved in the AST (stripped at lex time)
