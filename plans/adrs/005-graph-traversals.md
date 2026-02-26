# ADR 005: Graph-Aware Traversals via COLLECT FOLLOW

**Status:** Accepted  
**Date:** 2026-02-22

## Context

Sesam DTL "hops" are single-level key-value lookups that compile to `LEFT JOIN LATERAL`. Nested hops (via `apply-hops`) stack LATERALs. Recursive hops exist but are limited. There's no built-in concept of graph depth, cycle detection, path tracking, or bidirectional traversal.

## Decision

Graph walks are expressed as `COLLECT ... FOLLOW field DEPTH N` — an extension of the COLLECT syntax rather than a separate top-level construct. This compiles to `WITH RECURSIVE` CTEs with built-in cycle detection, depth limits, and path tracking.

Syntax: `COLLECT dataset AS alias FOLLOW field DEPTH N { ... }`

Output shape is determined by the block contents:
- No `CHILDREN()` in the block → flat array of visited nodes
- `CHILDREN()` used in the block → nested parent→children tree

### Root inclusion

The source row is included in the result by default — "this node and its subtree." The root is shaped through the same `{ SET ... }` block as every other node, with `DEPTH() = 0` and `PATH()` containing only the root's own ID.

Use `EXCLUDING ROOT` to return only the traversed descendants:

```sql
-- Default: root + descendants
SET subtree = COLLECT employees AS e FOLLOW manager_id DEPTH 5 { ... }

-- Descendants only
SET reports = COLLECT employees AS e FOLLOW manager_id DEPTH 5 EXCLUDING ROOT { ... }
```

`EXCLUDING ROOT` is only valid with the `FOLLOW` form.

### Flat vs nested output

All `COLLECT ... FOLLOW` forms perform a recursive walk with the same safety rules (cycle detection + depth limit). The output shape is determined by whether the block uses `CHILDREN()`:

```sql
-- Flat: no CHILDREN() → flat list of visited nodes
SET descendants = COLLECT people AS d FOLLOW children DEPTH 3 {
    SET id = d.id,
    SET name = d.name
}

-- Nested: CHILDREN() → parent→children tree
SET descendants_tree = COLLECT people AS d FOLLOW children DEPTH 3 {
    SET id = d.id,
    SET name = d.name,
    SET sub = CHILDREN()
}
```

Simple single-hop lookups use `LET` + `LOOKUP ... ON`.

### Traversal metadata helpers

`PATH()` and `DEPTH()` are DSL helpers for `COLLECT ... FOLLOW` traversals. They are **not** PostgreSQL built-ins.

- `DEPTH()` → `integer`
    - Hop distance from the source row to the current visited node
    - Root is depth `0`, first hop is `1`, next is `2`, etc.
- `PATH()` → `text[]`
    - Ordered visited-node identifiers along the traversal path
    - From root to current node (inclusive)
- `CHILDREN()` → `jsonb`
    - The nested array of child nodes for the current node
    - Only valid inside `COLLECT ... FOLLOW` blocks
    - Presence of `CHILDREN()` switches output from flat list to nested tree
    - The user chooses the field name via `SET whatever = CHILDREN()`
    - Composable with `WHEN` and other expressions: `SET sub_depts = CHILDREN() WHEN DEPTH() < 3`

Scope and validity:
- All three helpers are valid only inside `COLLECT ... FOLLOW ...` blocks
- Compile-time error if used outside traversal context
- Lowered from traversal state columns in generated `WITH RECURSIVE` SQL

Example usage:

```sql
FROM employees AS e {
    SET management_chain = COLLECT employees AS m FOLLOW manager_id DEPTH 5 {
        SET id         = m.id,
        SET level      = DEPTH(),
        SET path       = PATH(),
        SET breadcrumb = array_to_string(PATH(), ' > '),
        SET near_top   = DEPTH() <= 2
    }
}
```

Example output (root included by default):

```json
[
    {"id": "e_1",    "level": 0, "path": ["e_1"],                         "near_top": true},
    {"id": "vp_1",   "level": 1, "path": ["e_1", "vp_1"],                 "near_top": true},
    {"id": "dir_9",  "level": 2, "path": ["e_1", "vp_1", "dir_9"],        "near_top": true},
    {"id": "mgr_42", "level": 3, "path": ["e_1", "vp_1", "dir_9", "mgr_42"], "near_top": false}
]
```

Typical uses: reporting level distance (`DEPTH()`), rendering breadcrumb/debug paths (`PATH()`), and applying depth-based flags.

### Nested tree example

```sql
FROM departments AS dept {
    SET org_tree = COLLECT departments AS d FOLLOW parent_id DEPTH 4 {
        SET id        = d.id,
        SET name      = d.name,
        SET level     = DEPTH(),
        SET sub_depts = CHILDREN() WHEN COUNT OF CHILDREN() > 0
    }
}
```

Example output (`sub_depts` omitted on leaf nodes):

```json
{
    "id": "eng",
    "name": "Engineering",
    "level": 0,
    "sub_depts": [
        {
            "id": "backend",
            "name": "Backend",
            "level": 1,
            "sub_depts": [
                {"id": "api", "name": "API Team", "level": 2},
                {"id": "infra", "name": "Infrastructure", "level": 2}
            ]
        },
        {
            "id": "frontend",
            "name": "Frontend",
            "level": 1
        }
    ]
}
```

## Rationale

- **Natural fit** — COLLECT gathers related data into arrays; graph walks are just multi-hop collection
- **Real graph operations** — organizational hierarchies, supply chains, social networks need depth + cycle awareness
- **Built-in safety** — cycle detection and depth limits prevent runaway recursion
- **Path as data** — `PATH()` and `DEPTH()` functions expose traversal metadata

## Consequences

- `WITH RECURSIVE` CTEs may be harder for IVM extensions to maintain incrementally
- Graph traversals over large datasets can be expensive — need depth limits and index guidance

Out of scope for this ADR: bidirectional traversal (forward + reverse edge walking) and its SQL lowering strategy.

## Example: hobbies of all female descendants

Hop 1 uses `FOLLOW` (recursive, same dataset). `FLATMAP` inside the `COLLECT` block flattens the second hop into a single array of hobby names.

```sql
FROM person AS p {
    SET hobby_names = COLLECT person AS d FOLLOW children
        WHERE d.gender = 'female' {
        -- each daughter may have multiple hobbies; FLATMAP flattens them
        FLATMAP d.hobbies AS hobby_id {
            LET h = LOOKUP hobby ON h.id = hobby_id,
            SET name = h.name
        }
    }
}
```
