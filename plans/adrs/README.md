# pg_weave ADRs

Architecture Decision Records for the pg_weave project.

| # | Title | Status |
|---|-------|--------|
| [001](001-fresh-dsl.md) | Fresh SQL-Inspired DSL | Accepted |
| [005](005-graph-traversals.md) | Graph-Aware Traversals | Accepted |
| [007](007-library-first-architecture.md) | Library-First Architecture | Accepted |
| [008](008-dsl-syntax-constructs.md) | DSL Syntax Constructs — SET, LET, WHEN | Accepted |
| [009](009-fp-aligned-keywords.md) | FP-Aligned Keywords — LOOKUP, COLLECT, MAP, FLATMAP | Accepted |
| [010](010-explicit-schema-model.md) | Catalog Introspection Data Model | Accepted |
| [011](011-dialect-neutral-ast.md) | Dialect-Neutral AST — PG-Only for Now | Accepted |
| [012](012-sql-style-comments.md) | SQL-Style Comments (`--`) | Accepted |
| [013](013-sql-pass-through-expressions.md) | SQL Pass-Through Expressions | Accepted |
| [014](014-where-clause.md) | WHERE Clause — Inline and Block-Trailing | Accepted |
| [015](015-set-wildcard.md) | SET Wildcard (`SET *`, `EXCEPT`) | Accepted |
| [016](016-collect-output-types.md) | Nested Output Defaults — JSONB Without Output Schema | Accepted |
| [017](017-array-length-helper.md) | Polymorphic Array-Length Helper | Accepted |
| [018](018-polymorphic-collection-helpers.md) | Polymorphic Collection Helpers (Beyond COUNT OF) | Accepted |
| [019](019-build-constructor.md) | BUILD Constructor for Nested Non-Array Output | Accepted |
| [021](021-set-conditional-sugar.md) | WHEN PRESENT Sugar for Conditional SET | Rejected |
| [022](022-stage-locality-and-visibility.md) | Stage Locality and Visibility Rules | Accepted |
