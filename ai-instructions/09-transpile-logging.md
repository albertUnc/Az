# Transpile-Time Logging System

This covers **only** the Az transpiler's own build-time diagnostics — errors, warnings, and info
messages produced while turning `.az` source into `.cpp`. This is a completely separate system from
the planned `logs` library (not yet specified — that will be a *runtime* facility for compiled Az
programs to log things while they themselves are running, used by the end program, not by the
transpiler). Do not conflate the two when implementing.

---

## Severity levels

| Level | Meaning | Compilation outcome |
|---|---|---|
| `ERROR` | Transpilation cannot complete — e.g. bad syntax, `string + int`, calling `.write()` on a `'r'`-mode file, a name collision, missing `main()`, etc. | Build fails, no output binary produced |
| `WARNING` | Transpilation succeeds, but something risky was flagged — e.g. a non-`string`/`char` value passed to `fstring()`'s `{}` or to `console.write()`/`file.write()` (see `02-language-spec.md` section 8 and `03-library-console.md`) | Build still succeeds |
| `INFO` | Purely informational, no problem — e.g. summary stats like files transpiled, line counts, build duration | Build still succeeds |

Every diagnostic the transpiler produces anywhere in the whole pipeline (lexer, parser, symbol
table pre-pass, semantic analysis, codegen) should route through this one shared logging system,
tagged with the appropriate level — there shouldn't be a separate, ad hoc way of reporting problems
in different compiler stages.

---

## Output destinations

- **`ERROR` and `WARNING`** — printed to the terminal **and** written to the log file.
- **`INFO`** — written to the log file **only**, not printed to the terminal (keeps the terminal
  output focused on things that actually need attention; full detail is still available in the
  file for anyone who wants it).

## Log file behavior

- File name: **`transpile-logs.txt`**
- Location: written next to the **target `.az` file being compiled** (i.e. in the same directory as
  the source file passed to `az-transpiler`).
- **Wiped and recreated fresh on every transpile run** — never appended across runs. Each run's
  `transpile-logs.txt` reflects only that run's diagnostics, nothing from previous runs.

---

## Log entry format

```
[LEVEL] filename.az at ln<line>,col<column> - message
```

Concrete example, exactly as specified:
```
[WARNING] HelloWorld.az at ln14,col5 - fstring() argument is not a string/char, auto-converting
```

- `LEVEL` is one of `ERROR`, `WARNING`, `INFO`
- `filename.az` — the source file the diagnostic pertains to (relevant since a compile can span
  multiple files via `using` — each entry should point at whichever file actually contains the
  issue, not always the entry file)
- `ln<N>,col<N>` — 1-based line and column of the relevant location in that file
- Message — free text describing the issue

`INFO`-level entries that don't correspond to a specific source location (e.g. an end-of-run
summary like "transpiled 4 files, 812 lines total") don't need a `ln/col` — use whatever format
reads naturally for that case, this exact template is primarily for location-specific diagnostics
(which covers essentially all `ERROR`/`WARNING` entries, and possibly some `INFO` ones too).

---

## Implementation note

Recommend implementing this as a single, small logging module in the transpiler
(e.g. `az_log.cpp`/`.hpp`, since the transpiler itself is written in C++ per
`01-compiler-architecture.md` section 0) with one function like:

```cpp
void log(LogLevel level, const std::string& file, int line, int col, const std::string& message);
```

that every other stage of the compiler calls into, rather than each stage independently deciding how
to print/format/route diagnostics. At the end of a run, this module is also what handles wiping and
writing `transpile-logs.txt`, and printing the accumulated `ERROR`/`WARNING` entries to the terminal.

## Still open

- Whether a build that produced only `WARNING`s (no `ERROR`s) should still exit with a success status
  code, or something distinguishable from a fully clean build (relevant if this compiler is ever
  used in scripts/CI). Not discussed — success-with-warnings is the conventional default across most
  compilers, but not yet explicitly confirmed.
- Whether `INFO`-level summary stats (files transpiled, line counts, duration, etc.) have a required
  minimum content, or this is left to the implementer's judgment. Not discussed in detail.
