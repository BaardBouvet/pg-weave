# ADR 005: Graph-Aware Traversals via COLLECT FOLLOW

**Status:** Proposed  
**Date:** 2026-02-22

## Context

Sesam DTL "hops" are single-level key-value lookups that compile to `LEFT JOIN LATERAL`. Nested hops (via `apply-hops`) stack LATERALs. Recursive hops exist but are limited. There's no built-in concept of graph depth, cycle detection, path tracking, or bidirectional traversal.

## Decision

Graph walks are expressed as `COLLECT ... FOLLOW field DEPTH N` — an extension of the COLLECT syntax rather than a separate top-level construct. This compiles to `WITH RECURSIVE` CTEs with built-in cycle detection, depth limits, and path tracking.

Two forms:
- `COLLECT dataset AS alias FOLLOW field DEPTH N { ... }` — flat array of visited nodes
- `COLLECT RECURSIVE dataset AS alias FOLLOW field DEPTH N { ... }` — nested tree

Simple single-hop lookups use `LET` + `LOOKUP ... ON`.

## Rationale

- **Natural fit** — COLLECT gathers related data into arrays; graph walks are just multi-hop collection
- **Real graph operations** — organizational hierarchies, supply chains, social networks need depth + cycle awareness
- **Built-in safety** — cycle detection and depth limits prevent runaway recursion
- **Path as data** — `PATH()` and `DEPTH()` functions expose traversal metadata

## Consequences

- `WITH RECURSIVE` CTEs may be harder for IVM extensions to maintain incrementally
- Graph traversals over large datasets can be expensive — need depth limits and index guidance
- Bidirectional traversal requires generating reverse-edge subqueries

## Example: hobbies of all female descendants

Hop 1 uses `FOLLOW` (recursive, same dataset). Hop 2 is a regular `COLLECT ... ON` (cross-dataset join). The result should be a flat list of hobby names across all female descendants.

```sql
FROM person AS p {
    -- hop 1 (recursive): follow children, keep only females
    LET daughters = COLLECT person AS d FOLLOW children
        WHERE d.gender = 'female' {
        SET id      = d.id,
        SET hobbies = d.hobbies
    },

    -- hop 2: collect hobby names via SQL flatten
    SET hobby_names = COLLECT hobby AS h
        ON h.id IN (SELECT unnest(d.hobbies) FROM unnest(daughters) AS d)
        -> h.name
}
```

**Open question:** flattening nested collection results currently requires SQL pass-through (`unnest`). A DSL-native flatten (e.g. expression-form `FLATMAP` with `->`) would make chained hops more ergonomic.
