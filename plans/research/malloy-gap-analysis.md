# Gap Analysis: Malloy Quickstart vs pg_weave

> **Date:** 2026-02-26  
> **Source:** https://docs.malloydata.dev/documentation/user_guides/basic.html  
> **Status:** Internal analysis

## Scope

Feature-by-feature comparison of every concept demonstrated in the Malloy Quickstart against the current pg_weave language (as documented in `docs/language.md` and `docs/grammar.md`). The goal is to surface genuine gaps — not to chase feature parity with a multi-database analytics language.

## Feature comparison

### Covered — direct equivalent exists

| Malloy concept | pg_weave equivalent | Notes |
|---|---|---|
| `select:` (project columns) | `SET field = expr` | Both emit named output fields. pg_weave requires explicit alias. |
| `select: *` with all columns | `SET *` / `SET * EXCEPT (...)` | Same intent; pg_weave also supports EXCEPT exclusions. |
| Named outputs (`name is expr`) | `SET name = expr` | Both enforce name-first output. Malloy allows bare column names; pg_weave always uses `SET`. |
| SQL expressions in fields | SQL pass-through in `SET`/`LET`/`WHERE` | Malloy supports DB-native expressions; pg_weave passes through to PG. |
| `where:` filtering | `WHERE` (inline + trailing) | pg_weave supports WHERE at both positions; Malloy uses `where:` in queries and sources. |
| Nested output (`nest:`) | `COLLECT ... ON ... { }` | Both produce nested subtables. pg_weave is 1:N join-based; Malloy nests any aggregate query. |
| Nested filtering | `WHERE` inside `COLLECT` blocks | Both isolate filters per nesting level. |
| Infinite nesting depth | COLLECT/MAP/BUILD nestable | Malloy nests views arbitrarily; pg_weave nests COLLECT, MAP, BUILD. |
| Comments (`--`, `//`, `/* */`) | `--` SQL-style comments | Malloy also supports `//` and `/* */`. pg_weave uses `--` (ADR 012). |
| ORDER BY | `ORDER BY` on COLLECT/MAP | Both support ordering of nested results. |
| Joins (1:1 lookup) | `LET x = LOOKUP table ON ...` | Malloy declares joins in sources; pg_weave uses inline LOOKUP per weave. |
| Multi-hop joins | Chained `LOOKUP` bindings | Malloy traverses `a.b.c` through pre-declared joins; pg_weave chains explicit LOOKUPs. |
| Conditional fields | `SET field = expr WHEN pred` + `CASE` | Malloy uses `CASE/WHEN` in expressions; pg_weave adds `SET ... WHEN` sugar. |

### Different by design — not a gap

| Malloy concept | pg_weave position | Why it differs |
|---|---|---|
| `group_by:` + `aggregate:` | Not supported | pg_weave produces row-level VIEW transforms, not analytics queries. Aggregation happens in SQL queries against the VIEW, not in the weave definition. This is intentional: weaves define *shape*, not *analysis*. |
| Measures (`count()`, `avg()`, `sum()`) | Not supported | Same rationale. pg_weave has `COUNT OF` (array length), but general SQL aggregates belong to query-time, not view-time. |
| Measure-level filters (`count() { where: ... }`) | N/A | No measures → no measure filters. Equivalent filtering in pg_weave is `WHERE` on the COLLECT input or SQL `CASE` inside a pass-through aggregate. |
| Aggregate locality (fan-out correction) | N/A | pg_weave doesn't aggregate over joins. COLLECT produces a nested array; LOOKUP is 1:1. No fan-out ambiguity. |
| Pipelines (`->` multi-stage) | Chain weaves via VIEW references | Malloy pipes query output into another query inline. pg_weave compiles each weave to a VIEW; chaining = `FROM view_a AS a { ... }`. Explicit and auditable but more verbose. |
| `limit:` | Not in DSL | VIEWs don't limit rows. COLLECT's `ORDER BY` controls nested array ordering; row limiting is query-time. |

### Genuine gaps — worth evaluating

| # | Malloy concept | Current pg_weave state | Severity | Notes |
|---|---|---|---|---|
| 1 | **Reusable source definitions** | Each weave is standalone | Medium | Malloy's `source: ... extend { }` lets you declare joins, dimensions, measures once and reuse across queries. pg_weave has no model layer — every weave re-declares its LOOKUPs and LET bindings. For projects with many weaves over the same tables, this is repetitive. |
| 2 | **Pre-declared joins** | Inline LOOKUP per weave | Low | Malloy's source-level `join_one:`/`join_many:` means queries don't repeat join conditions. pg_weave makes joins explicit per weave, which is more verbose but also more transparent. |
| 3 | **Reusable named views** | No equivalent | Medium | Malloy's `view:` inside a source is a named, stored query fragment. pg_weave has no way to define reusable partial weave blocks. You'd use SQL VIEWs or repeat the weave logic. |
| 4 | **Import / cross-file references** | No module system | Low | Malloy supports `import "file.malloy"` for cross-file model reuse. pg_weave definitions are self-contained SQL strings. Multi-file support would require a build/compilation step. |
| 5 | **Date/time sugar** | Raw SQL pass-through | Low | Malloy provides `@2003` time literals, `.year`/`.month` truncation, `day()` extraction, and range syntax (`@2003 to @2005`, `@2004-Q1 for 6 quarters`). pg_weave relies on PG-native `date_trunc()`, `EXTRACT()`, and standard date literals. Verbose but fully capable. |
| 6 | **Default ordering heuristics** | No defaults | Low | Malloy auto-sorts by first measure descending. pg_weave requires explicit `ORDER BY`. |
| 7 | **Dimensions vs measures distinction** | No type classification | Low | Malloy's source model distinguishes `dimension:` (group-by-able) from `measure:` (aggregate-able). pg_weave has `SET` (output) and `LET` (intermediate) but no semantic classification. Related to potential semantic annotations (see external landscape research). |

### Out of scope (Malloy features not in quickstart but visible)

| Feature | Why out of scope |
|---|---|
| Window functions / calculations | pg_weave passes SQL window functions through unchanged. |
| Composite "Cube" sources | Analytics feature; not relevant to row-level VIEW transforms. |
| Visualization annotations (`# percent`, etc.) | Rendering concern; pg_weave is data-layer only. |
| Access modifiers | Relevant only with a model/module system (gap #4). |
| DuckDB / BigQuery / Snowflake dialect support | pg_weave is PostgreSQL-only by design (ADR 011). |

## Recommendations

### Consider for future ADRs

1. **Source / model layer** (gaps #1, #2, #3) — The biggest ergonomic gap. A lightweight `SOURCE` or `MODEL` declaration that pre-binds LOOKUPs, default WHERE clauses, and reusable LET bindings would reduce repetition across weaves targeting the same tables. This is the single highest-leverage idea from Malloy.

2. **Semantic annotations** (gap #7) — Already identified in [external landscape research](external-landscape-2026-02.md). A minimal `DIMENSION`/`MEASURE` annotation could improve tooling without affecting compilation.

### Defer

3. **Import system** (gap #4) — Only valuable once a model layer exists. No benefit without reusable definitions to import.

4. **Date/time sugar** (gap #5) — PG-native syntax is sufficient. Adding DSL sugar would increase parser complexity for marginal readability gain.

5. **Default ordering** (gap #6) — Low value for VIEW definitions. Ordering matters mainly in COLLECT arrays, which already have explicit ORDER BY.

## Summary

pg_weave already covers the core structural operations from Malloy's quickstart: named projection, filtering, nested output, joins, and composable expressions. The main gap is **modeling infrastructure** — reusable source definitions, pre-declared joins, and named view fragments. These reduce repetition in real projects but require careful design to avoid undermining pg_weave's current transparency-first approach where every weave is self-contained and auditable.
