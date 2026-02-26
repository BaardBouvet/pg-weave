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
