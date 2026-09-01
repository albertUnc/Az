# Az Compiler — Architecture & Implementation Guide

This file describes how to build the Az compiler itself: pipeline stages, project structure, and
specific implementation guidance for the trickier language features. Where the design conversation
didn't explicitly settle an implementation detail (as opposed to a *language* rule, which is always
in `02-language-spec.md`), this file gives a concrete, clearly-labeled **recommendation** so the
implementer isn't stuck guessing — but these recommendations are open to being overridden.

## 0. CONFIRMED: the compiler itself is written in C++

The Az compiler (the transpiler that reads `.az` files and emits `.cpp`) is written in **C++**,
compiled into its own standalone executable named **`az-transpiler`** (`az-transpiler.exe` on Windows).
Run from `cmd`/PowerShell (or any shell) either directly if it's in the current folder, or from
anywhere if it's been added to `PATH`. This is a genuine, deliberate decision by the designer (not
the earlier draft Python recommendation, which is superseded and kept below only for historical
context).

**Confirmed invocation model (v1, before the two tools are unified — see below):**

```
az-transpiler HelloWorld.az
```

This produces **one single generated `.cpp` file** (default name `HelloWorld.cpp`) containing
**everything needed to build the program** — not just `HelloWorld.az`'s own functions, but every
function pulled in transitively through its `using` statements (and their own `using` statements,
and so on), all flattened into that one file. See section 7 below ("Combined single-file output")
for exactly how this works, including how it avoids duplicating a file that's imported through more
than one path (a "diamond" import). The user then separately invokes `g++` (or `clang++`) on that
one generated file to produce the final executable. **Planned follow-up (not yet built, but the
intended end state):** unify this into a single command that both transpiles *and* invokes the C++
compiler automatically, so the user only ever runs one command (e.g. `az HelloWorld.az`) and gets a
finished binary directly, without a manual intermediate `g++` step. Build the two-step version
first, then wrap it.

All code snippets and structural suggestions elsewhere in this document should be read as C++
(e.g. the lexer/parser/codegen modules described in section 9's project structure would be `.cpp`/
`.hpp` files, not `.py`).

<details>
<summary>Superseded: earlier Python recommendation (kept for context only, no longer applies)</summary>

Previously, before this was confirmed, this section recommended Python for faster iteration during
development, since the compiler is a dev tool and Python's speed doesn't affect the speed of
programs *compiled by* Az. The designer has since confirmed C++ instead, favoring a single
dependency-free native binary and consistency with the rest of the toolchain (`g++`/`clang++` are
already required to be present regardless, so building the transpiler itself in C++ adds no new
dependency for the end user). This note is kept only so nothing in project history implies Python
was ever actually decided on.
</details>

---

## 1. Pipeline overview

```
 .az source file(s)
        │
        ▼
 ┌─────────────┐
 │   Lexer     │  →  stream of tokens
 └─────────────┘
        │
        ▼
 ┌─────────────┐
 │   Parser    │  →  Abstract Syntax Tree (AST), one per file
 └─────────────┘
        │
        ▼
 ┌───────────────────────────┐
 │  Import Resolution Pass   │  →  discovers all `using` targets, recursively parses
 │                           │     those files too, builds one combined program view
 └───────────────────────────┘
        │
        ▼
 ┌───────────────────────────┐
 │  Symbol Table Pre-Pass    │  →  scans ALL functions across ALL files first, before
 │                           │     any code generation. Enables forward references
 │                           │     (call a function before its textual definition).
 │                           │     Also where name-collision checking happens.
 └───────────────────────────┘
        │
        ▼
 ┌───────────────────────────┐
 │  Semantic Analysis /      │  →  type checking, verifying exactly one main(),
 │  Validation                │     verifying no top-level code outside functions,
 │                           │     verifying temp/global rules, etc.
 └───────────────────────────┘
        │
        ▼
 ┌───────────────────────────┐
 │  C++ Code Generation      │  →  emits one .cpp (and/or .h) per Az file, using
 │                           │     the Az runtime support library for built-ins
 └───────────────────────────┘
        │
        ▼
 ┌───────────────────────────┐
 │  Invoke gcc/clang         │  →  compiles + links generated C++ into final binary
 └───────────────────────────┘
        │
        ▼
   final executable
```

Each stage is described in more detail below where it has Az-specific subtleties beyond what a
standard compiler course would cover.

---

## 2. Lexer

Standard tokenizer. Notable Az-specific points:

- **Whitespace and newlines carry zero meaning** — never emit a token for them, never let them
  terminate anything. The whole language must parse identically regardless of formatting.
- **`;` is the only statement terminator.** No implicit statement-end inference from newlines.
- **Comments:** `//` to end of line, `/* ... */` can span multiple lines. Strip during lexing,
  never becomes a token.
- **String/char literals are fixed-delimiter**, not interchangeable: `"..."` is always `string`,
  `'...'` is always `char` (must contain exactly one character). This is a lexer-level distinction,
  not something resolved later.
- **The universal escape rule**: inside string literals, `\` followed by *any* character means "the
  literal version of that character" — this is one general rule (`\` + X → literal X), not a fixed
  whitelist like C++'s `\n`/`\t`/etc. Implement this as: whenever the lexer sees `\` inside a string
  literal, consume the next character as-is (map the common ones like `\n`→newline, `\t`→tab as
  usual, but the *mechanism* should generalize — `\{`, `\}`, `\"`, `\\` all need to work the same way
  so that literal braces/quotes/backslashes can appear inside both plain strings and `fstring()`
  calls without being misinterpreted as syntax).
- **Context-sensitive `:` is fine** — `:` appears in two different grammatical contexts (function
  return-type declarations, and the ternary operator). This does not require special lexer handling;
  the *parser* naturally disambiguates because each context expects a `:` in a completely different
  grammatical position (right after a function's closing `)`, vs. inside an expression after `?`).
  Treat `:` as a single token type at the lexer level.

---

## 2a. Lexer: token types (concrete C++ sketch)

```cpp
enum class TokenType {
    // Literals
    INT_LITERAL, DOUBLE_LITERAL, STRING_LITERAL, CHAR_LITERAL,
    TRUE, FALSE,
    IDENTIFIER,

    // Types
    INT, DOUBLE, LONG, BOOL, STRING_TYPE, CHAR_TYPE, LIST, DICT, VAR, NONE, FILE_TYPE,
    SIGNED, UNSIGNED,

    // Qualifiers
    CONST, CONSTEXPR, GLOBAL, TEMP, TEMPDELETE, SWAP,

    // Control flow
    IF, ELSE, WHILE, FOR, TIMES, BREAK, CONTINUE, SWITCH, CASE, DEFAULT, SKIP, HERE,

    // Functions / structure
    FUNC, RETURN, USING, OVERLOAD,

    // Built-in-adjacent (recognized identifiers, not always full keywords -- see note below)
    FSTRING, LL,

    // Operators
    PLUS, MINUS, STAR, SLASH, PERCENT,
    EQ_EQ, NOT_EQ, LESS, GREATER, LESS_EQ, GREATER_EQ,
    AND_AND, OR_OR, BANG,
    EQ, PLUS_EQ, MINUS_EQ, STAR_EQ, SLASH_EQ, PERCENT_EQ,
    PLUS_PLUS, MINUS_MINUS,
    QUESTION, COLON,

    // Punctuation
    LPAREN, RPAREN, LBRACE, RBRACE, LBRACKET, RBRACKET,
    COMMA, DOT, SEMICOLON,

    END_OF_FILE
};

struct Token {
    TokenType type;
    std::string text;      // raw lexeme, e.g. "42", "myVar", "\"hello\""
    int line;               // 1-based
    int col;                 // 1-based
    std::string sourceFile;  // which .az file this token came from (for diagnostics)
};
```

**Note on `console`/`math`/`random`/`time`/`file`/`logs` as identifiers vs. keywords:** these are
*not* reserved words in the lexer's sense (a user could theoretically not know they're special) --
they're ordinary `IDENTIFIER` tokens that the **parser/semantic analysis** stages recognize specially
when they appear as a `using` target or as the left side of a `.` in a call expression. Don't add
them to `TokenType` as their own keyword entries; keep them as regular identifiers and special-case
them structurally later in the pipeline, per `02-language-spec.md` section 7's note that built-ins
go through the same `using` mechanism as user code, syntactically.

**Note on `FSTRING` and `LL` getting dedicated tokens, unlike the library names above:** `fstring`
needs its own token because its argument requires special sub-parsing (the `{...}` interpolation
segments, section 5 of `13-grammar.md`), not just ordinary function-call parsing. `LL` needs its own
token because, unlike `int`/`double`/`string` (which reuse their existing type-keyword tokens for
double duty as conversion-function names — see `13-grammar.md`'s `conversionCall` rule and
`02-language-spec.md` section 2), there is no existing `LL` type token to reuse (`long long` is
lexed as two separate `LONG LONG` tokens, not a single `LL` token) — so `LL` needs to be recognized
as its own reserved word specifically for this purpose.

**Note on `Info`/`Warning`/`Error` (the `logs` library's severity literals):** same treatment --
these are ordinary identifiers, recognized specially only in the second-argument position of a
`logs.write(...)` call, per `11-library-logs.md`'s note that they are not a general Az enum feature.

---

## 3. Parser

Recursive-descent parsing is the natural fit here (this is a common, well-understood technique and
Az's grammar doesn't require anything more exotic like operator-precedence parsing beyond the usual
expression-precedence handling, which recursive descent handles fine with precedence-climbing for
the arithmetic/logical/comparison operators).

Build one AST per `.az` file. Suggested top-level AST node types:
- `FunctionDecl` (name, params w/ optional defaults, return type, body, source file)
- `UsingStatement` (either whole-file: `using foo.az;`, or specific: `using foo.bar;`, and their
  comma-separated combinations all normalize to a list of these)
- Statement nodes: `VarDecl`, `If`, `While`, `For` (both C-style and range-based), `Times`, `Switch`,
  `Break`, `Continue`, `Return`, `TempDelete`, `Swap`, expression-statements
- Expression nodes: standard binary/unary ops, function calls, `fstring(...)`, literals, identifiers,
  ternary, list/dict literals

**Important constraint to enforce at parse time (or immediately after, in semantic analysis — either
is fine, but it must be enforced somewhere):** no statement is allowed to exist outside of a function
body, at the top level of any file. This includes the entry file and every `using`-imported file. See
`02-language-spec.md` section on program structure for full details.

---

## 3a. AST node definitions (concrete C++ sketch)

A tagged-hierarchy approach (base class + `enum class NodeKind` + `dynamic_cast` or a manual `kind`
switch) is simpler to get working quickly than a full visitor pattern, and is a reasonable choice
for a first implementation:

```cpp
struct Type {
    enum class Kind { Int, Double, LongLong, Bool, String, Char, List, Dict, Var, None, File };
    Kind kind;
    bool isUnsigned = false;
    std::unique_ptr<Type> elementType;   // for list<T>
    std::unique_ptr<Type> keyType;       // for dict<K, V> (elementType doubles as V)
};

struct Expr { virtual ~Expr() = default; int line, col; std::string sourceFile; };
struct Stmt { virtual ~Stmt() = default; int line, col; std::string sourceFile; };

// --- Expressions ---
struct IntLiteralExpr : Expr { long long value; };
struct DoubleLiteralExpr : Expr { double value; };
struct StringLiteralExpr : Expr { std::string value; };
struct CharLiteralExpr : Expr { char value; };
struct BoolLiteralExpr : Expr { bool value; };
struct IdentifierExpr : Expr { std::string name; };
struct BinaryExpr : Expr { TokenType op; std::unique_ptr<Expr> left, right; };
struct UnaryExpr : Expr { TokenType op; std::unique_ptr<Expr> operand; bool isPrefix; };
struct TernaryExpr : Expr { std::unique_ptr<Expr> condition, ifTrue, ifFalse; };
struct AssignExpr : Expr { TokenType op; std::unique_ptr<Expr> target, value; };
struct CallExpr : Expr {
    std::unique_ptr<Expr> callee;                 // e.g. IdentifierExpr for a bare call
    std::string libraryPrefix;                     // e.g. "console" for console.write(...), empty if none
    std::vector<std::unique_ptr<Expr>> args;
};
struct MethodCallExpr : Expr {                      // e.g. myFile.write(...), myFile.getline()
    std::unique_ptr<Expr> instance;
    std::string methodName;
    std::vector<std::unique_ptr<Expr>> args;
};
struct FstringExpr : Expr {
    // pre-split into literal text and nested expression segments -- see 13-grammar.md section 5
    std::vector<std::variant<std::string, std::unique_ptr<Expr>>> segments;
};
struct ListLiteralExpr : Expr { std::vector<std::unique_ptr<Expr>> elements; };
struct DictLiteralExpr : Expr {
    std::vector<std::pair<std::unique_ptr<Expr>, std::unique_ptr<Expr>>> entries;
};
struct IndexExpr : Expr {   // nums[0], myDict["key"] -- CONFIRMED, see 02-language-spec.md 2a
    std::unique_ptr<Expr> container;   // the list/dict expression being indexed
    std::unique_ptr<Expr> index;        // the index/key expression inside [ ]
};

// --- Statements ---
struct VarDeclStmt : Stmt {
    bool isConst = false, isConstexpr = false, isGlobal = false, isTemp = false;
    Type type;
    std::string name;
    std::unique_ptr<Expr> initializer;
};
struct TempDeleteStmt : Stmt { std::string name; };
struct SwapStmt : Stmt { std::string nameA, nameB; };
struct ExprStmt : Stmt { std::unique_ptr<Expr> expr; };
struct ReturnStmt : Stmt { std::unique_ptr<Expr> value; /* null if bare `return;` */ };
struct BreakStmt : Stmt {};
struct ContinueStmt : Stmt {};
struct SkipStmt : Stmt { std::string targetName; };
struct SkipHereStmt : Stmt { std::string name; };
struct BlockStmt : Stmt { std::vector<std::unique_ptr<Stmt>> statements; };

struct IfStmt : Stmt {
    std::vector<std::pair<std::unique_ptr<Expr>, std::unique_ptr<BlockStmt>>> branches; // if + else-ifs
    std::unique_ptr<BlockStmt> elseBranch;   // null if no else
};
struct WhileStmt : Stmt { std::unique_ptr<Expr> condition; std::unique_ptr<BlockStmt> body; };
struct CStyleForStmt : Stmt {
    std::unique_ptr<Stmt> init;       // null if `none`
    std::unique_ptr<Expr> condition;  // null if `none`
    std::unique_ptr<Expr> increment;  // null if `none`
    std::unique_ptr<BlockStmt> body;
};
struct RangeForStmt : Stmt {
    Type elementType;         // may be Type::Kind::Var
    std::string listExprName; // the identifier of the list being iterated
    std::unique_ptr<BlockStmt> body;
};
struct TimesStmt : Stmt { std::unique_ptr<Expr> count; std::unique_ptr<BlockStmt> body; };
struct SwitchStmt : Stmt {
    std::unique_ptr<Expr> subject;
    std::vector<std::pair<std::unique_ptr<Expr>, std::unique_ptr<BlockStmt>>> cases;
    std::unique_ptr<BlockStmt> defaultCase;  // null if absent (though semantic analysis may
                                               // eventually require it -- not currently mandated)
};

// --- Top level ---
struct Param { Type type; std::string name; std::unique_ptr<Expr> defaultValue; /* null if none */ };
struct FunctionDecl {
    std::string name;
    bool isOverload = false;
    std::vector<Param> params;
    Type returnType;
    std::unique_ptr<BlockStmt> body;
    std::string sourceFile;
    int line, col;
};
struct UsingStatement {
    bool isWholeFile;             // true: `using foo.az;`  false: `using foo.bar;`
    std::string target;            // "foo" (file stem) either way
    std::string specificFunction;  // only set when !isWholeFile
    int line, col;
};
struct AzFile {
    std::string path;
    std::vector<UsingStatement> usingStatements;
    std::vector<FunctionDecl> functions;
};
```

---

## 3b. Parser: expression parsing sketch (precedence climbing)

Direct implementation of the precedence table in `13-grammar.md` section 4, from lowest to highest
binding:

```cpp
std::unique_ptr<Expr> Parser::parseExpression() { return parseAssignment(); }

std::unique_ptr<Expr> Parser::parseAssignment() {
    auto left = parseTernary();
    if (matchAny({TokenType::EQ, TokenType::PLUS_EQ, TokenType::MINUS_EQ,
                  TokenType::STAR_EQ, TokenType::SLASH_EQ, TokenType::PERCENT_EQ})) {
        TokenType op = previous().type;
        auto value = parseAssignment();   // right-associative: recurse into itself
        auto node = std::make_unique<AssignExpr>();
        node->op = op; node->target = std::move(left); node->value = std::move(value);
        return node;
    }
    return left;
}

std::unique_ptr<Expr> Parser::parseTernary() {
    auto condition = parseLogicalOr();
    if (match(TokenType::QUESTION)) {
        auto ifTrue = parseExpression();
        expect(TokenType::COLON, "expected ':' in ternary expression");
        auto ifFalse = parseTernary();   // right-associative
        auto node = std::make_unique<TernaryExpr>();
        node->condition = std::move(condition);
        node->ifTrue = std::move(ifTrue);
        node->ifFalse = std::move(ifFalse);
        return node;
    }
    return condition;
}

// Generic left-associative binary-operator level -- reused for logicalOr, logicalAnd,
// equality, comparison, additive, multiplicative by passing the right operator set and
// the next-tighter parse function:
std::unique_ptr<Expr> Parser::parseBinaryLevel(
    const std::vector<TokenType>& ops,
    std::unique_ptr<Expr> (Parser::*nextLevel)())
{
    auto left = (this->*nextLevel)();
    while (matchAny(ops)) {
        TokenType op = previous().type;
        auto right = (this->*nextLevel)();
        auto node = std::make_unique<BinaryExpr>();
        node->op = op; node->left = std::move(left); node->right = std::move(right);
        left = std::move(node);
    }
    return left;
}

std::unique_ptr<Expr> Parser::parseLogicalOr()   { return parseBinaryLevel({TokenType::OR_OR}, &Parser::parseLogicalAnd); }
std::unique_ptr<Expr> Parser::parseLogicalAnd()  { return parseBinaryLevel({TokenType::AND_AND}, &Parser::parseEquality); }
std::unique_ptr<Expr> Parser::parseEquality()    { return parseBinaryLevel({TokenType::EQ_EQ, TokenType::NOT_EQ}, &Parser::parseComparison); }
std::unique_ptr<Expr> Parser::parseComparison()  { return parseBinaryLevel({TokenType::LESS, TokenType::GREATER, TokenType::LESS_EQ, TokenType::GREATER_EQ}, &Parser::parseAdditive); }
std::unique_ptr<Expr> Parser::parseAdditive()    { return parseBinaryLevel({TokenType::PLUS, TokenType::MINUS}, &Parser::parseMultiplicative); }
std::unique_ptr<Expr> Parser::parseMultiplicative() { return parseBinaryLevel({TokenType::STAR, TokenType::SLASH, TokenType::PERCENT}, &Parser::parseUnary); }

std::unique_ptr<Expr> Parser::parseUnary() {
    if (matchAny({TokenType::BANG, TokenType::MINUS, TokenType::PLUS_PLUS, TokenType::MINUS_MINUS})) {
        TokenType op = previous().type;
        auto operand = parseUnary();   // right-recursive: handles `!!x`, `--x`, etc.
        auto node = std::make_unique<UnaryExpr>();
        node->op = op; node->operand = std::move(operand); node->isPrefix = true;
        return node;
    }
    return parsePostfix();
}

std::unique_ptr<Expr> Parser::parsePostfix() {
    auto expr = parsePrimary();
    while (true) {
        if (matchAny({TokenType::PLUS_PLUS, TokenType::MINUS_MINUS})) {
            auto node = std::make_unique<UnaryExpr>();
            node->op = previous().type; node->operand = std::move(expr); node->isPrefix = false;
            expr = std::move(node);
        } else if (match(TokenType::DOT)) {
            // could be a library call (console.write) or an instance method (myFile.write) --
            // disambiguated during semantic analysis, not here; parse structurally the same way
            std::string name = expect(TokenType::IDENTIFIER, "expected name after '.'").text;
            expect(TokenType::LPAREN, "expected '(' after method/library-function name");
            auto args = parseArgList();
            expect(TokenType::RPAREN, "expected ')' after arguments");
            auto node = std::make_unique<MethodCallExpr>();
            node->instance = std::move(expr); node->methodName = name; node->args = std::move(args);
            expr = std::move(node);
        } else if (match(TokenType::LPAREN)) {
            auto args = parseArgList();
            expect(TokenType::RPAREN, "expected ')' after arguments");
            auto node = std::make_unique<CallExpr>();
            node->callee = std::move(expr); node->args = std::move(args);
            expr = std::move(node);
        } else if (match(TokenType::LBRACKET)) {
            // list/dict element access, e.g. nums[0], myDict["key"] -- CONFIRMED,
            // see 02-language-spec.md section 2a
            auto index = parseExpression();
            expect(TokenType::RBRACKET, "expected ']' after index expression");
            auto node = std::make_unique<IndexExpr>();
            node->container = std::move(expr); node->index = std::move(index);
            expr = std::move(node);
        } else {
            break;
        }
    }
    return expr;
}
```

**Implementation note on disambiguating `console.write(...)` (library call) from
`myFileInstance.write(...)` (instance method call) from `mathutils.square(...)` (whole-file-import
prefixed call):** the parser doesn't need to tell these apart — they all parse identically as
"postfix `.` followed by a call." The distinction is resolved later, during semantic analysis, by
looking up whether the identifier before the `.` refers to: (a) a known built-in/imported library
name, or (b) a local variable of type `file` (or another type with instance methods, if that set
ever grows). This keeps the parser simple and pushes the type-dependent disambiguation to the stage
that actually has symbol-table information available.

---

## 4. Import Resolution

---

## 4. Import Resolution

Given the entry file, recursively:
1. Parse it.
2. For every `using` statement, resolve the referenced `.az` file relative to the entry file's
   directory (exact resolution rules — same directory only vs. a search path — are an open detail;
   same-directory-only is the simplest default and is a reasonable starting assumption).
3. Parse that file too, if not already parsed.
4. Repeat until no new files are discovered.

Build one combined "program" structure holding: the entry file's AST, all imported files' ASTs, and
a flattened record of exactly which functions from each imported file were actually brought into
scope (whole-file imports bring in everything; specific imports bring in only the named function).

This flattened "what's actually visible" list is what the collision-checking pass (next section)
operates over — **not** the raw contents of every parsed file, since a function that exists in an
imported file but was never actually imported (in a selective `using foo.bar;` import) must be
completely invisible to collision checking, per the confirmed design.

---

## 4a. Import resolution algorithm, with cycle detection

**Note not previously flagged elsewhere: circular imports.** Nothing in the design conversation
addressed what happens if `a.az` imports `b.az` which imports `a.az` (directly or through a longer
chain). This must be handled defensively regardless -- an unguarded recursive resolver would infinite-
loop. Recommended approach: track "currently being resolved" files, and treat a cycle as a compile
error (consistent with how most languages with file-based imports behave; nothing suggests Az should
allow or need circular imports).

```cpp
struct ImportResolver {
    std::unordered_map<std::string, AzFile> resolvedFiles;   // path -> parsed file
    std::unordered_set<std::string> inProgress;                // cycle detection

    AzFile& resolve(const std::string& path) {
        if (resolvedFiles.count(path)) return resolvedFiles[path];   // already done

        if (inProgress.count(path)) {
            // Cycle detected -- fatal compile error, e.g.:
            // "E016: circular import detected involving '<path>'"
            fatalError("circular import detected involving '" + path + "'");
        }

        inProgress.insert(path);
        AzFile file = parseFile(path);   // lexer + parser for this one file

        for (const UsingStatement& u : file.usingStatements) {
            if (isBuiltinLibrary(u.target)) continue;   // console/math/random/time/file/logs --
                                                          // not real files, skip resolution entirely,
                                                          // per 02-language-spec.md section 7
            std::string resolvedPath = resolveImportPath(u.target, /* relative to */ path);
            resolve(resolvedPath);   // recursive call
        }

        inProgress.erase(path);
        resolvedFiles[path] = std::move(file);
        return resolvedFiles[path];
    }
};
```

`resolveImportPath` implements the confirmed rule from `02-language-spec.md` section 7: resolve
relative to the file *containing* the `using` statement (the `path` parameter here, not the entry
file and not the current working directory), and pass absolute paths through unchanged.

---

## 5. Symbol Table Pre-Pass

---

## 5. Symbol Table Pre-Pass

Before generating any code, walk every function declaration across every file actually in scope
(entry file + everything brought in via `using`, per the previous section) and register:

- Function name
- Parameter types (needed for overload resolution)
- Return type
- Which file it came from
- Whether it requires a `library.` prefix to call (whole-file import) or is called bare (selective
  import) — **this affects only how the call is written, not identity for collision purposes**

**Collision rule (must be enforced here, exactly as specified):** two functions collide if they
share the same bare name, *regardless* of parameter types/counts and *regardless* of whether one or
both require a prefix to call. This is a fatal compile error. This is different from normal C++
overload resolution (which allows same-name/different-signature) — Az explicitly does NOT use
signature-based disambiguation for this check. Overloading (same name, different params) is still
allowed *within* a single, intentional definition site (i.e., a user can write two versions of
`func print(int x)` and `func print(string x)` in the *same* file as deliberate overloads of a
function they're authoring) — the collision rule specifically targets accidental name clashes
**across separately-authored/imported code**. (Implementation note: the cleanest way to reconcile
these two facts — overloading is allowed, but cross-file name collision is fatal regardless of
signature — is that overload sets are only permitted to grow within their original declaring file;
once a name is imported or referenced from elsewhere with a colliding name from a different origin,
it's an error even if the signatures differ. Flag this nuance for the user to confirm if it comes up
during implementation, as the original conversation's examples were about accidental duplicate
names, not intentional multi-file overload sets.)

### Collision detection algorithm (concrete sketch)

Implements the `overload`-keyword rule confirmed in `02-language-spec.md` section 7 (which
supersedes the earlier no-exceptions collision rule):

```cpp
struct FunctionSignature {
    std::string name;
    std::vector<Type> paramTypes;
    bool operator==(const FunctionSignature& o) const {
        return name == o.name && paramTypes == o.paramTypes;   // Type needs a matching operator==
    }
};

struct SymbolTable {
    // name -> all known signatures declared under that name so far
    std::unordered_map<std::string, std::vector<FunctionSignature>> byName;

    void registerFunction(const FunctionDecl& fn) {
        FunctionSignature sig{fn.name, extractParamTypes(fn.params)};
        auto& existing = byName[fn.name];

        if (existing.empty()) {
            // First declaration of this name -- always fine, no `overload` needed
            existing.push_back(sig);
            return;
        }

        if (!fn.isOverload) {
            // E003: reusing a name without `overload`
            fatalError("'" + fn.name + "' is already defined -- add 'overload' to declare "
                       "an intentional overload");
        }

        for (const auto& other : existing) {
            if (other == sig) {
                // E004: `overload` used, but signature is identical to an existing one
                fatalError("'" + fn.name + "(...)' conflicts with an existing declaration "
                           "of the exact same signature");
            }
        }

        existing.push_back(sig);   // legal overload -- different signature, `overload` present
    }
};
```

Run this once per function actually in scope for the current compile (entry file's own functions,
plus everything brought in via `using`, respecting whole-file-vs-selective import scoping from
section 4 above) -- order doesn't matter for correctness here, since the rule only cares about the
*set* of declarations sharing a name, not which one came "first" in any meaningful sense beyond
"which one happened to be registered first in this pass."

This pre-pass is also what enables **forward references** — a function can be called anywhere in the
program regardless of whether its textual definition appears before or after the call site, because
by the time code generation starts, every function's full signature is already known.

---

## 6. Semantic Analysis

Additional checks beyond the symbol table pass:

- **Exactly one `main()`** across the whole program, and it must be in the entry file specifically
  (not merely present somewhere in the combined program). Any imported file containing a `main()` is
  a fatal error.
- **No executable statements outside function bodies**, anywhere.
- **Type checking**, including:
  - Implicit numeric conversion allowed between `int`/`double`/`long long` (see rounding rule below)
  - `string + string` valid; `string + <non-string>` is a compile error
  - Function return types can never be declared with `const`/`global`/`temp` qualifiers — these only
    apply to variables at the point they receive a value, never to a function's declared return type
    itself
- **`global` collection for codegen**: since a `global` variable must be usable *even before its
  textual declaration point*, the compiler cannot rely on source order for globals. During this
  pass, collect every `global` declaration across the whole program and record it separately, so
  codegen (section 7.4 below) can generate the lazy-initialization accessor function for each one —
  this collection step doesn't need to hoist anything to a specific position in the generated C++
  the way an earlier draft of this document described (that approach is superseded — see section
  7.4 for the current, confirmed lazy-init design), it just needs to know the full set of `global`
  names and their declared initial values/types before codegen runs, regardless of where in the
  `.az` source each one was written.
- **`temp` / `global` incompatibility check**: a `temp` variable cannot be assigned into a `global`
  variable, and more generally must be fully consumed (used, and — if ever deleted — deleted via
  `tempdelete`) within the same scope it was declared in. (Since v1 has no references/pointers,
  assigning a `temp`'s *value* into a `global` is actually fine, per the earlier design discussion —
  it's a copy, not an escape. What must be prevented is a `temp` variable *itself* being referenced
  outside its declaring scope, which can't structurally happen in v1 anyway since there's no
  mechanism to hold a reference to it. Treat this primarily as a documentation/spec point rather than
  a complex check — implementation should confirm this with the user if any ambiguous case arises.)

---

## 7. Code Generation (Az → C++)

### Combined single-file output — CONFIRMED

**Running `az-transpiler` on one entry file produces exactly one `.cpp` file, containing everything**
— the entry file's own functions, plus every function pulled in (transitively) through `using`
statements, all emitted into that single file. This is a confirmed decision, **not** the
"one `.cpp` per source file" approach an earlier draft of this document suggested — that earlier
approach is superseded.

Concretely: after the import resolution pass (section 4/4a above) has produced the full set of
`.az` files actually reachable from the entry file, codegen walks that whole set and emits every
function's translated C++ into one output stream, in some deterministic order (e.g. entry file's
functions first, then each imported file's functions in the order they were first encountered).
`main()` — which per `02-language-spec.md` section 7 must exist exactly once, only in the entry
file — becomes the generated file's own `int main()`.

**Diamond imports are handled automatically by the existing import-resolution cache, not by any
extra logic.** If `main.az` uses both `a.az` and `b.az`, and `a.az` *itself* also uses `b.az`, this
is **not a cycle** (a cycle would be `a.az` using `b.az` using `a.az` again — see section 4a's
`inProgress` cycle check) — it's a "diamond": `b.az` is reachable from `main.az` by two different
paths, but it's still just one file. The `ImportResolver`'s `resolvedFiles` cache (section 4a)
already deduplicates this correctly: the first time `b.az` is resolved (via whichever path reaches
it first), it's parsed and cached; the second time it's reached (via the other path), the cached
result is returned immediately without re-parsing or re-registering its functions. So `b.az`'s
functions end up emitted into the combined output **exactly once**, regardless of how many
different `using` chains lead to it. No additional deduplication logic is needed beyond what
section 4a already describes — this is called out explicitly here because it's easy to assume a
diamond needs special handling when it actually falls out for free from the existing cache.

A shared Az runtime support header/library provides implementations for all built-ins (`console`,
`math`, `random`, `time`, `file`, `logs`, plus general helpers like the `fstring`/`string()`/
`int()`/`double()`/`LL()` conversion machinery) — this runtime is `#include`d by (or otherwise
linked against) the one generated `.cpp` file, same as before; the single-combined-output decision
only changes how many `.cpp` files the *user's* `.az` program turns into (one, always), not how the
runtime library itself is organized.

### 7.1 Numeric conversion — rounding vs. truncation (IMPORTANT, deliberate deviation from C++)

Raw C++ truncates on implicit `double`→`int` conversion (`(int)3.7` → `3`). Az requires **rounding**
instead (`3.7` → `4`). The generated C++ must never rely on a bare C-style/static cast for this
conversion. Instead, emit something equivalent to:

```cpp
// Az: int x = someDouble;
// Generated C++:
int x = static_cast<int>(std::round(someDouble));
```

This must be applied consistently everywhere an implicit `double`→`int` (or `double`→`long long`)
conversion occurs: variable initialization, assignment, function parameter passing, and function
return values. Recommend implementing this as a single codegen helper function
(e.g. `emit_numeric_conversion(fromType, toType, expr)`) that all these call sites route through, so
the rounding behavior lives in exactly one place.

### 7.2 `fstring(...)` codegen

`fstring("... {expr} ...")` must be parsed (at the *Az* parser level, not left as an opaque string)
so that the contents of each `{}` are recognized as a real sub-expression in the Az grammar — they
can contain arithmetic, function calls, variable references, anything that's a valid Az expression.

Recommended generation strategy: split the template string into a sequence of literal-text and
expression segments, generate C++ code that evaluates each expression segment, converts it to
`std::string` via the same conversion machinery backing the `string()` built-in (see
`04-library-math-random.md`/general library notes — `string()` should be a single overloaded/templated
runtime helper, e.g. `az::to_string<T>(value)`), and concatenates everything with `+` or into a
`std::ostringstream`.

**Compile-time warning requirement:** if a `{}` segment's expression does not have static type
`string` or `char`, emit a compiler warning (not an error) at that call site — something like
`warning: fstring() argument is not a string/char, will be auto-converted (<file>:<line>)`. This is
a deliberate ergonomics/safety-net decision — the designer specifically wants this to catch
accidental large-value dumps (e.g. printing a whole `list` where a single value was intended) without
blocking legitimate uses.

### 7.3 `temp` / `tempdelete` codegen

**This mechanism's underlying implementation was not explicitly specified in the design
conversation and needs a concrete approach — recommendation below.**

The requirement: a `temp` variable behaves exactly like a normal variable (same syntax, same access
patterns, gets cleaned up automatically at scope exit if never manually deleted) — the *only*
difference is that it can *additionally* be deleted early, on demand, via `tempdelete`.

Recommended implementation: wrap `temp` variables in a small RAII holder type in the C++ runtime,
e.g. conceptually:

```cpp
template <typename T>
class AzTemp {
    std::optional<T> value;
public:
    AzTemp(T initial) : value(std::move(initial)) {}
    T& get() { /* assert value.has_value(), else runtime error */ return *value; }
    void del() { value.reset(); }
    ~AzTemp() { /* no-op if already deleted; otherwise value's destructor runs normally here */ }
};
```

- `temp int x = 5;` → `AzTemp<int> x(5);`
- using `x` elsewhere → `x.get()` (transparently generated by the compiler wherever the Az source
  references the identifier — the Az programmer never sees `.get()`)
- `tempdelete x;` → `x.del();`
- Falling out of scope without a `tempdelete` → the `AzTemp` destructor runs normally, matching
  standard C++ stack-unwind behavior — no special-casing needed, this "just works" via RAII.

This keeps `temp`'s behavior 1:1 with what was specified (normal variable, optionally
early-deletable, auto-cleaned otherwise) without requiring real heap-pointer exposure to the Az
programmer.

### 7.4 `global` codegen — lazy initialization via function-local statics

**Confirmed approach: a `global` should not be allocated/constructed until it is actually first
used somewhere in the running Az program** — not eagerly at program start. This is a deliberate
choice (a plain top-level C++ global would construct at program startup regardless of whether it's
ever used) and is implemented using a well-known, safe C++ pattern: wrapping each global in a
function that holds a `static` local variable. C++ guarantees a function-local `static` initializes
the first time (and only the first time) that function is actually called — exactly the "reserve
memory and run the constructor only when it's called in the Az code" behavior wanted.

**Codegen mapping:**

```
Az:                                      Generated C++:
global int visitCount = 0;               int& az_global_visitCount() {
                                              static int value = 0;
                                              return value;
                                          }
```

Every **use** of a `global`-qualified variable anywhere in the generated code — reads, writes,
passing it to a function, etc. — is rewritten to call this accessor function instead of referencing
a plain variable name directly:

```
Az:                                      Generated C++:
visitCount = visitCount + 1;             az_global_visitCount() = az_global_visitCount() + 1;
console.write(fstring("{visitCount}"));  az::console_write(az::to_string(az_global_visitCount()));
```

This achieves both required properties at once:
- **Available everywhere, even "before" its textual declaration** (per
  `02-language-spec.md` section 3) — since every reference is just a function call, it doesn't
  matter where in the source the `global` statement itself appears; the accessor function exists
  (as a declaration) across the whole generated file regardless.
- **Lazy construction** — the `static int value = 0;` inside `az_global_visitCount()` is not
  initialized until the first time something actually calls `az_global_visitCount()`, matching the
  designer's request precisely.

One consequence worth noting: since the accessor returns a reference (`int&`), assignment through
it (`az_global_visitCount() = ...`) works correctly in generated C++ exactly like assigning to a
normal variable would — this is standard, idiomatic C++ (the "Meyers' singleton" pattern uses the
same trick), not a hack specific to Az.

### 7.5 Function overloading

Since the transpile target is C++, which natively supports overloading, this is close to a direct
pass-through: multiple Az `func` declarations with the same name and differing parameter
types/counts (within the collision rules described in section 5) simply become multiple C++
function overloads with the same name. No special dispatch logic needs to be hand-written.

### 7.6 Built-ins that need to look like flexible/generic functions

Some built-ins (documented individually in the library files) present a flexible-looking interface
at the Az level — e.g. `console.input(int)` "returning `int`" and `console.input(string)` "returning
`string`" from what looks like one conceptual function — but Az v1 has no user-facing generics. The
resolution (already agreed in the design conversation): this is purely a compiler-level illusion.
**The generated C++ should implement these as real C++ templates in the runtime library**
(e.g. `template<typename T> T az_console_input();`), with the Az compiler recognizing the special
built-in call syntax and emitting the correctly-instantiated template call. This keeps the runtime
implementation non-duplicated while the Az-facing syntax stays simple. Apply this same principle to
any other built-in whose underlying logic is naturally generic over a type, even if it looks like
flat overloads from the Az side.

### 7.6a `skip` / `skip here` — codegen and legality checking

**Codegen is a direct, almost 1:1 mapping onto C++ labels and `goto`:**

```
Az:                              Generated C++:
skip here point;                 point: ;
skip point;                      goto point;
```

This is safe specifically because of the legality rule (below) — C++ only forbids `goto` from
jumping **into** a scope past variable initializations; jumping **out** of nested scopes (which is
all this rule ever permits) is always well-defined, and correctly runs destructors for anything
going out of scope along the way — including `AzTemp<T>` (section 7.3), so `temp` variables get
cleaned up properly even when a `skip` jumps out past them.

**Legality-checking algorithm** (semantic analysis stage, before codegen):

Represent each block with a unique ID and a pointer to its parent block (the block that lexically
contains it), forming a tree per function. While walking a function's AST:

```cpp
struct BlockScope {
    int id;
    BlockScope* parent;   // nullptr for the function's top-level block
};

struct SkipHereLabel {
    std::string name;
    BlockScope* declaringBlock;
    int line, col;
};

struct SkipJump {
    std::string targetName;
    BlockScope* jumpBlock;
    int line, col;
};

void validateSkips(const std::vector<SkipHereLabel>& labels, const std::vector<SkipJump>& jumps) {
    // 1. No duplicate label names within one function
    std::unordered_map<std::string, const SkipHereLabel*> byName;
    for (const auto& label : labels) {
        if (byName.count(label.name)) {
            fatalError("'skip here " + label.name + "' is already declared in this function "
                       "(at line " + std::to_string(byName[label.name]->line) + ")");
        }
        byName[label.name] = &label;
    }

    // 2. Every jump must have a matching label, and the label's block must be an
    //    ancestor of (or equal to) the jump's block
    for (const auto& jump : jumps) {
        if (!byName.count(jump.targetName)) {
            fatalError("'skip " + jump.targetName + "' has no matching "
                       "'skip here " + jump.targetName + "' in this function");
        }
        BlockScope* target = byName[jump.targetName]->declaringBlock;

        bool found = false;
        for (BlockScope* b = jump.jumpBlock; b != nullptr; b = b->parent) {
            if (b == target) { found = true; break; }
        }
        if (!found) {
            fatalError("'skip " + jump.targetName + "' would jump into a scope it was "
                       "never inside -- the target label must be in the same block or "
                       "an enclosing block relative to this jump");
        }
    }
}
```

This directly implements the rule from `02-language-spec.md`: walking outward from the jump's own
block through its ancestor chain, the label's block must appear somewhere along that walk (including
being the jump's own block). If it isn't found by the time the walk reaches the function's top-level
block (`parent == nullptr`), the jump is illegal.

**Scoping note:** since this validation is run per-function, `skip`/`skip here` naturally can't
cross function boundaries — a label declared in one function is simply never visible to this check
when validating a different function, which matches how C++'s own labels work (function-scoped), so
no extra enforcement is needed beyond running this pass once per function independently.

### 7.6b `list`/`dict` element access (`IndexExpr`) — codegen

Implements the confirmed behavior in `02-language-spec.md` section 2a: Python-style bracket access,
negative `list` indices, bounds/key checking that reports through the runtime diagnostics system
(`12-runtime-diagnostics.md`) as a fatal `ERROR`, and the deliberate `dict` read/write asymmetry
(read on a missing key errors; write on a missing key auto-inserts).

Recommend implementing this via small runtime helper functions in `az_runtime.hpp` (or a dedicated
`az_collections.hpp`) rather than emitting raw `std::vector`/`std::map` `operator[]` calls directly
— raw `operator[]` doesn't bounds-check for `std::vector` at all (silent UB) and auto-inserts for
`std::map` on *any* access including reads (not just writes), neither of which matches the confirmed
Az semantics:

```cpp
template <typename T>
T& az_list_index(std::vector<T>& list, long long index) {
    long long size = static_cast<long long>(list.size());
    long long resolved = (index < 0) ? (size + index) : index;   // negative-index support
    if (resolved < 0 || resolved >= size) {
        az::g_runtimeDiagnostics.report(az::RuntimeLevel::ERROR,
            "list index out of range (index " + std::to_string(index) +
            ", size " + std::to_string(size) + ")");
        // report() on ERROR terminates the program (std::exit) -- see
        // az_runtime_diagnostics.hpp in the Appendix -- so execution never continues past here
    }
    return list[resolved];
}

template <typename K, typename V>
V& az_dict_read(std::map<K, V>& dict, const K& key) {
    auto it = dict.find(key);
    if (it == dict.end()) {
        az::g_runtimeDiagnostics.report(az::RuntimeLevel::ERROR,
            "dict key not found");   // (message would include the key's string representation
                                       // via az::to_string, omitted here for brevity)
    }
    return it->second;
}

template <typename K, typename V>
V& az_dict_write(std::map<K, V>& dict, const K& key) {
    return dict[key];   // std::map's own operator[] already auto-inserts on write --
                          // this is exactly the desired behavior for the WRITE path specifically,
                          // it's only unsafe to use directly for READS (which az_dict_read avoids
                          // by using find() instead)
}
```

Codegen mapping (the compiler must distinguish read vs. write position for `dict` — an `IndexExpr`
appearing as the target of an `AssignExpr` uses `az_dict_write`, everywhere else uses
`az_dict_read`; `list` uses `az_list_index` uniformly for both, since list access is symmetric):

```
Az:                          Generated C++ (list):              Generated C++ (dict):
nums[0]                      az_list_index(nums, 0)              az_dict_read(myDict, key)
nums[0] = 5;                 az_list_index(nums, 0) = 5;          az_dict_write(myDict, key) = 5;
```

### 7.7 Opaque handle types (`time` stopwatch/timer)

`time.startStopwatch()` and `time.startTimer(...)` return something that isn't one of Az's core
value types (`int`/`double`/`bool`/`string`/`char`/`list`/`dict`) — it's an opaque handle. This is
the first case in the spec requiring a genuinely new built-in type.

Recommended implementation: define runtime C++ classes (e.g. `AzStopwatch`, `AzTimer`) that
internally hold a `std::shared_ptr<Impl>` (or equivalent) to their actual timing state. The handle
"variable" at the Az level is a value-type wrapper around that shared pointer — so even though Az
semantics say everything is pass-by-value, *multiple handle variables referring to the same
underlying stopwatch* isn't a concern here because each `startStopwatch()` call creates one handle
that the user holds onto and passes to the various `time.*Stopwatch(handle)` functions — there's no
scenario in the current spec where a handle needs to be duplicated/copied and still refer to the
same timer, so this can be kept simple (a straightforward class instance is sufficient; the
shared_ptr suggestion is only relevant if duplication-with-shared-state ever becomes a requirement).

---

## 8. Invoking the C++ toolchain

Once all `.cpp` files are generated, shell out to `g++` or `clang++` (recommend detecting whichever
is available, defaulting to one) with all generated files plus the Az runtime support library,
producing the final executable at the user-specified (or default) output path. This step is a
straightforward subprocess invocation — the interesting work is entirely upstream of it.

---

## 9. Project structure — CONFIRMED constraint: everything lives under one `CONTENTS/` folder

**Confirmed requirement:** the entire project — every file, with no exceptions — must live inside a
single top-level folder named **`CONTENTS`**. Nothing belongs in any parent directory above it.
Within `CONTENTS`:
- **`CONTENTS/source-code/`** holds all of the transpiler's own source code (both the compiler's
  own C++ source and the runtime support library it generates code against).
- **`CONTENTS/info.md`** — a single info file at the root of `CONTENTS`. **This file is written by
  whichever AI assistant is doing the actual implementation work (via a separate prompt the
  designer provides directly to it) — it is not something this spec set dictates the content of.**
  Its location is reserved here so the overall structure is complete, but don't invent its contents.

```
CONTENTS/
├── info.md                       # written separately by the implementing assistant -- not
│                                   # specified by this document, just reserving its location
├── source-code/
│   ├── compiler/                  # the az-transpiler executable's own source (C++)
│   │   ├── lexer.hpp / lexer.cpp
│   │   ├── token.hpp              # TokenType enum, Token struct -- see section 2a
│   │   ├── ast.hpp                # AST node definitions -- see section 3a
│   │   ├── parser.hpp / parser.cpp
│   │   ├── imports.hpp / imports.cpp   # import resolution + cycle detection -- section 4/4a
│   │   ├── symbol_table.hpp / .cpp      # pre-pass + collision checking -- section 5
│   │   ├── semantics.hpp / .cpp         # semantic analysis -- section 6
│   │   ├── codegen.hpp / .cpp           # C++ code generation, single combined output -- section 7
│   │   ├── diagnostics.hpp / .cpp       # shared logging module -- section 7.8 / 09-transpile-logging.md
│   │   ├── cli.hpp / .cpp               # entry point -- see 10-cli.md
│   │   └── main.cpp
│   └── runtime/                   # the C++ support library every generated program links against
│       ├── az_runtime.hpp         # core helpers: to_string/fstring machinery, conversion helpers
│       ├── az_temp.hpp            # AzTemp<T> RAII wrapper -- section 7.3
│       ├── az_console.hpp / .cpp
│       ├── az_math.hpp
│       ├── az_random.hpp / .cpp
│       ├── az_time.hpp / .cpp
│       ├── az_file.hpp / .cpp
│       ├── az_logs.hpp / .cpp
│       └── az_runtime_diagnostics.hpp / .cpp   # runtime_errors.log -- 12-runtime-diagnostics.md
├── tests/
│   ├── lexer_tests/
│   ├── parser_tests/
│   ├── semantic_tests/
│   ├── codegen_tests/
│   ├── e2e/
│   └── negative/               # see 16-testing-strategy.md for the full test layout
└── examples/
    └── (the programs from 14-example-programs.md)
```

The `source-code/compiler/` vs. `source-code/runtime/` split, and the exact file names within each,
are still just a suggested organization *within* the confirmed `CONTENTS/source-code/` constraint —
adjust freely as implementation needs dictate, as long as everything stays under `CONTENTS/` and the
transpiler's own source stays under `CONTENTS/source-code/` specifically, per the confirmed rule
above. See the Appendix at the end of this file for concrete C++ header sketches for several of the
`runtime/` files.

---

## Appendix: Runtime library header sketches

Concrete starting points for the `runtime/` files referenced in section 9's project structure.
These are illustrative sketches, not a complete/final implementation — they exist to give the
implementer a real starting shape rather than an empty file, consistent with everything else in
this document being labeled recommendations where the original design conversation didn't dictate
exact code.

### `az_runtime.hpp` — core conversion helpers (`fstring`/`string()` machinery, rounding)

```cpp
#pragma once
#include <string>
#include <cmath>
#include <sstream>

namespace az {

// Generic to_string, specialized per type -- backs both fstring()'s {} segments and the
// string() built-in. See 02-language-spec.md section 8 and 01-compiler-architecture.md 7.2.
template <typename T>
std::string to_string(const T& value) {
    std::ostringstream oss;
    oss << value;
    return oss.str();
}

inline std::string to_string(bool value) { return value ? "true" : "false"; }
inline std::string to_string(const std::string& value) { return value; }
inline std::string to_string(char value) { return std::string(1, value); }

template <typename T>
std::string to_string(const std::vector<T>& list) {
    std::string result = "[";
    for (size_t i = 0; i < list.size(); i++) {
        result += to_string(list[i]);
        if (i + 1 < list.size()) result += ", ";
    }
    return result + "]";
}

// The deliberate double -> int rounding deviation from raw C++ truncation.
// See 01-compiler-architecture.md section 7.1 -- ALL implicit double->int/long long
// conversions in generated code must route through this, never a bare static_cast.
inline int round_to_int(double value) {
    return static_cast<int>(std::round(value));
}
inline long long round_to_long_long(double value) {
    return static_cast<long long>(std::round(value));
}

} // namespace az
```

### `az_temp.hpp` — the `temp`/`tempdelete` RAII wrapper

```cpp
#pragma once
#include <optional>
#include <stdexcept>

namespace az {

// See 01-compiler-architecture.md section 7.3 for the design rationale.
template <typename T>
class AzTemp {
    std::optional<T> value;
public:
    explicit AzTemp(T initial) : value(std::move(initial)) {}

    T& get() {
        if (!value.has_value()) {
            throw std::runtime_error("use of temp variable after tempdelete");
        }
        return *value;
    }

    void del() { value.reset(); }

    // Falling out of scope without tempdelete: std::optional's destructor handles cleanup
    // automatically -- no custom destructor logic needed here.
};

} // namespace az
```

### `az_console.hpp` — `console.input`'s templated implementation

```cpp
#pragma once
#include <iostream>
#include <string>
#include <sstream>
#include "az_runtime.hpp"

namespace az {

// See 01-compiler-architecture.md section 7.6 -- console.input(type) / console.input(type, msg)
// is implemented as ONE templated function per type, not duplicated per-type logic.
template <typename T>
T console_input(const std::string& errorMessage = "") {
    std::string line;
    while (true) {
        std::getline(std::cin, line);
        std::istringstream iss(line);
        T value;
        if (iss >> value && iss.eof()) {
            return value;
        }
        // Default message per 03-library-console.md: fstring("Invalid input type! Enter {type}")
        std::cout << (errorMessage.empty()
            ? "Invalid input type! Enter " + type_name<T>()
            : errorMessage);
    }
}

// Specialization for string: no parsing/validation needed, any line is valid.
template <>
inline std::string console_input<std::string>(const std::string&) {
    std::string line;
    std::getline(std::cin, line);
    return line;
}

inline void console_write(const std::string& s) { std::cout << s; }

inline std::string console_run(const std::string& command) {
    // Recommended: popen()-based capture rather than a bare system() call, so output
    // can actually be returned per 03-library-console.md's console.run spec.
    std::string result;
    char buffer[256];
    FILE* pipe = popen(command.c_str(), "r");
    if (!pipe) return "";
    while (fgets(buffer, sizeof(buffer), pipe) != nullptr) result += buffer;
    pclose(pipe);
    return result;
}

inline void console_clear() {
#ifdef _WIN32
    system("cls");
#else
    system("clear");
#endif
}

} // namespace az
```

(`type_name<T>()` above is a small helper template that would need one specialization per supported
type, returning `"int"`, `"double"`, `"bool"`, etc. — used to build the default error message.)

### `az_time.hpp` — stopwatch/timer handles

```cpp
#pragma once
#include <chrono>

namespace az {

// See 01-compiler-architecture.md section 7.7 for the design rationale.
class AzStopwatch {
    std::chrono::steady_clock::time_point startTime;
    double accumulatedSeconds = 0.0;
    bool paused = false;
public:
    AzStopwatch() : startTime(std::chrono::steady_clock::now()) {}

    void pause() {
        if (!paused) {
            accumulatedSeconds += elapsedSinceStart();
            paused = true;
        }
    }
    void resume() {
        if (paused) {
            startTime = std::chrono::steady_clock::now();
            paused = false;
        }
    }
    double elapsed() const {
        return accumulatedSeconds + (paused ? 0.0 : elapsedSinceStart());
    }
    // Confirmed behavior: using a handle after stop() returns the frozen value, not an error --
    // this naturally falls out of pause()'s accumulation logic if stop() just calls pause().
    void stop() { pause(); }

private:
    double elapsedSinceStart() const {
        return std::chrono::duration<double>(std::chrono::steady_clock::now() - startTime).count();
    }
};

class AzTimer {
    double durationSeconds;
    AzStopwatch internalStopwatch;
public:
    explicit AzTimer(double seconds) : durationSeconds(seconds) {}
    void pause() { internalStopwatch.pause(); }
    void resume() { internalStopwatch.resume(); }
    double remaining() const {
        double r = durationSeconds - internalStopwatch.elapsed();
        return r > 0.0 ? r : 0.0;
    }
    bool isDone() const { return remaining() <= 0.0; }
    void stop() { internalStopwatch.stop(); }
};

} // namespace az
```

### `az_runtime_diagnostics.hpp` — `runtime_errors.log` writer

```cpp
#pragma once
#include <fstream>
#include <string>
#include <chrono>
#include <cstdlib>

namespace az {

enum class RuntimeLevel { ERROR, WARNING };   // deliberately no INFO -- 12-runtime-diagnostics.md

class RuntimeDiagnostics {
    std::ofstream log;
public:
    // Wiped fresh each run -- opened in truncate mode, next to the executable.
    explicit RuntimeDiagnostics(const std::string& exeDir) {
        log.open(exeDir + "/runtime_errors.log", std::ios::trunc);
    }

    void report(RuntimeLevel level, const std::string& message) {
        std::string levelStr = (level == RuntimeLevel::ERROR) ? "ERROR" : "WARNING";
        log << "[" << levelStr << "] at " << currentTimeHHMMSS()
            << " - " << message << std::endl;

        if (level == RuntimeLevel::ERROR) {
            log.close();
            std::exit(1);   // terminate the program, per 12-runtime-diagnostics.md
        }
    }

private:
    std::string currentTimeHHMMSS() const {
        // Implementation detail: format current time as hh:mm:ss, matching the format
        // established in 12-runtime-diagnostics.md and time.now()'s token conventions.
        // (Left as a stub here -- straightforward with <chrono>/<ctime>.)
        return "00:00:00";
    }
};

// A single global instance, constructed once at program startup, used by every runtime
// library function that needs to report a diagnostic (file.rename mismatches,
// logs.write with no log loaded, math.factorial misuse, etc.)
extern RuntimeDiagnostics g_runtimeDiagnostics;

} // namespace az
```

This appendix is intentionally not exhaustive (no `az_math.hpp`, `az_random.hpp`, `az_file.hpp`,
`az_logs.hpp` sketches included here) — the four included above cover the trickiest/most
novel implementation patterns (templated built-ins, RAII wrappers, handle types, the diagnostics
sink), which are the ones most valuable to have a concrete starting shape for. The remaining
runtime files are comparatively mechanical once these patterns are established.
