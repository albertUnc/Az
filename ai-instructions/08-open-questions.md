# Open Questions / Not Yet Decided

This file exists so the implementing assistant (or the language designer, in a follow-up session)
knows exactly what's still unresolved, rather than having gaps get silently filled in with
assumptions during implementation. **If you're an AI assistant implementing this spec: do not
silently invent answers to anything below. Either ask the user, or implement a clearly-labeled,
easily-replaceable placeholder/stub and flag it prominently in code comments and in your response.**

---

## 1. ~~`file` built-in library~~ — RESOLVED

Full spec in `06-library-file.md`, including filesystem utility functions (`exists`,
`createDirectories`, `parentPath`, `extension`, `rename`, `delete`, `move`, `copy`, `encrypt`/
`decrypt`). `parentPath`'s no-parent case and `rename`'s mismatch case route through the runtime
diagnostics system (`12-runtime-diagnostics.md`) as `ERROR`s. The constructor's own failure case is
narrower than earlier drafts of this document described: opening a path whose **file** doesn't
exist, in *any* mode (including `'r'`), now **auto-creates it** rather than erroring — the only
remaining fatal case is a missing **parent directory**, which is still an `ERROR`. **Binary file
support and directory listing remain intentionally out of scope for v1** — left out on purpose for
now, not oversights.

## 2. `system` / `env` built-in library

Explicitly deferred to v1.1 by the designer — environment variables, OS info, command-line
arguments passed to the compiled program. Not designed at all yet, not even a rough sketch. Do not
attempt to implement this for v1.0.

## 3. ~~CLI design~~ — RESOLVED (for the current two-step toolchain)

Full spec in `10-cli.md`: `az-transpiler <sourceFile.az>` plus optional `-o <outputFile.cpp>`.
Minimal by design, since transpile+compile are still two separate steps (unification is a planned
follow-up, not yet built — see `01-compiler-architecture.md` section 0). Exit codes confirmed:
`0` success (warnings included), `1` + message on any `ERROR`. One small detail remains open: the
eventual unified command's name (`az` used illustratively elsewhere in the spec set, not confirmed).

## 4. ~~Compiler implementation language~~ — RESOLVED

**Confirmed: the Az compiler/transpiler itself is written in C++**, built into a standalone
executable (e.g. `az-transpiler.exe`), invoked as `az-transpiler HelloWorld.az` to produce a
generated `.cpp` file. `g++`/`clang++` is then invoked separately (for now) to finish the build.
**Planned follow-up:** unify transpile + compile into a single command so the user only runs one
tool. See `01-compiler-architecture.md` section 0 for full detail.

## 5. ~~Import path resolution~~ — RESOLVED

**Confirmed:** relative `using` paths resolve relative to the file containing the `using` statement
itself (not the CWD, not always the entry file's directory). Absolute paths are also accepted.
Built-in libraries (`console`, `math`, `random`, `time`, `file`) are **not real files at all** —
they're recognized directly by the transpiler, no path resolution happens for them. See
`02-language-spec.md` section 7.

## 6. ~~Overload sets vs. cross-file collision rule interaction~~ — RESOLVED

**Confirmed: overloading now requires an explicit `overload` keyword.** The first declaration of a
function name is plain `func`; any additional declaration reusing that name must be written
`func overload name(...)` to be legal, regardless of whether the parameters differ, and this works
across files (not file-scoped). An `overload`-marked declaration with a signature identical to an
existing one is still an error. See `02-language-spec.md` sections 6 and 7 (Functions, and Name
collision rule) for the full updated rule. This supersedes the earlier "same name anywhere =
automatic collision, no exceptions" version of the rule.

## 7. ~~Handle misuse behavior (`time` library)~~ — RESOLVED

**Confirmed:** using a stopwatch/timer handle after it's been stopped (e.g. calling
`elapsedStopwatch` after `stopStopwatch`) is **not an error** — it returns the frozen value the
handle held at the moment it was stopped. See `05-library-time.md`.

## 8. ~~`math` function list completeness~~ — RESOLVED

`log`/`log2`/`log10` and `factorial` (with negative-input and overflow-past-`20!` both handled as
runtime `ERROR`s, per `12-runtime-diagnostics.md`) are now confirmed and added — see
`04-library-math-random.md`. **Trigonometric functions (`sin`/`cos`/`tan`/etc.) are explicitly
excluded, not wanted** — this is a deliberate omission now, not an oversight.

## 9. OOP design (v2)

Explicitly out of scope for v1, and not sketched at all yet beyond "it's why C++ was chosen as the
transpile target." No syntax, no class/inheritance model, nothing — this is a placeholder for an
entirely future design conversation, not something to draft ahead of time.

## 10. Pointers / references / memory model (v2)

Same as above — explicitly deferred, not designed. One substantive note already flagged for
whenever this design work happens: the `temp`/`tempdelete`/`global` interaction rules from v1
(see `02-language-spec.md` section 3) were only proven safe *because* v1 has no
references/pointers. That safety argument must be re-derived, not assumed to still hold, once v2
introduces them.

## 11. ~~Runtime `logs` library~~ — RESOLVED

Full spec now in `11-library-logs.md`: single global "currently loaded log" slot (no handles/
variables), `logs.load(path)` (append-mode, auto-creates, writes a session-start line),
`logs.write(message, level = Info)` (auto-timestamp + auto-newline, `Info`/`Warning`/`Error` are
compiler-recognized literals like `console.input(int)`'s type argument, not general Az enums),
`logs.unload()`. `write()`-with-no-log-loaded now routes through the runtime diagnostics system
(`12-runtime-diagnostics.md`) rather than the old stderr-print placeholder. One small item remains
open: the exact wording of the session-start auto-message.

## 12. ~~General runtime warning/diagnostic mechanism~~ — RESOLVED

Full spec now in `12-runtime-diagnostics.md`: a dedicated `runtime_errors.log` file, wiped fresh on
every run of the compiled program, written next to the executable (confirmed location). Only two
severity levels — **`ERROR` and `WARNING`, deliberately no `INFO` at runtime** (informational
logging is left to the programmer via the `logs` library instead). Nothing is printed to the
terminal — file only, since a compiled Az program might not have one (GUI apps, background
services, etc.). `ERROR` terminates the program; `WARNING` logs and continues. This resolves the
placeholder behavior previously left open in `06-library-file.md` (`parentPath`, `rename`, and the
constructor's missing-parent-directory case — see item 1 above for that case's refined scope) and
`11-library-logs.md` (`write()` with no log loaded, and `load()`'s own failure case, which is a
`WARNING` rather than an `ERROR` — see `11-library-logs.md`) — all now route through this system. Only a full enumeration of every possible runtime condition and
its level remains open — not a gap in the mechanism itself, just not every future condition has
been individually decided yet.

## 13. ~~`list`/`dict` beyond basic declaration~~ — RESOLVED

**Confirmed: Python-style bracket access, for both `list` and `dict`, both reading and writing.**
Full spec now in `02-language-spec.md` section 2a: negative `list` indices work like Python
(`nums[-1]` is the last element), out-of-range `list` access (read or write) and `dict` **read** of
a missing key are both fatal runtime `ERROR`s (see `12-runtime-diagnostics.md`, and R005/R006 in
`15-diagnostics-catalog.md`) — Az has no exception-handling construct, so this is the closest
faithful translation of what an uncaught Python `IndexError`/`KeyError` would mean. `dict` **write**
to a missing key auto-inserts (no error), matching Python's dict-assignment behavior exactly — this
asymmetry between `dict` read and write is intentional, not a bug. Non-empty list/dict literals
(`[10, 20, 30]`, `{"a": 1}`) are confirmed as part of the same answer. Grammar formalized in
`13-grammar.md` section 5a; codegen (including the `az_list_index`/`az_dict_read`/`az_dict_write`
runtime helpers) in `01-compiler-architecture.md` section 7.6b.

## 14. ~~`switch` subject/case value type~~ — RESOLVED

**Confirmed: `switch` works on any type**, not just `int` — `string`, `bool`, `double`, anything
`==` supports. This costs nothing extra to implement: `switch` was already being translated into a
chained `if`/`else if` sequence using `==` comparisons in the generated C++ (not C++'s own native
`switch`, which really is int/enum-only) specifically to get the no-fallthrough behavior — so
extending it to any `==`-comparable type is a natural consequence of that choice, not a separate
feature. See `02-language-spec.md`'s `switch` section for the updated example (a `string`-based
switch).
