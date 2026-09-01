# Az Language Specification (v1.0)

This is the complete, confirmed syntax and semantics of Az. Every rule here was explicitly decided
in the design conversation. Nothing in this file should be treated as a suggestion — it's the
contract the compiler must implement. Anything genuinely undecided is called out explicitly and/or
listed in `08-open-questions.md`, never silently invented.

---

## 1. General syntax rules

- File extension: **`.az`**
- **Whitespace and newlines are never semantically meaningful.** A legal program could be written
  entirely on one line (except that comments still work the normal way — a `//` comment still
  extends to the end of its physical line, since that's how `//` comments inherently work in any
  language, but this doesn't make newlines *meaningful* in the sense of ending statements).
- **`;` terminates every statement.** There is no alternative terminator and no context where it can
  be omitted.
- **Scoping uses curly braces `{ }`**, exactly like C++.
- **Comments:**
  - `//` — rest of the line is a comment
  - `/* ... */` — multi-line block comment

---

## 2. Types

| Type | Notes |
|---|---|
| `int` | signed by default; `unsigned int` available |
| `double` | signed by default; `unsigned double` available |
| `long long` | signed by default; `unsigned long long` available |
| `bool` | `true` / `false` |
| `string` | always double-quoted: `"like this"`. Never interchangeable with `char`. |
| `char` | always single-quoted: `'a'`. Exactly one character. Never interchangeable with `string`. |
| `list<T>` | e.g. `list<int> nums = [];` |
| `dict<K, V>` | e.g. `dict<string, int> scores = {};` |
| `var` | type inference, equivalent to C++'s `auto`. Discouraged as a default habit — intended for cases where the type is genuinely non-obvious, not as a routine replacement for explicit types, since most of the time the function/expression being assigned already makes the type clear. |
| `none` | (a) the "no return value" type for functions (like `void`); (b) reused as the literal filler for an omitted clause in a C-style `for` loop (see Control Flow) |

`signed` / `unsigned` work exactly like C++ and apply to the numeric types (`int`, `double`,
`long long`). Default is `signed`.

**Numeric literals: plain digits only.** There is no special literal syntax for `long long` or
`unsigned` (no C++-style suffixes like `100LL` or `100u`), and **no scientific notation** (no
`1e10`). A numeric literal is just plain digits (`100`) or digits with one decimal point (`3.14`) —
see `13-grammar.md` section 6. A plain integer literal like `100` works in any numeric-type context
(`int`, `long long`, `unsigned int`, etc.) via the same implicit-conversion rules as any other
numeric value — there's no need for a distinguishing suffix, since the target type is always known
from context (a variable's declared type, a parameter's type, etc.).

### Numeric conversion rules

Implicit conversion between `int`, `double`, and `long long` is allowed in variable
initialization/assignment, function parameter passing, and function return values — matching C++'s
general permissiveness here.

**Deliberate deviation from C++:** converting `double` → `int` (or `double` → `long long`) **rounds
to the nearest integer**, it does **not** truncate. (`3.7` becomes `4`, not `3`.) This must be
implemented explicitly in codegen — see `01-compiler-architecture.md` section 7.1 — because raw C++
casting truncates by default and the transpiler must not rely on that default behavior.

**No overflow protection, implicit or explicit.** Neither the implicit conversions above nor the
explicit conversion functions below add any safety net against a value not fitting in its target
type — if it doesn't fit, it silently overflows/wraps, exactly like a raw C++ cast would. Az does
not detect or reject this at compile time (in general it can't — whether a given runtime value
overflows depends on the value, not the code), and does not treat it as a runtime error either
(unlike, say, `math.factorial`'s overflow case, which is explicitly checked and treated as an error
— see `04-library-math-random.md`. General numeric conversion overflow is not checked at all).

### Explicit conversion functions: `int(value)`, `double(value)`, `LL(value)`

Three built-in conversion functions, mirroring `string(value)` (`02-language-spec.md` section 8) —
each takes a value and returns it converted to the named type: `int(value)`, `double(value)`, and
`LL(value)` (short for `long long`, since `long long` itself isn't a valid function name — it's two
tokens).

These perform **exactly the same conversion as the implicit rules above** — same rounding behavior
for `double` sources, same lack of overflow protection — the only difference is that the conversion
is **written explicitly** at the call site instead of happening invisibly through an assignment or
parameter pass. There is no behavioral distinction between `int x = someDouble;` and
`int x = int(someDouble);` — both round the same way, both overflow the same way if the value is
out of range. Use the explicit form when you want the conversion to be visible in the code, not
because it changes what happens.

```
double bigValue = 99999999999.9;
long long asLongLong = LL(bigValue);   // rounds, then may silently overflow if out of range
int asInt = int(bigValue);             // same — rounds, then may silently overflow
```

---

## 2a. `list`/`dict` element access — Python-style bracket syntax

Both `list<T>` and `dict<K,V>` support Python-style `[...]` element/key access, for both reading
and writing:

```
list<int> nums = [10, 20, 30];
int first = nums[0];        // 10
int last = nums[-1];        // 30 -- negative indices work like Python: -1 is the last element,
                              // -2 the second-to-last, and so on
nums[1] = 99;                 // [10, 99, 30]

dict<string, int> scores = {"alice": 90, "bob": 85};
int aliceScore = scores["alice"];   // 90
scores["carol"] = 70;                // inserts a new key -- see the read/write asymmetry below
scores["alice"] = 95;                // updates an existing key
```

### Bounds/key checking — fatal at runtime, matching Python's outcome (adjusted for Az's model)

Python raises `IndexError`/`KeyError` for invalid access, which — if never caught — crashes the
program. **Az has no exception-handling construct anywhere in the language** (no `try`/`catch`), so
there's no "catchable" equivalent to translate this into; the faithful translation is Az's existing
fatal-runtime-error pattern, the same one already used for other unrecoverable conditions like
`file.rename`'s path mismatch or `math.factorial`'s overflow (see `12-runtime-diagnostics.md`):

- **`list` read (`nums[i]`) out of range** — `ERROR`, written to `runtime_errors.log`, program
  terminates. This includes an out-of-range *negative* index too (e.g. `nums[-100]` on a
  3-element list).
- **`list` write (`nums[i] = value`) out of range** — **also** `ERROR`/terminates, matching
  Python's own behavior: assigning to an out-of-range list index raises in Python too (Python lists
  don't auto-grow on indexed assignment — only `append`/`insert`-style operations grow a list, and
  Az doesn't have those defined yet either — see `08-open-questions.md`). List access is symmetric:
  both reading and writing are bounds-checked the same way.
- **`dict` read (`myDict[key]`) with a missing key** — `ERROR`, written to `runtime_errors.log`,
  program terminates (mirrors Python's `KeyError` on a plain `d[key]` read).
- **`dict` write (`myDict[key] = value`) with a missing key** — **auto-inserts** the key, no error
  (mirrors Python's dict assignment behavior exactly — `d[key] = value` always succeeds, creating
  the key if it wasn't already there). This means `dict` access is deliberately **asymmetric**
  between read and write, exactly matching Python — this is not an inconsistency with `list`'s
  symmetric behavior, it's `dict` correctly mirroring what Python itself does.

---

## 3. Variables & qualifiers

Declaration form: `[qualifiers] Type name = value;` — types are always required explicitly, except
where `var` is used for inference.

```
int x = 5;
double pi = 3.14;
string greeting = "hello";
var y = someFunctionReturningAKnownType();   // legal, but use sparingly
```

### Qualifiers

- **`const`** — same semantics as C++. Available for all types.
- **`constexpr`** — same semantics as C++. Available for all types.
- **`global`** — the variable is available everywhere in the program, in every scope, **even before
  its textual declaration point in the source.** This works because of how it's implemented, not
  because the compiler physically relocates the declaration: each `global` is generated as an
  accessor function wrapping a lazily-initialized value, so every reference to it — regardless of
  where either the reference or the `global` statement itself sits in the source — resolves to a
  call to that same function (see `01-compiler-architecture.md` section 7.4 for the full
  implementation, including why this also means a `global`'s value isn't actually constructed until
  the first time it's genuinely used at runtime, not eagerly at program start).
- **`temp`** — a variable with an *additional* manual-deletion capability. Otherwise behaves exactly
  like a normal (non-temp) variable: if it's never manually deleted, it is cleaned up automatically
  at the end of its scope, same as any other variable. The only difference is you *may* delete it
  early using `tempdelete`.
- **`tempdelete <name>;`** — a statement (not a qualifier on a declaration) that deletes a `temp`
  variable immediately.

### `temp` + `global` interaction rule

A `temp` variable **must be fully handled within the same scope it was declared in** — it cannot be
allowed to "escape" its declaring scope. Assigning a `temp` variable's *value* into a `global`
variable is fine (this is just a value copy, since v1 has no references/pointers — the `global` gets
its own independent copy, completely decoupled from the `temp`'s own lifetime). What's not permitted
is anything that would let the `temp` variable *itself* persist or be referenced outside its own
scope — which, in v1, can't really happen by construction, since there is no mechanism (no pointers,
no references) that could hold onto a `temp` beyond its scope anyway. This rule is safe specifically
*because* v1 has no references/pointers — **this must be re-examined when v2 introduces
pointers/references**, since a pointer to a `temp` that outlives its scope would be a classic
use-after-free bug.

Example (from the design conversation) — a `temp` value being copied into a `global`, then the temp
is cleaned up, completely independently:

```
temp int unc = 5;
global int something = unc;   // copies the VALUE of unc into something; independent afterward
tempdelete unc;                // unc is gone; something is unaffected, still holds 5
```

### `swap`

`swap a, b;` — swaps the **values** of two variables of the same type. (Since v1 is entirely
value-based, this is the only kind of swap that's meaningful — there's no reference-swapping
distinction to make yet.)

---

## 4. Operators

| Category | Operators | Notes |
|---|---|---|
| Arithmetic | `+ - * / %` | Same as C++ |
| Comparison | `== != < > <= >=` | Same as C++, return `bool` |
| Logical | `&& \|\| !` | Same as C++. No word-based aliases (no `and`/`or`/`not`). |
| Assignment | `= += -= *= /= %=` | Same as C++ |
| Increment/decrement | `++ --` | Same as C++ |
| String concatenation | `+`, `+=` | See String Rules below |
| Ternary | `condition ? valueIfTrue : valueIfFalse` | Same as C++ |
| Bitwise | *(none in v1)* | `& \| ^ ~ << >>` explicitly deferred to a later version — not available in v1 |

### String operator rules

- `string + string` → valid, concatenates.
- `string + <anything else>` → **compile error**. No implicit stringification via `+`. (Use
  `fstring(...)` or `string(...)` — see String Rules section — for combining non-strings into text.)
- `string += string` → valid, appends.

---

## 5. Control Flow

**None of Az's control-flow constructs *require* parentheses around their condition/header.** This
is a deliberate, confirmed departure from C++ — the designer's reasoning is that C++'s required
parens around conditions add visual/typing overhead without adding real functionality.

**Parentheses are still allowed as ordinary grouping, optionally, wherever an expression appears —
including directly wrapping a whole condition.** Since `(expression)` is already legal anywhere an
expression is legal (see `13-grammar.md`'s `primary` rule), writing `if (x > 5) { }` parses exactly
the same as `if x > 5 { }` — parens here are just the normal grouping mechanism, not special
if/while/for syntax. This matters most for nested/compound conditions, where grouping clarifies
precedence:
```
if (a && b) || c { }        // groups (a && b) explicitly, though && already binds tighter than ||
if a && (b || c) { }        // here the parens change the actual grouping, not just clarify it
```
Parentheses are never required anywhere in a condition — always optional, purely for grouping or
readability.

### `if` / `else` / `else if`

```
if condition {
    // ...
} else if otherCondition {
    // ...
} else {
    // ...
}
```
Note: it's **`else if`**, two separate words — **not** `elif`.

### `while`

```
while condition {
    // ...
}
```

`_i` is automatically available inside a `while` loop body — an implicit counter that increments by
one each time through the loop, starting at 0. This mirrors exactly what a programmer would
otherwise have to hand-roll:

```
// what _i replaces, if the programmer had to do it manually:
temp int i = 0;
while condition {
    // ... use i ...
    i++;
}
tempdelete i;
```

### `for` (C-style, three clauses)

```
for init; condition; increment {
    // ...
}
```

No parentheses. The parser distinguishes the three clauses because each is grammatically
self-terminating (the init is a statement ending in `;`, the condition and increment are expressions
that naturally stop where a `{` — which can never legally appear inside an expression — begins).

**If any of the three clauses is intentionally omitted, use the `none` keyword** in its place rather
than leaving it visually blank (unlike C++'s `for(;;)` convention):

```
for none; condition; increment {  }   // no init
for init; none; increment {  }        // no condition (infinite unless broken out of)
for init; condition; none {  }        // no increment
```

`_i` is also available inside a C-style `for` loop, same semantics as in `while` (increments once
per iteration).

### `for` (range-based / "for-each")

```
for var : someList {
    // ...
}
// or, with an explicit type instead of `var`:
for int : someList {
    // ...
}
```

`_i` is available here too (0-based index of the current element).

### `times`

A dedicated "repeat N times" loop — something C++ notably lacks a clean built-in way to express.

```
5 times {
    console.write("Hey");
}
// prints "Hey" five times
```

Also works with a variable holding the count:
```
temp int x = 5;
x times {
    console.write("Hey");
}
tempdelete x;
```

`_i` is available inside `times` (0-based index of the current repetition).

### `break` / `continue`

Same as C++. Work inside **all** loop types, including `times`.

### `skip` / `skip here` — labeled multi-level break

Unlike `break` (which only exits the single innermost loop) or `return` (which exits a function),
`skip` can exit **any number of nested `{ }` blocks at once** — including `if` blocks, which
otherwise have no equivalent of `break`/`return` to escape out of early. This is effectively a
**named, multi-level break** — precedented in other languages as "labeled break" (Java, Kotlin) or
a restricted, safe form of `goto` (Go, C).

```
skip here point;    // marks a target location, named `point`
skip point;          // jumps to the `skip here point;` with that name
```

- **`skip here point;`** marks a label named `point`. There can be **at most one** `skip here` with
  a given name **per function** — declaring the same name twice in one function is a compile error.
  (Different functions can reuse the same label name freely — labels are scoped to the function
  they're declared in, same as C++'s own labels, which is what this transpiles to.)
- **`skip point;`** jumps to the `skip here point;` with that name. **Multiple `skip point;`
  statements can target the same label** — there's no one-jump-per-label restriction.
- Can jump **forward or backward** relative to where it's written in the source, and can exit
  **multiple levels of nested `{ }` at once** (e.g. escape from deep inside several nested `if`
  blocks straight to a `skip here` near the end of a function, in one jump).
- **Used inside a loop, `skip` behaves like `break`** — leaving the loop's block is just leaving a
  scope, nothing loop-specific happens differently.
- Any `temp` variables that go out of scope as part of the jump are cleaned up normally (this falls
  out for free from the underlying C++ `goto`/RAII interaction — see
  `01-compiler-architecture.md` for the codegen mapping).

**Legality rule (compile-time enforced):** a `skip point;` may only target a `skip here point;`
that sits at the **same block, or an ancestor (enclosing) block**, relative to the jump. Put another
way: walking outward from the `skip point;` statement through its enclosing blocks toward the
function's top level, the matching `skip here point;` must be found somewhere along that walk (or
in the same block the jump itself is in). If the label instead sits inside some *other*, unrelated
or *deeper* block relative to the jump, this is a compile error — such a jump would require entering
a scope the jump was never inside, which is unsafe (you could end up "inside" a block without any of
its variables having been declared/initialized) and is exactly the one case real `goto` also forbids.
This rule is what makes arbitrary forward/backward, multi-level jumping safe to allow at all.

```
func example(): none
{
    if true {
        if true {
            skip done;   // legal: `done`'s block (the function body) is an ancestor
                          // of this deeply-nested if-block
        }
    }

    console.write("this line is skipped over\n");

    skip here done;
    console.write("execution resumes here\n");
}
```

```
func brokenExample(): none
{
    if true {
        skip here target;   // `target` is declared INSIDE this nested if-block
    }

    skip target;   // ERROR: this jump is in the outer/top-level block, which is NOT
                     // nested inside the if-block that contains `target`. Reaching
                     // `target` would mean jumping INTO the if-block's scope from
                     // outside it -- exactly the unsafe "enter a never-entered scope"
                     // case this rule exists to forbid.
}
```

### `switch`

```
switch x {
    case 1 {
        // ...
    }
    case 2 {
        // ...
    }
    default {
        // ...
    }
}
```

No parentheses around `x`. **Crucially, this does NOT fall through to the next case like C++'s
switch does** — each `case` behaves like a chained `else if` block: exactly one branch executes, and
there is no need (and no ability) to write a `break` at the end of each case to prevent fallthrough.
`default` is a genuine keyword (not a wildcard pattern like Python's `_`), and behaves like a final

**`switch` works on any type, not just integers** — `x` and each `case` value can be `string`,
`bool`, `double`, or anything else `==` works on, not just `int`/enum-like values the way raw C++'s
native `switch` restricts you to. This works cleanly (and doesn't need any special codegen beyond
what's already necessary) precisely *because* `switch` was already going to be translated into a
chained `if`/`else if` sequence using `==` comparisons in the generated C++ — not C++'s own native
`switch` statement, which really is int/enum-only and couldn't have supported this. Since the
underlying mechanism was already an equality-comparison chain (needed anyway to get the
no-fallthrough behavior above), extending it to any `==`-comparable type costs nothing extra:

```
string command = console.input();
switch command {
    case "start" { console.write("Starting...\n"); }
    case "stop" { console.write("Stopping...\n"); }
    default { console.write("Unknown command\n"); }
}
```
`else`.

---

## 6. Functions

```
func name(paramType paramName, ...): returnType {
    // ...
}
```

- **Return type is always required to be written explicitly, even for a function that returns
  nothing** — use `none` as the return type in that case.
- **Default parameter values are allowed:**
  ```
  func greet(string name = "World"): none {
      console.write(fstring("Hello, {name}!\n"));
  }
  ```
- **Function overloading requires the explicit `overload` keyword.** The first declaration of a
  function name is written normally (`func`). Any *additional* declaration reusing that same name
  must be written as `func overload name(...)` to be legal — omitting `overload` on a repeated name
  is a fatal duplicate-definition error, **regardless of whether the parameters differ.** This
  applies across files, not just within one file — overloading is now a purely syntactic signal
  (the `overload` keyword), not something inferred from file origin. See the Program Structure
  section below for how this interacts with the general cross-file name-collision rule.
  ```
  func someFunc(int a): int { }
  func overload someFunc(double b): int { }   // valid — explicit overload
  func someFunc(string c): int { }            // ERROR: duplicate definition, missing `overload`
  ```
  **Even with `overload`, an identical signature to an existing declaration is still an error** —
  `overload` permits a *different* signature sharing a name, it does not allow two functions with
  the exact same signature to coexist (there would be no way to know which one a call site meant).
- **All parameters are pass-by-value in v1.** There is no pass-by-reference or pointer-parameter
  option yet — this is planned for v2 alongside the rest of the memory-management features.
- **Recursion works automatically** — a function can call itself, no special syntax required.
- **Return type qualifier restriction:** a function's declared return type can never carry `const`,
  `global`, or `temp`. E.g. `func myFunc(): const int { }` is **invalid** — a compile error. A
  function only ever returns a plain, unqualified value; any qualifiers are applied only at the
  point where the caller assigns the returned value to a variable:
  ```
  func myFunc(): int { return 42; }

  temp global const int myVar = myFunc();   // qualifiers applied HERE, not on myFunc's signature
  ```

---

## 7. Program structure: `main()` and imports

- **No executable statements are allowed outside of a function body, anywhere, in any file.**
  Every single Az file — whether it's the entry file or something pulled in via `using` — consists
  entirely of function declarations (plus `using` statements at the top). There is no "loose
  top-level script code" the way Python allows.
- **Exactly one `main()` per compiled program.**
  - The entry file (the one the compiler is actually invoked on) **must** contain a `main()`. If it
    doesn't, this is a fatal compile error.
  - Any file brought in via `using` **must not** contain a `main()`. If it does, this is a fatal
    compile error.
- Program execution begins at `main()`, exactly as in C++. (Unlike the language's very first draft
  concept, there is no special "code before `main` just runs" behavior — that idea was explicitly
  reconsidered and dropped in favor of this stricter, more predictable model.)

### Imports (`using`)

Two import forms, deliberately mirroring Python's `import x` vs. `from x import y`:

**1. Whole-file import** — brings in every function from the target file, but each must be called
with the file's name as a prefix:
```
using tools.az;

int result = tools.randomInt();
```

**2. Selective import** — brings in one specific function, callable without any prefix:
```
using tools.randomInt;

int result = randomInt();
```

**3. Comma-separated combinations of both forms are allowed in one statement:**
```
using tools.az, calc.factorial, calc.div;
```

**There is no wildcard import** (no `using tools.*` equivalent) — every whole-file or per-function
import must be spelled out as shown above. This was an explicit, deliberate choice to avoid
namespace ambiguity, tying directly into the collision rule below.

**Built-in libraries (`console`, `math`, `random`, `time`, `file`) use this exact same `using`
syntax** — a program must write `using console;` (etc.) before using any of `console`'s functions,
exactly as shown in the very first example program in `00-overview.md`. However, **built-ins are
not actual `.az` files on disk** — they're recognized and implemented directly inside the
transpiler itself. `using console;` does not trigger a file lookup the way `using tools.az;` does;
the compiler simply recognizes `console` (and the other built-in library names) as special,
always-available identifiers. There's no risk of a built-in colliding with a missing file on the
user's filesystem, and no path resolution happens for them at all.

### Import path resolution (for actual `.az` files, not built-ins)

- **Relative paths are resolved relative to the file containing the `using` statement itself** —
  not the directory the compiler was run from, and not always the entry file's directory. This
  means a file's imports behave the same regardless of how deeply it's been pulled into a larger
  program via someone else's `using` chain.
- **Absolute paths are also accepted** (expected to be used rarely, but explicitly supported).

### Name collision rule

There are no separate namespaces in Az — a `library.` prefix is purely a *call-site convenience*, it
does not create an isolated namespace that protects against name clashes.

**Rule:** any function declaration that reuses a name already in scope is a fatal collision **unless
it is explicitly marked `func overload name(...)`** — see the Functions section above for the full
`overload` keyword rule, including that even an `overload`-marked declaration is still an error if
its signature exactly duplicates an existing one. This applies regardless of whether a `library.`
prefix would be required to call either function, and applies across files (an imported file's
function and a locally-defined function can validly overload each other, as long as the later
declaration uses `overload`).

Scope of the check (which functions are even "in scope" to check against):
- If a whole file is imported (`using tools.az;`), **every** function in that file is checked for
  collisions against the entry file and everything else currently in scope.
- If only specific functions are imported (`using calc.factorial;`), **only** those named functions
  are checked. Any other function that happens to live in `calc.az` but wasn't imported is invisible
  to the compiler for this purpose and cannot cause a collision, because it was never brought into
  scope in the first place.

### Forward references

A function may be called anywhere in the program, regardless of whether its textual definition
appears before or after the call site in the source — this works because the compiler scans and
registers every function signature across the whole program before generating any code (see the
Symbol Table Pre-Pass in `01-compiler-architecture.md` section 5).

---

## 8. Strings

### Escaping

`\` is a **universal escape character** — `\` followed by any character means "the literal version
of that character," rather than a fixed, memorized whitelist of special sequences. This covers the
familiar cases (`\n` newline, `\t` tab, `\\` literal backslash, `\"` literal quote) as one instance
of a single general rule, and the same rule is what allows literal `{` and `}` to appear inside an
`fstring(...)` template without being parsed as interpolation syntax (`\{`, `\}`).

### `fstring(...)`

```
fstring("You entered: {input}\n")
```

- Anything inside `{}` can be **any valid Az expression** — a bare variable, arithmetic, a function
  call, whatever — as long as it evaluates to *something*. It is not restricted to bare identifiers.
- The value of a `{}` expression is **automatically converted to a string** for insertion into the
  result, using the same conversion logic as the `string(...)` built-in (see below) — regardless of
  its actual type.
- **If the expression's type is not already `string` or `char`, the compiler must emit a warning**
  (not an error) at that call site, flagging that the value is being auto-converted. This exists
  specifically to catch accidental cases like dumping an entire `list` into a message meant to hold
  a single value.

### `string(...)` built-in conversion function

A general-purpose, overloaded conversion function: takes a value of essentially any type (numbers,
booleans, lists, dicts, whatever else comes up) and returns its `string` representation. This is the
same underlying mechanism `fstring` uses internally for its `{}` segments — implement it once in the
runtime and have both features route through it (see `01-compiler-architecture.md` section 7.2).

---

## 9. Reserved / implicit identifiers

`_i` (used inside `while`, `for`, `times` loop bodies) is not a keyword exactly, but is an
implicitly-defined identifier inside those loop bodies and should be treated as reserved within that
context to avoid the user accidentally shadowing/conflicting with it. See
`07-keyword-reference.md` for the full reserved-word list.

---

## 10. Explicitly out of scope for v1 (do not implement, do not silently add)

- Pointers, references, `owned`/`shared`/`weak` smart-pointer-style keywords — **planned for v2**
- User-facing generics/templates — v1 built-ins may use real C++ templates *internally*, but this is
  never exposed as a language feature to the Az programmer
- Bitwise operators (`& | ^ ~ << >>`) — deferred, possibly to v1.1 or v2
- OOP (classes, inheritance, etc.) — planned for a future version, specifically enabled by the
  choice of C++ as the transpile target (see `00-overview.md`)
- Callbacks / passing functions as values — came up specifically regarding the `time` library's
  countdown timer (whether it could trigger a callback at zero); explicitly deferred, v1 uses polling
  only (`time.isDoneTimer(handle): bool`, checked manually)
