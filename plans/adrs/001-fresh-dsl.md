# ADR 001: Fresh SQL-Inspired DSL

**Status:** Accepted  
**Date:** 2026-02-22

## Context

Sesam DTL uses JSON-array S-expression syntax: `["add", "name", ["concat", "_S.first", " ", "_S.last"]]`. This is machine-friendly but hard to read, write, and debug.

## Decision

Design a new SQL-inspired declarative DSL. Key constructs: `FROM` (transform body), `LOOKUP`/`COLLECT`/`MAP`/`FLATMAP` (data operations), `SCHEMA` (type declarations). Expressions use familiar infix operators with explicit iteration (`MAP array AS alias`).

## Rationale

- **Readability** — SQL-like syntax is familiar to the target audience (data engineers, DBAs)
- **No legacy baggage** — no Transit encoding, no `_S.`/`_T.` context prefixes, no `apply-hops` indirection
- **Tooling** — SQL-like syntax is easier to syntax-highlight, lint, and autocomplete
- **Smaller surface area** — DTL has ~80 functions; pg_weave can start with a focused core and grow

## Consequences

- No migration path from existing DTL transforms — manual rewrite required
- Need to design and stabilize a grammar before implementation
- Parser complexity: must handle path expressions, object literals, and SQL-like clauses
