# Grammar Specification (EBNF)

This document defines the formal grammar for pg_weave.

## Scope

- Keywords are case-insensitive.
- Whitespace and newlines are insignificant except as token separators.
- `sql_expr` is delegated to PostgreSQL SQL expression parsing.

```ebnf
weave_definition     ::= from_block EOF ;

from_block           ::= "FROM" source_ref [inline_where] block ;
source_ref           ::= identifier ["AS" identifier] ;

inline_where         ::= "WHERE" sql_expr ;

block                ::= "{" statement_list [trailing_where] "}" ;
statement_list       ::= statement { "," statement } [","] ;
trailing_where       ::= "WHERE" sql_expr ;

statement            ::= set_stmt
                       | let_stmt
                       | flatmap_stmt ;

set_stmt             ::= "SET" set_target "=" value_expr ["WHEN" sql_expr]
                       | "SET" "*" [except_clause] ;
set_target           ::= identifier | quoted_identifier ;
except_clause        ::= "EXCEPT" "(" identifier { "," identifier } ")" ;
quoted_identifier    ::= '"' /[^"]+/ '"' ;
let_stmt             ::= "LET" identifier "=" value_expr ;

flatmap_stmt         ::= "FLATMAP" value_expr "AS" identifier
                         [inline_where]
                         [order_by_clause]
                         block ;

value_expr           ::= dsl_expr
                       | sql_expr ;

dsl_expr             ::= count_expr
                       | empty_expr
                       | first_last_expr
                       | lookup_expr
                       | collect_expr
                       | map_expr
                       | build_expr ;
count_expr           ::= "COUNT" "OF" value_expr ;
empty_expr           ::= "IS" ["NOT"] "EMPTY" value_expr ;
first_last_expr      ::= ("FIRST" | "LAST") "OF" value_expr ;

lookup_expr          ::= "LOOKUP" source_ref "ON" sql_expr ;

collect_expr         ::= "COLLECT"
                         source_ref
                         "ON" sql_expr
                         [inline_where]
                         ["INTO" type_ref "[" "]"]
                         block
                         [order_by_clause]
                       | "COLLECT"
                         source_ref
                         "FOLLOW" identifier
                         ["DEPTH" integer]
                         ["EXCLUDING" "ROOT"]
                         [inline_where]
                         ["INTO" type_ref "[" "]"]
                         block
                         [order_by_clause] ;

map_expr             ::= "MAP" value_expr "AS" identifier
                         [inline_where]
                         ["INTO" type_ref "[" "]"]
                         ( "->" sql_expr | block [order_by_clause] ) ;

build_expr           ::= "BUILD" ["INTO" type_ref] block ;

traversal_helper     ::= "DEPTH" "(" ")" | "PATH" "(" ")" | "CHILDREN" "(" ")" ;

order_by_clause      ::= "ORDER" "BY" order_item { "," order_item } ;
order_item           ::= sql_expr ["ASC" | "DESC"] ;

type_ref             ::= pg_type
                       | "jsonb"
                       | "[" type_ref "]" ;

identifier           ::= /[A-Za-z_][A-Za-z0-9_]*/ ;
integer              ::= /[0-9]+/ ;
pg_type              ::= identifier { identifier } ;
sql_expr             ::= <PostgreSQL SQL expression> ;
```

## Interpretation notes

- `SET` and `LET` statements can be interleaved within a block.
- Each `SET` field name must be unique within a block. Use `CASE ... WHEN ... END` for conditional values.
- A `LET` binding is available from its declaration onward in the same block; use-before-declaration is invalid.
- `WHERE` is valid in two places:
  - Inline (`FROM ... WHERE ... {}` / `COLLECT ... WHERE ... {}`)
  - Trailing (`{ ... WHERE ... }`) for filtering after computed `LET`/`SET` values
- `MAP` supports both shorthand (`-> sql_expr`) and block form (`{ SET ... }`).
- `COLLECT` in the stable grammar uses `ON ...` join semantics for related-row collection.
- `COLLECT ... FOLLOW field DEPTH N` expresses recursive graph walks compiled to `WITH RECURSIVE`. `DEPTH()`, `PATH()`, and `CHILDREN()` are valid only inside `FOLLOW` blocks.
- `INTO type[]` on `COLLECT`/`MAP` targets a PG composite type for typed array output. Without `INTO`, nested outputs default to JSONB.
- `BUILD { ... }` constructs a single nested object (JSONB by default, or typed composite with `INTO`).
- `SET *` copies all source columns at compile time. `SET * EXCEPT (...)` excludes listed columns. Explicit `SET` always wins regardless of position. `SET *` is valid at the top-level `FROM` block only.
- `COUNT OF` returns an integer — the length of the array produced by its operand.
- `IS EMPTY` / `IS NOT EMPTY` return booleans for array emptiness. `FIRST OF` / `LAST OF` return the first/last element of an array (`null` if empty).
- **Visibility rules:** each block sees its own input alias/columns plus `LET` bindings inherited from outer blocks. Outer table aliases are not directly visible in nested blocks — pass values down via `LET`. `LET o = o` (alias rebinding) is a compile-time error.
