# Gap Analysis: ADR 005 vs DTL Recursive Hops

> **Date:** 2026-02-26

## Feature Comparison

| Capability | DTL `hops` with `recurse: true` | ADR 005 `COLLECT ... FOLLOW` |
|---|---|---|
| Recursive traversal | `"recurse": true` — feeds output back as input until exhausted | `FOLLOW field` — walks the FK chain via `WITH RECURSIVE` CTE |
| Depth limit | `max_depth` (default **10**, `null` = unlimited) | `DEPTH N` (mandatory, explicit) |
| Cycle detection | Implicit — stops when no new entities appear | Explicit — built-in cycle detection in generated CTE |
| Root inclusion | **Included by default** (`exclude_root: false`) | **Included by default** — root goes through the same `{ SET ... }` block |
| Root exclusion | `"exclude_root": true` | `EXCLUDING ROOT` modifier |
| Output shape: flat | Always flat (list of entities) | Default (no `RECURSIVE` keyword) |
| Output shape: nested tree | Not supported | `RECURSIVE` keyword → nested parent→children structure |
| Path tracking | Not supported | `PATH()` → `text[]` of visited-node IDs |
| Depth metadata | Not supported | `DEPTH()` → `integer` hop distance |
| Transform inside recursion | `return` **cannot** be used with `recurse` — raw entities only | `{ SET ... }` block reshapes each visited node inline |
| Chained hops after recursion | Native — add another HOPS_SPEC in the chain | `FLATMAP` + `LOOKUP` inside the COLLECT block |
| Multi-dataset hop | `"datasets": ["Person p", "Company c"]` — multiple datasets in one step | Single dataset only — `COLLECT dataset AS alias FOLLOW field` |
| Filter during traversal | `where` expressions in HOPS_SPEC | `WHERE` clause on the COLLECT |
| Pre-filter optimization | `subsets` — index-based pre-filtering before join | No equivalent — WHERE filters post-join |
| Dependency tracking | `track-dependencies` flag (Sesam pipe system) | N/A — VIEWs have no pipe dependencies |
| Execution tracing | `trace` property for join cardinality stats | N/A — use `EXPLAIN ANALYZE` on the generated SQL |

---

## Gaps: DTL has, ADR 005 doesn't

### ~~1. Root inclusion~~ — Resolved

Root is now included by default (matching DTL). Use `EXCLUDING ROOT` to opt out. See updated ADR 005.

### 2. Unlimited depth (`max_depth: null`) — Won't fix

DTL allows `max_depth: null` for unbounded recursion (terminates when no new entities appear). ADR 005 requires an explicit `DEPTH N`.

**Impact:** Forces the user to guess a maximum depth. For self-referential structures of unknown depth (e.g., bill-of-materials), this is limiting.

**Workaround:** Pick a conservatively large `DEPTH` value. PostgreSQL's `WITH RECURSIVE` naturally terminates via the cycle-detection clause even without a depth cap.

**Decision:** `DEPTH N` stays mandatory. Unbounded `WITH RECURSIVE` on large graphs is too risky. Forcing users to think about depth is a safety feature, not a limitation.

### 3. Multi-dataset recursion

DTL supports `"datasets": ["Person p", "Company c"]` — a single hop can join across multiple datasets. ADR 005's `COLLECT dataset AS alias FOLLOW field` is single-dataset.

**Impact:** Heterogeneous graph traversal (person → company → person) can't be expressed as a single recursive COLLECT. Requires multiple nested COLLECTs or manual SQL.

**Workaround:** Chain multiple `COLLECT` expressions or use a UNION ALL SQL VIEW upstream that combines the datasets into a single "edges" table.

**Recommendation:** Low priority. Multi-dataset recursion is rare and the workaround (upstream UNION VIEW) is clean. Document it as out-of-scope rather than trying to support it.

### 4. Pre-filter subsets

DTL's `subsets` property lets you select an index-backed subset of a dataset before joining, which is a performance optimization.

**Impact:** pg_weave relies on PostgreSQL's query planner to push predicates down. If WHERE filters are selective and indexed, the planner achieves the same effect.

**Workaround:** Ensure appropriate indexes exist. PostgreSQL's optimizer typically handles predicate pushdown well.

**Recommendation:** No action — this is a Sesam-specific optimization. PostgreSQL's planner covers it.

---

## Gaps: ADR 005 has, DTL doesn't

### 1. Nested tree output (`RECURSIVE`)

DTL recursive hops always return a flat list. ADR 005's `RECURSIVE` keyword produces a nested parent→children JSON tree.

**Significance:** Major ergonomic win for hierarchical UIs, org charts, threaded comments, etc. Building nested trees from flat lists requires client-side logic in DTL.

### 2. Path and depth metadata (`PATH()`, `DEPTH()`)

DTL provides no traversal metadata — you get back the raw entities with no information about how they were reached.

**Significance:** Enables breadcrumb trails, depth-based styling, level-aware business rules — all common requirements for hierarchical data.

### 3. Inline transform during recursion

DTL's `return` clause **cannot** be used with `recurse: true`. Recursive hops return raw entities; reshaping requires a separate `apply` step after the hops.

ADR 005 allows a `{ SET ... }` block directly inside the recursive COLLECT, transforming each visited node as it's traversed.

**Significance:** Eliminates a separate transform step. Each node is shaped exactly once during traversal.

### 4. Cycle detection as explicit safety guarantee

DTL's implicit termination (no new entities → stop) handles cycles but doesn't surface them. ADR 005 names cycle detection as a first-class design goal and uses PostgreSQL's `CYCLE` clause or path-based detection.

**Significance:** Predictable behavior in graphs with cycles. Users know cycles won't cause errors — they'll just stop being followed.

---

## Summary

| Category | Count | Items |
|---|---|---|
| DTL has, ADR missing | 2 | Multi-dataset, subsets (low priority) |
| Resolved | 1 | Root inclusion (now default, with `EXCLUDING ROOT`) |
| Won't fix | 1 | Unlimited depth (mandatory `DEPTH N` is a safety feature) |
| ADR has, DTL missing | 4 | Nested tree, PATH/DEPTH, inline transform, explicit cycle detection |
| Equivalent | 3 | Recursive traversal, depth limit, filter during traversal |
| N/A (platform-specific) | 2 | Dependency tracking, execution tracing |

**Assessment:** Root inclusion gap is closed (now default, with `EXCLUDING ROOT` opt-out). Unlimited depth is intentionally rejected — mandatory `DEPTH N` is a safety feature. Remaining DTL-side gaps (multi-dataset, subsets) are low priority and handled by PostgreSQL's architecture. ADR 005 is strictly more powerful on output (nested trees, traversal metadata, inline transforms).
