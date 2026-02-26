# ADR 007: Library-First Architecture

**Status:** Accepted  
**Date:** 2026-02-22

## Context

pg_weave compiles a DSL to SQL VIEWs — it does no work at query time. All expressions transpile to SQL, so the extension glue is very thin — parse text, run `CREATE VIEW`.

This means pg_weave could work as a PG extension, a dbt plugin, or both. The heavy lifting is text-to-text compilation.

## Decision

**Rust library-first.** `pg_weave-sql` is the core crate — parser, AST, SQL generation — with no pgrx dependency. Consumers are built on top:

- **pg_weave-ext** (primary) — thin pgrx wrapper, PG extension
- **pg_weave-dbt** (future) — dbt plugin for file-based workflows, works with any PG hosting

## Rationale

- **Rust keeps doors open** — extension via pgrx, dbt plugin integration, WASM if needed later
- **Library is testable** — `pg_weave-sql` has zero PG dependencies, full unit test coverage
- **Extension is the right primary** — in-database UX, catalog storage, single deployment
- **dbt plugin gives reach** — RDS, Cloud SQL, Supabase can't install custom extensions; a dbt plugin works everywhere
- **No runtime evaluation** — everything compiles to SQL, so there's no performance argument for or against any language; Rust's value is in keeping multiple consumer options viable from one codebase

## Consequences

- Two crates now, optional third later — workspace stays small
- `pg_weave-sql` public API must be clean and stable since multiple consumers depend on it
- dbt plugin is explicitly deferred — not blocking v1
