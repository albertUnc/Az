# Az Formal Grammar (EBNF)

This is a formal, implementation-ready grammar for Az, derived from every rule established across
`02-language-spec.md` and the rest of the spec set. Use this as the direct basis for the parser's
grammar rules / recursive-descent function structure. Notation: `::=` defines a rule, `|` is
alternation, `[...]` is optional, `{...}` is zero-or-more repetition, `'...'` is a literal token,
`(* ... *)` is a comment.

This grammar assumes the lexer has already stripped whitespace/newlines/comments entirely — as
established, they carry no syntactic meaning anywhere in Az, so the grammar below never references
them.

---

## 1. Program structure

```ebnf
program        ::= { usingStatement } { functionDecl }

usingStatement ::= 'using' importTarget { ',' importTarget } ';'

importTarget   ::= identifier '.' identifier   (* file.az or file.funcName, or a built-in library *)

functionDecl   ::= 'func' [ 'overload' ] identifier '(' [ paramList ] ')' ':' type block

paramList      ::= param { ',' param }

param          ::= type identifier [ '=' expression ]
```

**Enforced separately from the grammar (semantic rules, not parse-time syntax):**
- Exactly one function named `main` across the whole program, and it must be in the entry file.
- No file imported via `using` may contain a function named `main`.
- Every executable statement must be inside a `block` that belongs to some `functionDecl` — there
  is no rule in this grammar that allows a bare `statement` at the top level of `program`, which is
  what enforces "no code outside functions" structurally, not just as an afterthought check.

---

## 2. Types

```ebnf
type           ::= numericType
               | 'bool'
               | 'string'
               | 'char'
               | 'list' '<' type '>'
               | 'dict' '<' type ',' type '>'
               | 'var'
               | 'none'
               | 'file'

numericType    ::= [ signedness ] ( 'int' | 'double' | 'long' 'long' )

signedness     ::= 'signed' | 'unsigned'
```

Note `'long' 'long'` is two separate tokens (`long long` is written as two words in source) — treat
this as a fixed two-token sequence recognized together as one type, not a single lexer token.

---

## 3. Statements

```ebnf
block          ::= '{' { statement } '}'

statement      ::= varDecl
               | tempDeleteStmt
               | swapStmt
               | exprStmt
               | ifStmt
               | whileStmt
               | forStmt
               | timesStmt
               | switchStmt
               | breakStmt
               | continueStmt
               | returnStmt
               | skipStmt
               | skipHereStmt
               | block

varDecl        ::= { qualifier } type identifier '=' expression ';'

qualifier      ::= 'const' | 'constexpr' | 'global' | 'temp'

tempDeleteStmt ::= 'tempdelete' identifier ';'

swapStmt       ::= 'swap' identifier ',' identifier ';'

exprStmt       ::= expression ';'

returnStmt     ::= 'return' [ expression ] ';'

breakStmt      ::= 'break' ';'

continueStmt   ::= 'continue' ';'

skipStmt       ::= 'skip' identifier ';'

skipHereStmt   ::= 'skip' 'here' identifier ';'
```

**Legality of `skip`/`skip here` (semantic rule, not enforceable purely by this grammar):** every
`skipStmt`'s target `identifier` must name a `skipHereStmt` in the same function such that the
label's enclosing block is the jump's own block or an ancestor of it. See
`02-language-spec.md`'s Control Flow section for the full rule and examples, and
`01-compiler-architecture.md` for the implementation algorithm.

### Control flow

```ebnf
ifStmt         ::= 'if' expression block
                   { 'else' 'if' expression block }
                   [ 'else' block ]

whileStmt      ::= 'while' expression block

forStmt        ::= cStyleFor | rangeFor

cStyleFor      ::= 'for' forInit ';' forCondition ';' forIncrement block

forInit        ::= varDecl | exprStmt | 'none' ';'
                   (* varDecl/exprStmt already include their own trailing ';',
                      so in practice the parser consumes exactly one ';' total between
                      init and condition -- implementers should treat forInit as
                      "a statement, minus requiring a second semicolon", or equivalently
                      parse the three clauses split on bare ';' and parse each
                      independently as an optional statement/expression *)

forCondition   ::= expression | 'none'

forIncrement   ::= expression | 'none'

rangeFor       ::= 'for' ( 'var' | type ) ':' identifier block

timesStmt      ::= ( expression ) 'times' block
                   (* expression must evaluate to an int at compile time or runtime;
                      literal ints and int-typed variables both valid *)

switchStmt     ::= 'switch' expression '{' { caseClause } [ defaultClause ] '}'

caseClause     ::= 'case' expression block

defaultClause  ::= 'default' block
```

**Important parser implementation note on `cStyleFor`:** because there are no parentheses, the
parser must rely on `;` to separate the three clauses, and rely on the fact that `{` can never
legally begin an expression to know when the third clause (the increment) has ended and the loop
body has begun. Concretely: parse `forInit` up to and including its terminating `;`, then parse
`forCondition` as an expression (or the literal `none`) up to the next `;`, then parse
`forIncrement` as an expression (or `none`) **until encountering a token that cannot continue an
expression** — in practice this will always be the block's opening `{`.

---

## 4. Expressions (with precedence)

Standard precedence-climbing / recursive-descent-with-precedence structure. Listed from **lowest to
highest** precedence (i.e. parse in this order, outermost first):

```ebnf
expression     ::= assignment

assignment     ::= ternary [ assignOp assignment ]
                   (* right-associative *)

assignOp       ::= '=' | '+=' | '-=' | '*=' | '/=' | '%='

ternary        ::= logicalOr [ '?' expression ':' ternary ]

logicalOr      ::= logicalAnd { '||' logicalAnd }

logicalAnd     ::= equality { '&&' equality }

equality       ::= comparison { ( '==' | '!=' ) comparison }

comparison     ::= additive { ( '<' | '>' | '<=' | '>=' ) additive }

additive       ::= multiplicative { ( '+' | '-' ) multiplicative }

multiplicative ::= unary { ( '*' | '/' | '%' ) unary }

unary          ::= ( '!' | '-' | '++' | '--' ) unary
               | postfix

postfix        ::= primary { ( '++' | '--' | '.' identifier | '(' [ argList ] ')' | '[' expression ']' ) }
                   (* trailing ++/-- ; '.' for method/library calls like console.write,
                      file instance methods; '(' ... ')' for function calls;
                      '[' expression ']' -- list/dict element access, CONFIRMED, see section 5a *)

primary        ::= intLiteral
               | doubleLiteral
               | stringLiteral
               | charLiteral
               | 'true' | 'false'
               | identifier
               | listLiteral
               | dictLiteral
               | fstringCall
               | conversionCall
               | '(' expression ')'
                   (* this rule is what makes wrapping any condition/sub-expression in
                      parentheses always legal, anywhere -- see 02-language-spec.md
                      section 5's note on optional grouping parens *)

conversionCall ::= ( 'int' | 'double' | 'LL' | 'string' ) '(' expression ')'
                   (* built-in explicit conversion functions -- 02-language-spec.md
                      section 2 (numeric ones) and section 8 (string()) *)

listLiteral    ::= '[' [ expression { ',' expression } ] ']'
                   (* non-empty comma-separated form CONFIRMED alongside element access -- see
                      section 5a and 02-language-spec.md section 2a *)

dictLiteral    ::= '{' [ dictEntry { ',' dictEntry } ] '}'
                   (* same -- CONFIRMED *)

dictEntry      ::= expression ':' expression

fstringCall    ::= 'fstring' '(' stringLiteral ')'
                   (* the string literal's {...} segments are further parsed as
                      nested `expression` -- see section 5 below *)

argList        ::= expression { ',' expression }
```

**Notes:**
- `_i` is not a grammar-level special case — it parses as an ordinary `identifier` in `primary`.
  Its "magic" (auto-defined, auto-incrementing) is a semantic-analysis/codegen concern: the
  compiler must inject a declaration for `_i` at the start of any `while`/`for`/`times` loop body's
  scope, and increment it appropriately — see `01-compiler-architecture.md`.
- The ternary's `:` and a function declaration's return-type `:` are the same token but never
  ambiguous in practice: `ternary`'s `:` only appears while parsing an `expression`, and a function
  return type's `:` only appears immediately after a `functionDecl`'s closing `)`. These are
  disjoint parser contexts, so no lookahead trickery is required.

---

## 5. `fstring(...)` interpolation sub-grammar

The string literal passed to `fstring(...)` is not just an opaque string — it must be scanned for
`{...}` segments (with the universal `\` escape making `\{`/`\}` literal, not interpolation
delimiters). Recommended approach: after the lexer produces the raw string literal token, run a
**second, separate scan** over its contents (this can happen at parse time, treating it as a
mini-language nested inside the outer grammar):

```ebnf
fstringBody    ::= { fstringSegment }

fstringSegment ::= literalText | '{' expression '}'
```

Where `literalText` is any run of characters not containing an unescaped `{`, with `\{`, `\}`, and
all other `\`-escapes already resolved per the universal escape rule
(`02-language-spec.md` section 8). Each `{...}` segment's contents are parsed as a full `expression`
using the same expression grammar as section 4 above — meaning `fstring("{a + b}")` is legal and
`a + b` is parsed exactly as it would be anywhere else.

---

## 5a. `list`/`dict` element access — CONFIRMED, Python-style bracket syntax

**Resolved** (previously flagged here as unconfirmed — now settled). Both `list<T>` and
`dict<K,V>` support Python-style `[...]` access for both reading and writing, including negative
indices for `list`. Full behavioral spec, including the bounds-checking rules and the deliberate
read/write asymmetry for `dict`, is in `02-language-spec.md` section 2a — this section only adds
the formal grammar production:

```ebnf
(* extends the `postfix` rule already defined in section 4 above *)
indexAccess    ::= postfix '[' expression ']'
```

`indexAccess` is left-associative and composes with the rest of `postfix`'s trailers (so
`nums[0]`, `matrix[i][j]`-style chained indexing if ever needed, and `nums[i].someMethod()` all
parse consistently through the same mechanism already described for `.`/`(`/`++`/`--` in section 4).
On the left side of an `=` (or compound assignment), an `indexAccess` expression is a valid
assignment target — see `AssignExpr` and the new `IndexExpr` AST node in
`01-compiler-architecture.md` section 3a.

## 6. Lexical grammar (tokens)

```ebnf
identifier     ::= letter { letter | digit | '_' }
letter         ::= 'a'..'z' | 'A'..'Z'
digit          ::= '0'..'9'

intLiteral     ::= digit { digit }
                   (* no suffixes -- no 'LL', 'u', etc. A plain int literal is usable in any
                      numeric context (int/long long/unsigned) via implicit conversion --
                      see 02-language-spec.md's numeric literal note *)

doubleLiteral  ::= digit { digit } '.' digit { digit }
                   (* no scientific notation -- '1e10' is NOT valid Az syntax *)

stringLiteral  ::= '"' { stringChar } '"'
stringChar     ::= anyCharExcept('"', '\') | '\' anyChar
                   (* universal escape: backslash + any character *)

charLiteral    ::= "'" ( anyCharExcept("'", '\') | '\' anyChar ) "'"

lineComment    ::= '//' { anyCharExceptNewline }

blockComment   ::= '/*' { anyChar } '*/'
                   (* can span multiple lines; not nestable unless explicitly desired --
                      nesting was never discussed, recommend NON-nestable to match
                      standard C++ /* */ behavior *)
```

Whitespace (spaces, tabs, newlines) between tokens is consumed and discarded by the lexer, never
emitted as a token, never used to terminate anything.

---

## 7. Reserved words (parser must never allow these as identifiers)

See `07-keyword-reference.md` for the full categorized list. For grammar purposes, every entry in
that file is a distinct terminal token, not a generic `identifier` — the lexer should check
identifier-shaped text against the keyword table before emitting an `IDENTIFIER` token, emitting the
specific keyword token type instead when it matches.
