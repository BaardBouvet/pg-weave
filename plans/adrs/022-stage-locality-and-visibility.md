# ADR 022: Stage Locality and Visibility Rules

**Status:** Accepted  
**Date:** 2026-02-26

## Context

Nested transforms need predictable scoping. We want reuse across levels, but without ancestor-path magic (`parent`, `root`) or deep alias reach-through that breaks during refactors.

## Decision

- Stage visibility is local: each block (`FROM`, `COLLECT`, `MAP`, `FLATMAP`) sees its immediate input alias/columns plus names defined in that stage.
- Lexical `LET` inheritance is allowed: nested blocks can read outer `LET` bindings; inner `LET` may shadow outer `LET`.
- Outer table aliases/columns are **not** directly visible in nested blocks; pass values down explicitly with `LET`.
- Rebinding an existing alias name (for example `LET o = o`) is a compile-time error.
- Non-goal: no DTL-style implicit ancestor references (`parent`, `parent.parent`, `root`).

Examples:

```sql
-- Valid
FROM orders AS o {
	LET total = o.total_amount,

	SET lines = COLLECT order_lines AS l ON l.order_id = o.id {
		SET share = l.amount / total
	}
}

-- Invalid
FROM orders AS o {
	SET lines = COLLECT order_lines AS l ON l.order_id = o.id {
		SET bad = o.total_amount
	}
}
```

## Rationale

- Keeps meaning stable when nesting depth changes.
- Enables useful reuse via explicit `LET` handoff.
- Avoids ambiguous name resolution and hidden coupling.
- Produces simpler, deterministic compiler diagnostics.

## Consequences

- Authors hoist needed outer values to `LET` before nested use.
- Compiler tracks lexical `LET` scopes and shadowing.
- Errors distinguish unknown name vs alias-out-of-scope vs illegal alias rebinding.
