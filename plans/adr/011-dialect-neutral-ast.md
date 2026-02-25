# ADR 011: Dialect-Neutral AST — PG-Only for Now

**Status:** Accepted  
**Date:** 2026-02-22

## Context

pg_weave generates heavily PG-specific SQL: `jsonb_build_object`, `jsonb_agg`, `jsonb_array_elements`, `LEFT JOIN LATERAL`, composite type access, `CREATE VIEW`. Other platforms with IVM-like capabilities (Snowflake Dynamic Tables, BigQuery Materialized Views, DuckDB) use different SQL dialects — e.g. Snowflake uses `OBJECT_CONSTRUCT`, `LATERAL FLATTEN`, `VARIANT` paths, and `CREATE DYNAMIC TABLE`.

A sqlglot-based transpiler could handle ~40-50% of constructs (scalar expressions, JSON path operators), but structural patterns (LATERAL joins, array functions, output DDL) differ too much for post-processing.

## Decision

Stay **PG-only** for SQL generation. Keep the **AST dialect-neutral** so dialect-specific backends can be added later as a clean extension.

Concretely:
- The AST represents operations (Lookup, Collect, Map, Flatmap, Set, Let) — not PG functions
- `generate_sql(ast, schema_provider)` targets PG exclusively for now
- No PG-specific constructs leak into the AST (no `JsonbAgg` AST nodes — use `Collect` instead)
- Future: add `dialect` parameter → `generate_sql(ast, schema_provider, dialect)` with per-dialect backends

## Rationale

- **Scope control** — multi-dialect backends are significant work; PG is the primary target
- **Architecture ready** — dialect-neutral AST means adding Snowflake/BigQuery/DuckDB backends is extension, not rewrite
- **Library-first** (per [ADR 007](007-library-first-architecture.md)) — pg_weave-sql already separates parsing from SQL generation; dialect backends slot in naturally
- **No premature abstraction** — building one backend well reveals the right abstraction points

## Consequences

- Only PostgreSQL is supported initially
- AST design must avoid embedding PG-specific SQL constructs — review AST nodes during implementation
- Adding a new dialect later requires: a new backend module, dialect-specific SQL emission, and a test suite
- sqlglot may still be useful for simple expression-level transpilation within a dialect backend
