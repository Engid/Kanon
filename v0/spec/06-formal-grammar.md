# 6. Formal Grammar

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

## 6.0 Notation

This grammar uses EBNF notation. `{ x }` means zero or more repetitions of x. `[ x ]` means x is optional. `|` separates alternatives. Quoted strings are terminal tokens.

## 6.1 EBNF

```ebnf
program        = { declaration } ;

declaration    = fun_decl
               | type_decl
               | enum_decl
               | let_decl
               | statement ;

(* --- Declarations --- *)

fun_decl       = "fun" IDENT [ type_params ] "(" [ param_list ] ")" [ ":" type ]
                 ( "=" expression | block ) ;

type_decl      = "type" IDENT [ type_params ]
                 ( "=" type                              (* alias *)
                 | "{" field_def { "," field_def } [ "," ] "}"   (* product type *)
                 ) ;

field_def      = IDENT ":" type ;

enum_decl      = "enum" IDENT [ type_params ] "{"
                 variant { "," variant } [ "," ]
                 "}" ;

variant        = IDENT [ "(" param_list ")" ] ;

let_decl       = ( "let" | "mut" ) IDENT [ ":" type ] "=" expression ;

(* --- Statements --- *)

statement      = expression
               | for_loop
               | while_loop
               | return_stmt ;

for_loop       = "for" IDENT "in" expression block ;
while_loop     = "while" expression block ;
return_stmt    = "return" [ expression ] ;

block          = "{" { declaration } "}" ;

(* --- Expressions (ordered by ascending precedence in Pratt parser) --- *)

expression     = assignment ;

assignment     = IDENT "=" expression
               | pipe_expr ;

pipe_expr      = or_expr { "|>" or_expr } ;

or_expr        = and_expr { "or" and_expr } ;
and_expr       = equality { "and" equality } ;
equality       = comparison { ( "==" | "!=" ) comparison } ;
comparison     = addition { ( "<" | "<=" | ">" | ">=" ) addition } ;
addition       = multiplication { ( "+" | "-" ) multiplication } ;
multiplication = unary { ( "*" | "/" | "%" ) unary } ;
unary          = ( "-" | "not" ) unary | postfix ;
postfix        = primary { call | dot | imaginary } ;

call           = "(" [ arg_list ] ")" ;   (* also used for indexing: list(0) *)
dot            = "." IDENT [ call ] ;
imaginary      = "i" ;

primary        = NUMBER
               | RATIO                      (* integer "/" integer *)
               | EDO                        (* integer "\" integer *)
               | FLOAT
               | STRING
               | "true" | "false" | "nil"
               | IDENT
               | lambda
               | gen_expr
               | if_expr
               | match_expr
               | list_literal
               | type_constructor
               | "(" expression ")" ;

(* --- Lambda --- *)

lambda         = "(" [ param_list ] ")" "=>" ( expression | block )
               | IDENT "=>" ( expression | block ) ;    (* single param, no parens *)

(* --- Generator --- *)

gen_expr       = "gen" IDENT "in" expression [ "where" expression ]
                 ( expression | block ) ;

(* --- If (expression) --- *)

if_expr        = "if" expression block [ "else" ( if_expr | block ) ] ;

(* --- Match --- *)

match_expr     = "match" expression "{"
                 match_arm { "," match_arm } [ "," ]
                 "}" ;
match_arm      = pattern [ "when" expression ] "=>" ( expression | block ) ;

pattern        = IDENT "(" [ pattern_list ] ")"   (* constructor *)
               | IDENT                             (* binding *)
               | "_"                               (* wildcard *)
               | literal ;                         (* literal *)

(* --- Type constructor / struct literal --- *)

type_constructor = IDENT "{" field_init { "," field_init } [ "," ] "}" ;
field_init     = IDENT ":" expression
               | "..." expression ;               (* spread for .with()-like copy *)

(* --- Types --- *)

type           = IDENT [ "[" type_list "]" ]       (* simple or generic *)
               | "(" [ type_list ] ")" "=>" type   (* function type *)
               | type "?" ;                        (* optional *)

type_params    = "[" IDENT { "," IDENT } "]" ;

(* --- Misc --- *)

param_list     = param { "," param } ;
param          = IDENT [ ":" type ] [ "=" expression ] ;
arg_list       = arg { "," arg } ;
arg            = [ IDENT ":" ] expression ;        (* positional or named *)
list_literal   = "[" [ expression { "," expression } [ "," ] ] "]" ;
pattern_list   = pattern { "," pattern } ;
type_list      = type { "," type } ;
```

## 6.2 Pratt Parser Precedence Table

The expression parser uses a Pratt (top-down operator precedence) parser. Each token has a prefix and/or infix handler with an associated binding power.

| Precedence | Operators              | Associativity | Notes                              |
|------------|------------------------|---------------|------------------------------------|
| 1          | `=`                    | Right         | Assignment                         |
| 2          | `\|>`                  | Left          | Pipe                               |
| 3          | `or`                   | Left          | Logical OR                         |
| 4          | `and`                  | Left          | Logical AND                        |
| 5          | `==` `!=`              | Left          | Equality                           |
| 6          | `<` `<=` `>` `>=`     | Left          | Comparison                         |
| 7          | `+` `-`               | Left          | Addition                           |
| 8          | `*` `/` `%`           | Left          | Multiplication, rational literal   |
| 9          | `-` `not`             | Right (prefix)| Unary                              |
| 10         | `()` `.`              | Left          | Call/index, member access           |
| 11         | `i`                   | Left (postfix)| Imaginary suffix                   |
| 12         | `\`                   | Left          | EDO literal (integer \ integer)    |

## 6.3 Parsing Notes

### Rational Literals (`/`)

`/` between two integer literals is parsed as a `Ratio` literal, not a division operation. The Pratt parser checks the types of left and right operands after parsing a multiplication-precedence expression: if both are integer literals, emit a `RatioLiteral` AST node; otherwise emit a `BinaryDiv` node.

**Alternative approach:** Handle this in the scanner — when the scanner sees `digit+ / digit+` with no intervening whitespace, emit a single `RATIO` token. This means `3/2` (no spaces) is a rational, while `3 / 2` (with spaces) would also be rational since both operands are integers. The parser approach is more flexible.

### EDO Literals (`\`)

`\` between two integer literals is parsed as an `Edo` literal. It is a parse error if either operand is not an integer literal. The `\` token has the highest precedence in the Pratt table.

### Imaginary Suffix (`i`)

`i` as a postfix on a numeric expression produces an imaginary component. The Pratt parser registers `i` as a postfix parselet that wraps the preceding expression in an `ImaginaryLiteral` node. It only applies when the preceding expression is a numeric literal or numeric expression.

### Unit Suffixes (`hz`, `c`)

`hz` and `c` after float literals are handled as UFCS calls to conversion functions. When the parser sees `440.0hz`, it parses `440.0` as a float literal, then `.hz` as a member access. The UFCS system resolves this to `hz(440.0)`.

### Square Brackets (`[]`)

`[]` is **only** for type parameterization and list literals. It never appears in value-position as an indexing operator. Indexing uses parentheses: `list(0)`, `scale(2)`. This eliminates all bracket ambiguity.

- `Seq[Note]` — type parameterization (always in a type position)
- `[1, 2, 3]` — list literal (bare, no preceding identifier)
- `scale(0)` — indexing (parentheses, interpreted as a call on an indexable value)

### Parenthesis Indexing vs. Function Calls

From the parser's perspective, `f(x)` and `list(0)` are both `CallExpr` nodes. The interpreter disambiguates at runtime: if the callee is a function/closure, it's a call; if it's a list/string, it's an index. See §8.2 for implementation details.

### Type Constructor vs. Block (`{}`)

When the parser encounters `Ident { ... }`, it must determine whether this is a type constructor (`Note { pitch: 7\12, ... }`) or a block. The rule: if the content starts with `Ident ":"`, it's a type constructor. Otherwise, it's a block.
