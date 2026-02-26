# Gap Analysis: DTL Examples → pg_weave Equivalents

> **Date:** 2026-02-26
> **Status:** Under discussion
> **Source:** https://docs.sesam.io/hub/dtl/

This document walks through every major DTL example from the Sesam documentation and shows the pg_weave equivalent, flagging gaps where they exist.

---

## 1. Annotated Example (person → person-with-orders)

### DTL

```json
["copy", "_id"],
["add", "type", "customer"],
["add", "name", ["upper", "_S.name"]],
["add", "orders",
  ["sorted", "_.amount", ["apply-hops", "order", {
    "datasets": ["orders o"],
    "where": [["eq", "_S._id", "o.cust_id"]]
  }]]],
["add", "order_count", ["count", "_T.orders"]],
["filter", ["gt", "_T.order_count", 10]]
```

With the `order` rule:

```json
["copy", "_id"],
["add", "amount", "_S.amount"]
```

### pg_weave

```sql
FROM person AS p {
    SET id    = p.id,
    SET type  = 'customer',
    SET name  = UPPER(p.name),
    LET orders = COLLECT orders AS o ON o.cust_id = p.id {
        SET id     = o.id,
        SET amount = o.amount
    } ORDER BY amount ASC,
    SET orders      = orders,
    SET order_count = COUNT OF orders,
    WHERE order_count > 10
}
```

**Gaps:** None. `COLLECT` replaces `apply-hops` + `order` rule. `ORDER BY` replaces `sorted`. Trailing `WHERE` replaces `filter`. `COUNT OF` replaces `count`.

---

## 2. Transform Functions

### 2.1 `add` / `SET`

| DTL | pg_weave |
|---|---|
| `["add", "age", 26]` | `SET age = 26` |
| `["add", "upper_name", ["upper", "_S.name"]]` | `SET upper_name = UPPER(s.name)` |
| `["add", ["concat", "x-", "_S.key"], "_S.value"]` | **Gap:** dynamic field names not supported. pg_weave field names are static identifiers. |

**Gap:** Dynamic field names computed at runtime are not supported and are unlikely to be — pg_weave produces VIEWs with fixed schemas.

### 2.2 `add-if` / `SET ... WHEN`

| DTL | pg_weave |
|---|---|
| `["add-if", "foo", "_S.bar"]` | `SET foo = s.bar WHEN s.bar IS NOT NULL` |
| `["add-if", "foo", ["gt", "_.", 0], "_S.bar"]` | `SET foo = s.bar WHEN s.bar > 0` |

**Gaps:** None. `SET ... WHEN` covers `add-if` semantics.

### 2.3 `copy` / `SET *`

| DTL | pg_weave |
|---|---|
| `["copy", "_id"]` | `SET id = s.id` |
| `["copy", "a*", "ab*"]` | `SET * EXCEPT (ab*)` (ADR 015, Proposed) |

**Gap:** Wildcard copy with exclude patterns is proposed (ADR 015) but not yet accepted.

### 2.4 `rename` / `SET new = old`

| DTL | pg_weave |
|---|---|
| `["rename", "age", "current_age"]` | `SET current_age = s.age` |

**Gaps:** None.

### 2.5 `remove`

| DTL | pg_weave |
|---|---|
| `["remove", "age"]` | Simply omit `SET age` from the block. |
| `["remove", "temp_*"]` | `SET * EXCEPT (temp_*)` (ADR 015, Proposed) |

**Gaps:** None — this is natural in a projection model.

### 2.6 `default` / `COALESCE`

| DTL | pg_weave |
|---|---|
| `["default", "age", 26]` | `SET age = COALESCE(s.age, 26)` |

**Gaps:** None. SQL pass-through.

### 2.7 `merge`

| DTL | pg_weave |
|---|---|
| `["merge", "_S.orders"]` | No direct equivalent. Would need multiple `LOOKUP` + individual `SET` per field, or SQL `jsonb_each`. |

**Gap:** No merge/spread operator to unpack all fields from a nested entity onto the output. Use explicit `SET` per field or SQL JSONB functions.

### 2.8 `filter` / `discard` → `WHERE`

| DTL | pg_weave |
|---|---|
| `["filter", ["gt", "_S.age", 42]]` | `WHERE s.age > 42` (inline or trailing) |
| `["discard", ["gt", "_S.age", 42]]` | `WHERE s.age > 42` (same — no entity lifecycle to manage) |

**Gaps:** None.

### 2.9 `if` / `case` / `case-eq` → SQL `CASE`

| DTL | pg_weave |
|---|---|
| `["if", ["eq", "_S.type", "person"], ["add", "type", "person"]]` | `SET type = CASE WHEN s.type = 'person' THEN 'person' END` |
| `["case", ["gte", "_S.age", 18], "adult", ...]` | `SET group = CASE WHEN s.age >= 18 THEN 'adult' WHEN s.age >= 13 THEN 'teenager' ELSE 'unknown' END` |
| `["case-eq", "_S.country", "NO", "Norway", ...]` | `SET country = CASE s.country WHEN 'NO' THEN 'Norway' WHEN 'SE' THEN 'Sweden' ELSE 'Other' END` |

**Gaps:** None. SQL `CASE` is a superset.

### 2.10 `create` → `FLATMAP`

| DTL | pg_weave |
|---|---|
| `["create", "_S.orders"]` | `FLATMAP s.orders AS o { SET id = o.id, ... }` |

**Gaps:** None. `FLATMAP` explodes nested arrays into rows.

### 2.11 `create-child` → nested `COLLECT`

| DTL | pg_weave |
|---|---|
| `["create-child", "_S.orders"]` | `SET orders = COLLECT ...` (nested array output) |

**Gaps:** None.

### 2.12 `comment`

| DTL | pg_weave |
|---|---|
| `["comment", "This is a comment"]` | `-- This is a comment` (SQL-style comments) |

**Gaps:** None.

### 2.13 `fail!`, `sleep!`, `trip!`

**Intentional gap.** These are imperative side-effects. pg_weave produces declarative VIEWs — there is no pipeline to fail or pause.

---

## 3. Path Expressions and Hops

### 3.1 Property path strings

| DTL | pg_weave |
|---|---|
| `"_S.orders.amount"` | `s.orders.amount` (dot-path, compiles to JSONB extraction) |

**Gaps:** None.

### 3.2 `hops` (single hop)

DTL:
```json
["hops", {
    "datasets": ["orders o"],
    "where": [
        ["eq", "_S._id", "o.cust_id"],
        ["eq", "o.type", "BILLING"]
    ]
}]
```

pg_weave:
```sql
COLLECT orders AS o ON o.cust_id = s.id
    WHERE o.type = 'BILLING' {
    SET id     = o.id,
    SET amount = o.amount
}
```

**Gaps:** None. `COLLECT ... ON ... WHERE` maps directly to single-hop `hops`.

### 3.3 `hops` (recursive)

DTL:
```json
["hops", {
    "datasets": ["Person p"],
    "where": [["eq", "_S.children", "p._id"]],
    "recurse": true
}]
```

pg_weave (ADR 005, Proposed):
```sql
COLLECT person AS d FOLLOW children
    WHERE d.gender = 'female' {
    SET id = d.id
}
```

**Gap:** `COLLECT ... FOLLOW` is Proposed (ADR 005), not yet accepted.

### 3.4 `hops` (chained — hobbies of female descendants)

DTL:
```json
["hops", {
    "datasets": ["Person p"],
    "where": [["eq", "_S.children", "p._id"], ["eq", "p.gender", "female"]],
    "recurse": true
}, {
    "datasets": ["Hobby h"],
    "where": ["eq", "_S.hobbies", "h._id"],
    "return": "h.name"
}]
```

pg_weave:
```sql
COLLECT person AS d FOLLOW children
    WHERE d.gender = 'female' {
    FLATMAP d.hobbies AS hobby_id {
        LET h = LOOKUP hobby ON h.id = hobby_id,
        SET name = h.name
    }
}
```

**Gap:** Depends on ADR 005 (Proposed). But the `FLATMAP`-inside-`COLLECT` pattern cleanly solves the flatten problem.

### 3.5 `apply-hops` (hop + transform rule)

DTL:
```json
["apply-hops", "order", {
    "datasets": ["orders o"],
    "where": ["eq", "_S._id", "o.cust_id"]
}]
```

pg_weave:
```sql
COLLECT orders AS o ON o.cust_id = p.id {
    SET id     = o.id,
    SET amount = o.amount
}
```

**Gaps:** None. `COLLECT` with a block replaces `apply-hops` + named rule. No separate "rules" abstraction needed — the block *is* the transform.

### 3.6 `apply` (transform rule on values)

DTL:
```json
["apply", "order", "_S.orders"]
```

pg_weave:
```sql
MAP s.orders AS o {
    SET id     = o.id,
    SET amount = o.amount
}
```

**Gaps:** None. `MAP` with a block replaces `apply` + named rule.

### 3.7 `lookup-entity`

DTL:
```json
["lookup-entity", "code-table", "foo"]
```

pg_weave:
```sql
LET entry = LOOKUP code_table ON id = 'foo'
```

**Gaps:** None.

---

## 4. Join Semantics

### 4.1 One-to-one

| DTL | pg_weave |
|---|---|
| `["eq", "a.value", "b.value"]` | `LOOKUP b ON b.value = a.value` |

### 4.2 One-to-many / Many-to-many

| DTL | pg_weave |
|---|---|
| `["eq", "a.value", "b.values"]` | `COLLECT b ON a.value = ANY(b.values) { ... }` |
| `["eq", "a.values", "b.values"]` | `COLLECT b ON b.values && a.values { ... }` (PG array overlap) |

**Gaps:** None — PG array operators cover all join patterns.

---

## 5. List/Array Functions

### 5.1 `count` → `COUNT OF`

| DTL | pg_weave |
|---|---|
| `["count", "_S.orders"]` | `COUNT OF s.orders` |

**Gaps:** None.

### 5.2 `sorted` → `ORDER BY`

| DTL | pg_weave |
|---|---|
| `["sorted", "_.amount", "_S.orders"]` | `COLLECT ... { ... } ORDER BY amount ASC` |
| `["sorted-descending", ...]` | `ORDER BY amount DESC` |

**Gaps:** None.

### 5.3 `first` / `last` / `nth`

| DTL | pg_weave |
|---|---|
| `["first", "_S.tags"]` | `s.tags[1]` (PG arrays are 1-based) or `s.tags->0` (JSONB) |
| `["last", "_S.tags"]` | `s.tags[array_length(s.tags, 1)]` or JSONB equivalent |
| `["nth", 1, "_S.tags"]` | `s.tags[2]` (PG, 1-based) or `s.tags->1` (JSONB) |

**Gap (ergonomic):** Proposed helpers `FIRST OF`, `LAST OF`, `NTH` (ADR 018) would be storage-neutral. Currently requires knowing whether the value is JSONB or a typed array.

### 5.4 `distinct`

| DTL | pg_weave |
|---|---|
| `["distinct", "_S.tags"]` | `ARRAY(SELECT DISTINCT unnest(s.tags))` (typed array) or JSONB equivalent |

**Gap (ergonomic):** No DSL helper. Requires SQL pass-through with `unnest`/`DISTINCT`.

### 5.5 `flatten`

| DTL | pg_weave |
|---|---|
| `["flatten", ["list", "_S.sisters", "_S.brothers"]]` | `FLATMAP` inside `COLLECT`, or SQL: `array_cat(s.sisters, s.brothers)` |

**Gap (ergonomic):** `FLATMAP`-inside-`COLLECT` handles the structural case. For simple array concatenation, use SQL `array_cat` or `||`.

### 5.6 `map` / `filter` (list expressions)

| DTL | pg_weave |
|---|---|
| `["map", ["lower", "_."], ["list", "A", "B"]]` | `MAP ARRAY['A','B'] AS v -> LOWER(v)` |
| `["filter", ["gt", "_.age", 42], "_S.people"]` | `MAP s.people AS p WHERE p.age > 42 -> p` (using `MAP` with `WHERE`) or bind and filter |

**Gap:** `MAP ... WHERE` acts as filter. No separate `FILTER` keyword, but inline `WHERE` on `MAP` serves the purpose.

### 5.7 `sum` / `max` / `min`

| DTL | pg_weave |
|---|---|
| `["sum", "_S.amounts"]` | SQL: `(SELECT SUM(v) FROM unnest(s.amounts) AS v)` |
| `["max", "_S.amounts"]` | SQL: `(SELECT MAX(v) FROM unnest(s.amounts) AS v)` |

**Gap (ergonomic):** SQL subquery with `unnest` is verbose. PG doesn't have `array_sum()` built-in — these are aggregate functions that need `unnest`.

### 5.8 `in` / `is-empty` / `is-not-empty`

| DTL | pg_weave |
|---|---|
| `["in", "a", ["list", "a", "b"]]` | `'a' = ANY(ARRAY['a','b'])` |
| `["is-empty", "_S.hobbies"]` | `COUNT OF s.hobbies = 0` or `s.hobbies IS NULL` |
| `["is-not-empty", "_S.hobbies"]` | `COUNT OF s.hobbies > 0` |

**Gap:** Proposed `IS EMPTY` / `IS NOT EMPTY` helpers (ADR 018).

### 5.9 `enumerate`

| DTL | pg_weave |
|---|---|
| `["enumerate", "_S.tags"]` | SQL: `SELECT row_number() OVER () - 1, v FROM unnest(s.tags) AS v` |

**Gap:** No DSL helper. Requires SQL pass-through.

### 5.10 `group` / `group-by`

| DTL | pg_weave |
|---|---|
| `["group-by", "_.ean", "_S.orders"]` | No direct DSL equivalent. Use SQL: `SELECT ean, array_agg(...) FROM unnest(...) GROUP BY ean` |

**Gap:** No grouping helper. Requires SQL subquery.

### 5.11 `combine` (concatenate lists)

| DTL | pg_weave |
|---|---|
| `["combine", "_S.a", "_S.b"]` | SQL: `array_cat(s.a, s.b)` or `s.a || s.b` |

**Gaps:** None — SQL array concatenation operators cover this.

### 5.12 `reversed`

| DTL | pg_weave |
|---|---|
| `["reversed", "_S.tags"]` | SQL: `ARRAY(SELECT v FROM unnest(s.tags) WITH ORDINALITY AS t(v,i) ORDER BY i DESC)` |

**Gap:** No DSL helper. Requires SQL pass-through.

### 5.13 `slice` / `range` / `insert`

| DTL | pg_weave |
|---|---|
| `["slice", 2, "_S.tags"]` | `s.tags[3:]` (PG array slicing, 1-based) |
| `["range", 0, 4]` | `ARRAY(SELECT generate_series(0, 3))` |
| `["insert", 1, list, item]` | SQL: `array_cat(list[:1], ARRAY[item] || list[2:])` |

**Gap (ergonomic):** PG has array slicing but syntax is different. No DSL helpers.

---

## 6. Set Functions

| DTL | pg_weave |
|---|---|
| `["union", a, b]` | SQL: `SELECT DISTINCT unnest(a) UNION SELECT DISTINCT unnest(b)` or `array_cat` + `DISTINCT` |
| `["intersection", a, b]` | SQL: `SELECT unnest(a) INTERSECT SELECT unnest(b)` |
| `["difference", a, b]` | SQL: `SELECT unnest(a) EXCEPT SELECT unnest(b)` |
| `["intersects", a, b]` | SQL: `a && b` (PG array overlap operator) |

**Gap (ergonomic):** Proposed set-algebra helpers in ADR 018. Currently requires SQL set operations with `unnest`.

---

## 7. Conditionals (expression form)

| DTL | pg_weave |
|---|---|
| `["if", cond, then, else]` | `CASE WHEN cond THEN then ELSE else END` |
| `["case", cond1, val1, ...]` | `CASE WHEN cond1 THEN val1 ... END` |
| `["case-eq", val, v1, r1, ...]` | `CASE val WHEN v1 THEN r1 ... END` |

**Gaps:** None. SQL `CASE` is a superset.

---

## 8. Dictionary Functions

| DTL | pg_weave |
|---|---|
| `["dict", ...]` | `jsonb_build_object(...)` |
| `["has-key", "a", obj]` | `obj ? 'a'` (JSONB containment) |
| `["keys", obj]` | `jsonb_object_keys(obj)` |
| `["values", obj]` | `SELECT value FROM jsonb_each(obj)` |
| `["items", obj]` | `jsonb_each(obj)` |
| `["path", "age", list]` | Dot-path: `item.age` (compiles to JSONB extraction) |

**Gaps:** None for common cases. SQL JSONB operators cover dictionary operations.

---

## 9. String Functions

| DTL | pg_weave |
|---|---|
| `["upper", "_S.name"]` | `UPPER(s.name)` |
| `["lower", "_S.name"]` | `LOWER(s.name)` |
| `["concat", a, b, c]` | `a \|\| b \|\| c` or `CONCAT(a, b, c)` |
| `["join", ",", list]` | `array_to_string(list, ',')` |
| `["split", ".", str]` | `string_to_array(str, '.')` |
| `["substring", 0, 3, str]` | `SUBSTRING(str FROM 1 FOR 3)` |
| `["replace", "a", "b", str]` | `REPLACE(str, 'a', 'b')` |
| `["matches", "^foo.*", str]` | `str ~ '^foo.*'` |
| `["length", str]` | `LENGTH(str)` |

**Gaps:** None. All SQL pass-through.

---

## 10. Math

| DTL | pg_weave |
|---|---|
| `["plus", 1, 2]` | `1 + 2` |
| `["abs", x]` | `ABS(x)` |
| `["round", 2, x]` | `ROUND(x, 2)` |
| `["ceil", x]` | `CEIL(x)` |
| `["floor", x]` | `FLOOR(x)` |
| `["sqrt", x]` | `SQRT(x)` |
| `["pow", x, 2]` | `POWER(x, 2)` |

**Gaps:** None. All SQL pass-through.

---

## 11. Null Handling

| DTL | pg_weave |
|---|---|
| `["coalesce", a, b]` | `COALESCE(a, b)` |
| `["if-null", val, default]` | `COALESCE(val, default)` |
| `["is-null", x]` | `x IS NULL` |
| `["is-not-null", x]` | `x IS NOT NULL` |

**Gaps:** None.

---

## 12. Boolean Logic

| DTL | pg_weave |
|---|---|
| `["and", a, b]` | `a AND b` |
| `["or", a, b]` | `a OR b` |
| `["not", a]` | `NOT a` |
| `["all", func, list]` | SQL quantified predicate or subquery |
| `["any", func, list]` | SQL quantified predicate or subquery |

**Gaps:** None for basic logic. Quantified predicates (`all`/`any` over lists) require SQL subquery with `unnest`.

---

## Summary of Gaps

### No gap (SQL pass-through covers it)
Transforms: `add`, `add-if`, `copy` (basic), `rename`, `remove`, `filter`/`discard`, `default`, `case`/`if`, `comment`, `create`, `create-child`
Expressions: comparisons, math, strings, null handling, boolean logic, conditionals, date/time

### Ergonomic gaps (works via SQL but verbose)
- Array aggregates: `sum`, `max`, `min` over arrays require `unnest` subquery
- Array utilities: `distinct`, `reversed`, `enumerate`, `slice` require verbose SQL
- Set operations: `union`, `intersection`, `difference` need `unnest` + set ops
- Element access: `first`, `last`, `nth` differ between JSONB and typed arrays

### Proposed DSL helpers (ADR 018)
- `IS EMPTY` / `IS NOT EMPTY`
- `FIRST OF` / `LAST OF` / `NTH`
- Set-algebra helpers

### Structural gaps (by design)
- **Dynamic field names:** DTL `add` with computed property names — not possible in VIEW-based output
- **`merge`:** Spread all fields from a nested entity — no direct equivalent
- **`group-by`:** Inline grouping of array elements — requires SQL subquery
- **Side-effects:** `fail!`, `sleep!`, `trip!` — intentionally out of scope

### Proposed features (not yet accepted)
- Wildcard copy: `SET * EXCEPT (...)` (ADR 015)
- Recursive hops: `COLLECT ... FOLLOW` (ADR 005)
- Collection helpers: `IS EMPTY`, `FIRST OF` etc. (ADR 018)
