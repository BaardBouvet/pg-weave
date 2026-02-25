# Gap Analysis: DTL Expression Functions vs pg_weave

> **Date:** 2026-02-25  
> **Status:** Under discussion
> **Source:** Sesam Quick Reference → DTL Expression Functions

Scope of this document: expression-function inventory only. Transform functions are covered separately in `DTL_TRANSFORMS.md`.

## Category matrix

| DTL expression category | Functions (quick-reference inventory) | pg_weave baseline | Gap status | Recommendation |
|---|---|---|---|---|
| Boolean logic | `all`, `and`, `any`, `or`, `not` | SQL boolean expressions (`AND`, `OR`, `NOT`, quantified predicates) | Minor gap | Keep pass-through; consider helper sugar for `ANY OF` / `ALL OF` later |
| Booleans | `boolean`, `is-boolean` | SQL casts + `jsonb_typeof` checks when needed | Minor gap | Keep pass-through; add typed helper only if validation pain appears |
| Bytes | `base64-decode`, `base64-encode`, `bytes`, `is-bytes` | PG has `encode`/`decode`, `bytea` | Partial gap | Defer dedicated helpers; use SQL functions directly |
| Comparisons | `eq`, `gt`, `gte`, `lt`, `lte`, `neq` | Native SQL operators | No gap | Pass-through |
| Conditionals | `case`, `case-eq`, `if` | SQL `CASE WHEN` | No gap | Pass-through |
| Date and time | `datetime`, `datetime-diff`, `datetime-format`, `datetime-parse`, `datetime-plus`, `datetime-shift`, `is-datetime`, `now` | PG date/time functions and operators | Minor gap | Keep pass-through; avoid hiding non-deterministic behavior (`now`) |
| Dictionaries | `apply`, `apply-hops`, `apply-ns dict`, `has-key`, `is-dict`, `items`, `key-values`, `keys`, `values`, `path`, `strip-ns` | PG JSON/JSONB object functions/operators | Partial gap | Keep pass-through; consider selective helper aliases (`HAS KEY`, `KEYS OF`) later |
| Encryption | `decrypt`, `decrypt-pgp`, `decrypt-pki`, `encrypt`, `encrypt-pgp`, `encrypt-pki` | Extension-dependent SQL functions | Intentional gap | Out of core DSL scope; use DB extensions/functions directly |
| Hops | `hops`, `lookup-entity` | `LOOKUP` (fixed depth) and joins | Partial gap | Keep fixed-depth via `LOOKUP`; recursive hops remain proposed (ADR 005) |
| JSON | `json`, `json-parse`, `json-transit`, `json-transit-parse` | PG JSON constructors/parsers | Minor gap | Pass-through; only add DSL sugar if repeated verbosity emerges |
| Lists | `combine`, `count`, `distinct`, `enumerate`, `filter`, `first`, `flatten`, `group`, `group-by`, `in`, `insert`, `is-empty`, `is-list`, `is-not-empty`, `last`, `list`, `map`, `map-dict`, `map-values`, `max`, `min`, `nth`, `range`, `reversed`, `slice`, `sorted`, `sorted-descending`, `sum` | `MAP`, `FLATMAP`, `COLLECT`, SQL array/JSONB ops; `COUNT OF` accepted | Partial gap | Prioritize storage-neutral helpers from ADR 018 (`IS EMPTY`, `FIRST OF`, `LAST OF`, `NTH`, set-like ops) |
| Math | `-`, `/`, `*`, `%`, `^`, `+`, `abs`, `ceil`, `cos`, `divide`, `floor`, `minus`, `mod`, `multiply`, `plus`, `pow`, `round`, `sin`, `sqrt`, `tan` | Native SQL math operators/functions | No gap | Pass-through |
| Misc | `completeness`, `hash128`, `is-changed`, `literal`, `tuples` | Mixed; partly SQL-native, partly pipeline-state semantics | Partial gap | Keep core DSL minimal; evaluate only clear relational use-cases |
| Namespaced identifiers | `is-ni`, `ni`, `ni-id`, `ni-ns` | No direct equivalent | Out of scope (pg_weave) | Treat as identity/merge domain (pgmerge) |
| Nulls | `coalesce`, `coalesce-args`, `if-null`, `is-not-null`, `is-null` | SQL null semantics (`COALESCE`, `IS NULL`) | No gap | Pass-through |
| Numbers | `decimal`, `float`, `hex`, `integer`, `is-decimal`, `is-float`, `is-integer` | SQL casts + type checks | Minor gap | Pass-through; helper aliases optional |
| Phonenumbers | `phonenumber-parse`, `phonenumber-format` | Extension/UDF dependent | Intentional gap | Out of core DSL scope |
| Sets | `difference`, `intersection`, `intersects`, `union` | SQL/array/set operations with custom lowering | Partial gap | Track in ADR 018 as likely v1.2 helper candidates |
| Strings | `concat`, `is-string`, `join`, `length`, `ljust`, `lower`, `lstrip`, `matches`, `replace`, `rjust`, `rstrip`, `split`, `string`, `strip`, `substring`, `upper` | Native SQL string functions/operators | No gap | Pass-through |
| URIs | `is-uri`, `uri`, `url-quote`, `url-unquote` | No dedicated core primitives | Partial gap | Defer to SQL/UDFs; avoid protocol-specific core helpers |
| UUIDs | `is-uuid`, `uuid` | Native `uuid` type + extension functions | Minor gap | Pass-through; avoid masking volatile generation semantics |

## Summary

- Most categories map cleanly to SQL pass-through, which is a strength of pg_weave’s model.
- The largest practical ergonomic gap remains collection semantics (lists/sets), where storage-neutral helper syntax adds value.
- Domain-specific function families (encryption, phone, URI normalization, NI identity synthesis) should remain out of core DSL scope.
- Recursive/graph traversal behavior remains gated by ADR status and should not be treated as stable language surface.

## Priority candidates

1. Continue collection-helper track (ADR 018): `IS EMPTY`, `IS NOT EMPTY`, `FIRST OF`, `LAST OF`, `NTH`, plus set-algebra helpers.
2. Keep all scalar/math/string/date logic as SQL pass-through unless concrete usability pain is observed.
3. Avoid adding wrappers over extension-specific capabilities (encryption/phone/URI) to preserve portability.
