# Gap Analysis: DTL Transforms vs pg_weave

> **Date:** 2026-02-25  
> **Status:** Under discussion
> **Source:** Sesam Quick Reference → DTL Transform Functions

Scope of this document: only the DTL transform-function inventory from the quick reference (`comment`, `case`, `case-eq`, `if`, `create`, `create-child`, `discard`, `filter`, `add`, `add-if`, `copy`, `default`, `make-ni`, `merge`, `merge-union`, `remove`, `rename`, `fail!`, `sleep!`, `trip!`).

## Function-by-function matrix

| DTL transform function | pg_weave equivalent | Gap status | Recommendation |
|---|---|---|---|
| `comment` | SQL comments / DSL comments | No gap | Ignore at compiler level; preserve comments if parser supports them |
| `case`, `case-eq`, `if` | SQL `CASE WHEN ...` / `COALESCE` patterns | No gap | Keep as SQL pass-through |
| `create` | `FROM ... { SET ... }` output projection | No gap | Native model already |
| `create-child` | `COLLECT ... { ... }` for nested arrays | No gap | Native model already |
| `discard` | `WHERE NOT (...)` / trailing `WHERE` | No gap | Use `WHERE`; no special keyword needed |
| `filter` | Inline/trailing `WHERE` | No gap | Covered by ADR 014 |
| `add` | `SET field = expr` | No gap | Native model already |
| `add-if` | `SET field = expr WHEN cond` | No gap | Native model already (`SET ... WHEN ...`) |
| `copy` | `SET *` (+ `EXCEPT`) | Closed gap | Covered by ADR 015 |
| `default` | `COALESCE(field, default)` | No gap | SQL pass-through |
| `make-ni` | No direct equivalent (identifier namespace synthesis) | Out of scope (pg_weave) | Treat as pgmerge concern |
| `merge` | SQL joins + `LOOKUP`/`COLLECT` composition | Partial gap | Keep as relational composition; no merge primitive yet |
| `merge-union` | SQL union + dedupe rules | Partial gap | Prefer explicit SQL VIEW staging |
| `remove` | Omit field from `SET`; or `SET * EXCEPT (...)` | No gap | Native model via projection |
| `rename` | `SET new_name = old_name` | No gap | Native model already |
| `fail!` | No safe VIEW equivalent | Intentional gap | Keep out of scope for declarative VIEW transforms |
| `sleep!` | No VIEW equivalent | Intentional gap | Keep out of scope |
| `trip!` | No VIEW equivalent | Intentional gap | Keep out of scope |

## Summary

- Strong parity exists for declarative row-shaping transforms (`create`, `add`, `remove`, `rename`, `filter`, `copy`, conditionals).
- Main intentional differences are side-effect / imperative controls (`fail!`, `sleep!`, `trip!`) that conflict with VIEW semantics.
- Identity-namespace construction (`make-ni`) remains outside pg_weave scope and aligns better with pgmerge responsibilities.
- `merge` / `merge-union` are available through SQL staging and relational composition, but remain candidates for future ergonomic sugar only if repeated pain appears.

## Follow-up candidates

- Optional merge helpers only after validating that common join/union recipes are too verbose in real transforms.
