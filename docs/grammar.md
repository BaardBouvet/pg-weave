# Grammar Specification (EBNF)

This document defines the formal grammar for pg_weave.

## Scope

- Keywords are case-insensitive.
- Whitespace and newlines are insignificant except as token separators.
- `sql_expr` is delegated to PostgreSQL SQL expression parsing.

```ebnf
weave_definition     ::= from_block EOF ;

from_block           ::= "FROM" source_ref [with_clause] [inline_where] block ;
source_ref           ::= identifier ["AS" identifier] ;

with_clause          ::= "WITH" with_entry { "," with_entry } ;
with_entry           ::= identifier json_shape ;

inline_where         ::= "WHERE" sql_expr ;

block                ::= "{" statement_list [trailing_where] "}" ;
statement_list       ::= statement { "," statement } [","] ;
trailing_where       ::= "WHERE" sql_expr ;

statement            ::= set_stmt
                       | let_stmt
                       | flatmap_stmt ;

set_stmt             ::= "SET" identifier "=" value_expr ["WHEN" sql_expr] ;
let_stmt             ::= "LET" identifier "=" value_expr ;

flatmap_stmt         ::= "FLATMAP" value_expr "AS" identifier
                         [inline_where]
                         [order_by_clause]
                         block ;

value_expr           ::= dsl_expr
                       | sql_expr ;

dsl_expr             ::= count_expr
                       | lookup_expr
                       | collect_expr
                       | map_expr ;
count_expr           ::= "COUNT" "(" value_expr ")" ;

lookup_expr          ::= "LOOKUP" source_ref "ON" sql_expr ;

collect_expr         ::= "COLLECT"
                         source_ref
                         "ON" sql_expr
                         [inline_where]
                         [order_by_clause]
                         block
                         [order_by_clause]
                         [limit_clause] ;

map_expr             ::= "MAP" value_expr "AS" identifier
                         [inline_where]
                         [order_by_clause]
                         ( "->" sql_expr | block [order_by_clause] ) ;

order_by_clause      ::= "ORDER" "BY" order_item { "," order_item } ;
order_item           ::= sql_expr ["ASC" | "DESC"] ;

limit_clause         ::= "LIMIT" integer ;

json_shape           ::= "{" json_field { "," json_field } [","] "}" ;
json_field           ::= identifier type_ref ;
type_ref             ::= pg_type
                       | "jsonb"
                       | "[" type_ref "]"
                       | json_shape ;

identifier           ::= /[A-Za-z_][A-Za-z0-9_]*/ ;
integer              ::= /[0-9]+/ ;
pg_type              ::= identifier { identifier } ;
sql_expr             ::= <PostgreSQL SQL expression> ;
```

## Interpretation notes

- `WHERE` is valid in two places:
  - Inline (`FROM ... WHERE ... {}` / `COLLECT ... WHERE ... {}`)
  - Trailing (`{ ... WHERE ... }`) for filtering after computed `LET`/`SET` values
- `MAP` supports both shorthand (`-> sql_expr`) and block form (`{ SET ... }`).
- `COLLECT` in the stable grammar uses `ON ...` join semantics for related-row collection.
- `COUNT(array_expr)` is a polymorphic language helper that lowers to `jsonb_array_length(...)` for JSONB arrays and `cardinality(...)` for typed PG arrays.
- `COUNT(array_expr)` is a DSL helper form (array length), distinct from SQL aggregate `COUNT(...)` over rows.
