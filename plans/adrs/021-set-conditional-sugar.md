# ADR 021: WHEN PRESENT Sugar for Conditional SET

**Status:** Rejected  
**Date:** 2026-02-26

## Context

We considered adding a trailing sugar form (`WHEN PRESENT`) for:

```sql
SET phone = p.phone WHEN PRESENT
SET sub_depts = CHILDREN() WHEN PRESENT
```

Intended lowering:
- scalar expressions → `expr IS NOT NULL`
- collections → `expr IS NOT NULL AND COUNT OF expr > 0`

## Decision

Reject this feature. Keep explicit `SET ... WHEN ...` only.

```sql
SET phone = p.phone WHEN p.phone IS NOT NULL
SET sub_depts = CHILDREN() WHEN COUNT OF CHILDREN() > 0
```

## Rationale

- Adds syntax surface for small ergonomic gain
- Introduces type-dependent lowering rules that are less obvious
- Slightly reduces readability versus explicit predicates
- Existing `WHEN` already solves the use case consistently

## Consequences

- No parser changes for new trailing sugar keywords
- No extra typing/lowering rules for scalar vs collection expressions
- Documentation remains simpler: one conditional form (`WHEN`)
