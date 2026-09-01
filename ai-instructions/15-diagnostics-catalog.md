# Diagnostics Catalog — Compile-Time and Runtime Messages

A concrete catalog of specific diagnostic messages the compiler and runtime should produce,
consolidating rules scattered across other files into one reference. This isn't meant to invent new
behavior — every entry here maps to a rule already established elsewhere (cited in each row) — it
exists so message wording/IDs stay consistent across the whole implementation instead of each
compiler stage inventing its own phrasing ad hoc.

Recommend giving each diagnostic a short stable **ID** (e.g. `E001`, `W003`) so error messages can
be referenced consistently (in tests, in documentation, in bug reports) even if the exact wording
changes later. IDs below are suggestions, not something the designer explicitly requested — feel
free to renumber, but keeping *some* stable ID scheme is good practice.

---

## Compile-time errors (`ERROR`, via `09-transpile-logging.md` — build fails)

| ID | Condition | Example message | Source |
|---|---|---|---|
| E001 | `string + <non-string>` | `cannot add 'string' and 'int' — use fstring() or string() to convert` | `02-language-spec.md` §4 |
| E002 | Function return type declared with `const`/`global`/`temp` | `function return types cannot be qualified with 'const'/'global'/'temp'` | `02-language-spec.md` §6 |
| E003 | Duplicate function name without `overload` | `'someFunc' is already defined — add 'overload' to declare an intentional overload` | `02-language-spec.md` §7 |
| E004 | `overload`-marked function has identical signature to an existing one | `'someFunc(int)' conflicts with an existing declaration of the exact same signature` | `02-language-spec.md` §7 |
| E005 | More than one `main()` in the program | `multiple main() functions found — exactly one is allowed, in the entry file` | `02-language-spec.md` §7 |
| E006 | Entry file has no `main()` | `entry file '<file>.az' has no main() function` | `02-language-spec.md` §7 |
| E007 | An imported file contains a `main()` | `imported file '<file>.az' must not contain main()` | `02-language-spec.md` §7 |
| E008 | Executable statement found outside any function body | `statements are not allowed outside of a function body` | `02-language-spec.md` §7 |
| E009 | `temp` variable escapes its declaring scope (assigned somewhere that would outlive it, if this ever becomes detectable) | `temp variable '<name>' cannot be used outside the scope it was declared in` | `02-language-spec.md` §3 |
| E010 | `file` instance method called against its opened mode | `cannot call '.write()' on a file opened in mode 'r'` (etc., per the mode/method table) | `06-library-file.md` |
| E011 | `file` mode argument is not a literal char | `file mode must be a literal 'a', 'w', or 'r' — not a variable or expression` | `06-library-file.md` |
| E012 | Wildcard import attempted (`using x.*;`) | `wildcard imports are not supported — import the whole file or specific functions by name` | `02-language-spec.md` §7 |
| E013 | `math.factorial` called with a compile-time-known negative literal | *(if statically detectable — see note below; otherwise this is R002, a runtime error instead)* | `04-library-math-random.md` |
| E014 | Syntax error (generic parse failure) | `unexpected token '<token>', expected <what was expected>` | `13-grammar.md` |
| E015 | Type mismatch in assignment/parameter/return where no implicit conversion rule applies | `cannot assign 'string' to a variable of type 'int'` | `02-language-spec.md` §2 |
| E016 | Circular `using` import detected | `circular import detected involving '<path>'` | `01-compiler-architecture.md` §4a |
| E017 | `skip here <name>` declared more than once in the same function | `'skip here <name>' is already declared in this function (at line <N>)` | `02-language-spec.md` Control Flow, `01-compiler-architecture.md` §7.6a |
| E018 | `skip <name>;` with no matching `skip here <name>;` in the same function | `'skip <name>' has no matching 'skip here <name>' in this function` | `02-language-spec.md` Control Flow, `01-compiler-architecture.md` §7.6a |
| E019 | `skip <name>;` targets a label that is not in the jump's own block or an ancestor block | `'skip <name>' would jump into a scope it was never inside` | `02-language-spec.md` Control Flow, `01-compiler-architecture.md` §7.6a |

**Note on E013:** most `math.factorial` misuse (a variable holding a negative value at runtime)
can't be caught at compile time at all — it's inherently a *runtime* condition (see R002 below).
E013 only applies in the narrow case where the compiler can prove at compile time that the argument
is negative (e.g. a literal `math.factorial(-5)` written directly) — this is a nice-to-have static
check, not a requirement, since the general case is unavoidably a runtime check.

---

## Compile-time warnings (`WARNING`, via `09-transpile-logging.md` — build still succeeds)

| ID | Condition | Example message | Source |
|---|---|---|---|
| W001 | `fstring()`'s `{}` segment is not `string`/`char` | `fstring() argument is not a string/char, will be auto-converted` | `02-language-spec.md` §8 |
| W002 | `console.write(...)` argument is not `string`/`char` | `console.write() argument is not a string/char, will be auto-converted` | `03-library-console.md` |
| W003 | `file.write(...)` argument is not `string`/`char` | `file.write() argument is not a string/char, will be auto-converted` | `06-library-file.md` |

---

## Compile-time info (`INFO`, file-only, no terminal output)

No specific `INFO` messages were explicitly dictated anywhere in the spec set. Reasonable,
low-risk defaults an implementer can add without needing further confirmation (these are genuinely
just informational, carry no behavioral weight, and match the general spirit of "summary stats" from
`09-transpile-logging.md`):

| ID | Example message |
|---|---|
| I001 | `transpiled '<file>.az' -> '<file>.cpp' (<N> lines)` |
| I002 | `build summary: <N> files transpiled, <N> warnings, <duration>ms` |

---

## Runtime errors (`ERROR`, via `12-runtime-diagnostics.md` — program terminates)

| ID | Condition | Example message | Source |
|---|---|---|---|
| R001 | `file.rename(...)` path mismatch | `file.rename: path component mismatch — target directory does not match 'target'` | `06-library-file.md` |
| R002 | `math.factorial(n)` called with `n < 0` at runtime | `math.factorial: n must be non-negative (got <n>)` | `04-library-math-random.md` |
| R003 | `math.factorial(n)` called with `n > 20` at runtime | `math.factorial: result would overflow unsigned long long for n > 20 (got <n>)` | `04-library-math-random.md` |
| R004 | `file` constructor / `loadFile(...)` — the path's **parent directory** doesn't exist (missing file itself is NOT an error — it auto-creates) | `file: cannot create '<path>' -- parent directory does not exist` | `06-library-file.md`, `12-runtime-diagnostics.md` |
| R005 | `list` index out of range (read or write, including out-of-range negative indices) | `list index out of range (index <i>, size <n>)` | `02-language-spec.md` §2a, `01-compiler-architecture.md` §7.6b |
| R006 | `dict` **read** of a missing key (write auto-inserts instead — not an error) | `dict key not found` | `02-language-spec.md` §2a, `01-compiler-architecture.md` §7.6b |

---

## Runtime warnings (`WARNING`, via `12-runtime-diagnostics.md` — program continues)

| ID | Condition | Example message | Source |
|---|---|---|---|
| RW001 | `file.parentPath(path)` — `path` has no parent | `file.parentPath: '<path>' has no parent directory, returning empty string` | `06-library-file.md` |
| RW002 | `logs.write(...)` called with no log loaded | `logs.write() called with no log loaded, message discarded` | `11-library-logs.md` |
| RW003 | `logs.load(...)` fails (target's parent directory doesn't exist) | `logs.load: cannot create '<path>' -- parent directory does not exist, logs remains unloaded` | `11-library-logs.md` |

---

## Implementation note

Recommend both the transpile-time logger (`09-transpile-logging.md`) and the runtime diagnostics
system (`12-runtime-diagnostics.md`) accept an optional ID alongside their level/location/message,
purely for traceability (e.g. so `transpile-logs.txt` entries could read
`[ERROR E003] main.az at ln12,col5 - 'someFunc' is already defined...`) — this is an enhancement to
the format already locked in those two files, not a contradiction of it; the ID is additive. Not a
hard requirement if it adds friction, but cheap to include and useful once the language has enough
users that referencing "error E003" is more useful than pasting the whole sentence.

## Explicitly not covered here

This catalog only lists diagnostics already implied by rules established elsewhere. It is **not**
a complete enumeration of every possible compiler bug/edge case (generic parse errors, internal
compiler errors, etc. will naturally arise during implementation and should get their own messages
as they're discovered) — and it does not invent behavior for anything still marked open in
`08-open-questions.md` (e.g. no ID is assigned yet for binary-file-related errors, since binary
file support itself isn't designed).
