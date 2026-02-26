# ADR 019: BUILD Constructor for Nested Non-Array Output

**Status:** Accepted  
**Date:** 2026-02-26

## Context

pg_weave handles flat fields (`SET`) and nested arrays (`COLLECT`/`MAP`) well.
Building nested single objects currently requires SQL pass-through (`jsonb_build_object`, `jsonb_set`, merges), which is verbose and inconsistent with block-style DSL composition.

## Decision

Introduce an expression-form build constructor:

```sql
SET profile = BUILD {
  SET name = c.name,
  SET contact = BUILD {
    SET email = c.email,
    SET phone = c.phone WHEN c.phone IS NOT NULL
  }
}
```

`BUILD { ... }` returns a single object value and can be used in `SET`/`LET` expressions.

Typed object output is supported with `INTO`, aligned with `MAP`/`COLLECT`:

```sql
SET profile = BUILD INTO profile_t {
  SET name  = c.name,
  SET email = c.email
}
```

- Without `INTO`, `BUILD` returns `jsonb`.
- With `INTO some_type`, `BUILD` lowers to a typed row/composite value (`some_type`).

## Rationale

- Reuses familiar block semantics (`SET`, `WHEN`, uniqueness rules)
- Reduces SQL verbosity for common nested-structure shaping
- Keeps SQL pass-through available for advanced object operations
- Complements `COLLECT`/`MAP` by covering non-array nesting

### Naming rationale (`BUILD`)

- **Functional verb style** — fits the existing action-oriented keyword set (`LOOKUP`, `COLLECT`, `MAP`, `FLATMAP`)
- **PostgreSQL alignment** — maps naturally to `jsonb_build_object(...)` / `json_build_object(...)` in lowering
- **Type-neutral with `INTO`** — reads well in both forms: `BUILD { ... }` (jsonb default) and `BUILD INTO some_type { ... }` (typed composite)

## Consequences

- Grammar must allow `object_expr` in `value_expr`
- Lowering must define key omission vs explicit `null` behavior
- Key collisions follow existing explicit `SET` precedence rules
- Grammar must support optional `INTO type_ref` on `BUILD`
- Typed `BUILD` values must validate field compatibility against the target composite type

## Future extension: dotted `SET` targets

Potential sugar:

```sql
SET profile.name = c.name,
SET profile.contact.email = c.email
```

Interpretation: unquoted dots build nested objects by path (equivalent to nested `BUILD` construction).

To target a literal key containing a dot (no nesting), quote the key:

```sql
SET "profile.name" = c.name
```

This keeps path semantics explicit while preserving access to literal dotted key names.
