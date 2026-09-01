# Az Programming Language — Project Overview

## What this document set is

This is a full specification for **Az**, a custom programming language, written up so an AI coding
assistant (working via API, in a tool like Continue in VS Code) can implement the compiler with
minimal ambiguity. These files were produced collaboratively in a planning conversation between the
language's designer and Claude. Every design decision below was explicitly confirmed by the designer
unless marked **OPEN / TBD**.

**Read order for implementation:**
1. `00-overview.md` (this file) — the big picture
2. `01-compiler-architecture.md` — how the compiler itself should be built, pipeline stages, project layout
3. `02-language-spec.md` — full syntax and semantics of Az
4. `03-library-console.md`, `04-library-math-random.md`, `05-library-time.md`, `06-library-file.md` — built-in library specs
5. `07-keyword-reference.md` — full reserved word list
6. `08-open-questions.md` — everything NOT yet decided; do not silently invent answers to these, ask the user or make a clearly-labeled placeholder implementation
7. `09-transpile-logging.md` — the compiler's own build-time error/warning/info logging system (separate from the runtime `logs` library)
8. `10-cli.md` — command-line usage for `az-transpiler`
9. `11-library-logs.md` — the runtime `logs` library, for compiled Az programs to log their own activity
10. `12-runtime-diagnostics.md` — the runtime's own internal error/warning system (`runtime_errors.log`), distinct from both the transpile-time logger and the `logs` library
11. `13-grammar.md` — formal EBNF grammar for the whole language
12. `14-example-programs.md` — complete, compilable example `.az` programs exercising every feature; use these as end-to-end test fixtures
13. `15-diagnostics-catalog.md` — a consolidated, ID-tagged catalog of every specific error/warning message implied elsewhere in the spec
14. `16-testing-strategy.md` — suggested test suite structure for the compiler itself

---

## What is Az?

Az is a compiled programming language that:

- Has **C++-like syntax** (curly-brace scoping, `;` statement terminators, whitespace/newlines are
  never semantically meaningful — a whole program could legally be written on one line).
- **Transpiles to C++**, then invokes an existing C++ compiler (gcc/clang) to produce the final
  native binary. Az's own compiler never generates machine code or assembly directly — it generates
  `.cpp` (and possibly `.h`) files and shells out to a real C++ toolchain to finish the job.
- Is used as a **single command that takes a source file and produces an executable**, e.g.
  conceptually: `az myprogram.az -o myprogram` (exact CLI is still open — see `08-open-questions.md`).
- File extension: **`.az`**

### Why C++ as the transpile target (design rationale, for context)

The designer wants speed (native compiled code) and a lower-level feel, but doesn't want to
hand-write an x86 codegen backend or wire up LLVM for a first version. Transpiling to C++ means:
- All of C++'s optimizer, portability, and linking is inherited for free.
- Object-oriented programming (planned for a future version) becomes much easier to implement,
  since C++ already has full OOP semantics — Az's compiler will eventually just need to translate
  Az's OOP syntax into equivalent C++ class syntax, not invent OOP from scratch.
- This is a legitimate, precedented approach — early C++ itself ("Cfront") was a transpiler to C.

### Why the name "Az"

The designer's name starts with "A." Considered and rejected: `A++` (already exists, twice, as an
unrelated educational language), `A#` (already exists, at least three times), `Ax` (already exists
on GitHub as an unrelated LLVM-based language), `A+` (a real, established APL-family language from
Morgan Stanley since 1988), `Z++` (an actively maintained unrelated compiler project exists with
this exact name). `Az` and `A&` came up clean in searches; `Az` was chosen as the final name for
readability/typability. **No further naming work needed — this is locked.**

---

## Design philosophy (important context for implementation decisions)

These aren't hard rules, but they explain *why* many of the choices in the spec were made, which
should guide judgment calls in places the spec doesn't fully cover:

1. **C++ ergonomics, minus C++'s pain points.** Az intentionally borrows C++'s type system and
   general shape, but adds quality-of-life features C++ notoriously lacks or makes verbose:
   automatic string formatting (`fstring`), a manual-lifetime variable without full pointer
   complexity (`temp`/`tempdelete`), a clean "repeat N times" loop (`times`), a built-in
   validated-input function, cross-platform terminal helpers, etc.
2. **No memory management complexity in v1.** No pointers, no references, no `shared`/`unique`/`weak`
   smart pointers, no manual heap management exposed to the user. Everything is pass-by-value.
   This is a deliberate, explicit choice to keep v1 achievable — **planned for v2**, not forgotten.
3. **No generics/templates exposed to the Az programmer in v1** — but the *compiler's own generated
   C++ code* is free to use real C++ templates internally wherever that's the natural implementation
   (see `console.input` in `03-library-console.md` for the canonical example: it looks like flat
   overloads to an Az programmer, but should be implemented as one templated function in the C++
   runtime support library).
4. **Built-ins should hide platform/annoyance complexity.** E.g. `console.clear()` must detect the
   OS and issue the right command (`cls` vs `clear`) rather than making the user handle that.
   `console.input(type)` handles its own retry-until-valid loop rather than making every Az program
   hand-roll one.
5. **Safety over cleverness, but not at the cost of ergonomics.** No implicit type coercion between
   incompatible types (e.g. `string + int` is a compile error, not silently stringified). But
   *numeric* types (`int`/`double`/`long long`) do implicitly convert in parameters and returns,
   because that's how C++ already behaves and fighting it would be unergonomic for no real safety
   benefit — see the one deliberate deviation from C++ behavior in `02-language-spec.md` regarding
   double→int rounding vs. truncation.

---

## Versioning roadmap

### v1.0 (the version this spec set primarily describes)
- Full core language: types, variables, operators, control flow, functions, imports/`main`, strings
- Built-in libraries: `console`, `math`, `random`, `time`, `file` (file library still being finalized
  — see `06-library-file.md`)
- No OOP, no pointers/references, no generics exposed to the user, no bitwise operators

### v1.1.x (planned, not yet designed in detail)
- `system`/`env` built-in library (environment variables, OS info, command-line args to the
  compiled program)
- Possibly bitwise operators if a need arises

### v2 (planned, not yet designed)
- Pointers, references, and smart-pointer-style keywords (`owned`, `shared`, `weak` were discussed
  and explicitly deferred, not rejected)
- **Important caution already flagged by the designer:** the `temp`/`tempdelete` system's safety in
  v1 relies entirely on the fact that everything is pass-by-value (a `temp` variable can never be
  referenced from outside the scope it was declared in, because nothing *can* hold a reference to
  anything in v1 — only copies). Once pointers/references exist in v2, this invariant breaks, and
  the rule "a temp must be deleted in the same scope it was declared" will need to be re-examined
  for potential use-after-free scenarios. **Do not port the v1 temp/global logic forward into v2
  without re-deriving its safety.**
- Bitwise operators (`& | ^ ~ << >>`) if not already added in v1.1
- OOP (classes, inheritance, etc.) — chosen to transpile to real C++ classes

---

## A representative Az program (from the design conversation)

This example predates a couple of later decisions (it uses `|` as a terminator, which was replaced
by `;`) but is preserved here because it's the clearest illustration of the language's intended feel.
**Treat `;` as the terminator, not `|`, everywhere below and in all real Az code.**

```
//first-idea.az
/*
multi
line
comment
*/
using console;

console.write("Hello world!\n");

func exampleVoid(int value, bool print): none
{
    if print {console.write(value + "\n");}
    if print {console.write(fstring("{value}\n"));}
}

func exampleGetInt(): int
{
    console.write("Enter integer: ");
    return console.input(int);
}

func main(): none
{
    int input = exampleGetInt();
    console.write(fstring("You entered: {input}\n"));
}
```

Note: the final design requires exactly one `main()` function (see `02-language-spec.md`), and no
executable statements are allowed outside of function bodies anywhere in the program — this is a bit
stricter than the very first draft implied, and is a deliberate, confirmed decision.
