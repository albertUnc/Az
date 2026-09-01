# Az Transpiler — Command-Line Interface (v1)

This describes the CLI for the **current, two-step** state of the toolchain: `az-transpiler`
converts `.az` source (plus everything it pulls in via `using`) into **one single** `.cpp` file,
and the user separately invokes `g++`/`clang++` on that result. Per
`01-compiler-architecture.md` section 0, the long-term plan is to unify transpiling and compiling
into a single command — this CLI describes the transpiler-only tool as it stands today. When
unification happens, this file should be revisited (it will likely gain flags related to the final
binary's name/location, optimization level, etc., none of which are relevant yet since the tool
currently only ever produces a `.cpp` file).

---

## Basic usage

```
az-transpiler <sourceFile.az>
```

Executable name: **`az-transpiler`** (`az-transpiler.exe` on Windows). Run it directly if it's in the
current folder, or from anywhere once it's on your system's `PATH`.

- `<sourceFile.az>` — **required**, positional, exactly one. The path is resolved relative to the
  **current working directory** (wherever the command is actually run from) — same convention as
  most command-line tools (e.g. `gcc file.c` from within the project folder). This is always the
  **entry file** — the one that must contain `main()` (see `02-language-spec.md` section 7); any
  files it pulls in via `using` are found automatically, following the import-resolution rules in
  `02-language-spec.md` section 7 and `01-compiler-architecture.md` section 4a — you never list
  them on the command line yourself.
- **Output is always one single `.cpp` file**, containing the entry file's own functions plus every
  function reachable through its `using` chain, all combined together (see
  `01-compiler-architecture.md` section 7, "Combined single-file output" for exactly how this
  works, including how a file imported through more than one path only contributes its functions
  once). Default output: a `.cpp` file with the **same base name as the entry file**, written into
  the **same directory as the entry file** — e.g. `az-transpiler HelloWorld.az` (run from wherever
  `HelloWorld.az` lives) produces `HelloWorld.cpp` right next to it, even if `HelloWorld.az` itself
  pulls in several other `.az` files.
- `transpile-logs.txt` is written next to the entry `.az` file regardless of any other flags (see
  `09-transpile-logging.md`) — this is not configurable.

## Flags

### `-o <outputFile.cpp>`

**What `-o` means:** short for "output" — it's a very common convention across command-line tools
(`gcc`, `g++`, `clang`, and many others all use `-o` the same way) for "here's where you want the
result saved, instead of the tool's default naming/location." Without `-o`, `az-transpiler` picks the
output name/location automatically per the default rule above; with `-o`, you're overriding that.

```
az-transpiler HelloWorld.az -o build/HelloWorld_generated.cpp
```

If omitted, the default naming/location rule above applies.

---

## Exit codes

- **0** — transpilation succeeded (this includes the case where `WARNING`-level diagnostics were
  produced — warnings do not fail the build, per `09-transpile-logging.md`)
- **1** — transpilation failed due to at least one `ERROR`-level diagnostic. A message is also shown
  (per the normal `ERROR`-prints-to-terminal behavior already specified in
  `09-transpile-logging.md`) — this is a flat exit code, not a distinct code per failure category.

---

## Explicitly not yet part of the CLI (don't add without confirming)

Nothing beyond `<sourceFile.az>` and `-o` is specified for v1. In particular, none of the following
were requested and shouldn't be silently added:
- A verbose/`--debug` flag
- A "transpile only, don't even check X" partial-mode flag
- Any flag related to the eventual unified transpile+compile command (that command doesn't exist
  yet — this file only covers the transpiler as a standalone tool)

## Still open

- Whether the future unified transpile+compile command reuses the same `az-transpiler` executable
  or becomes a separate tool (e.g. `az`, as used illustratively elsewhere in this spec set) — not
  decided. `az-transpiler` as a name specifically emphasizes the transpile step, so once that tool
  also invokes `g++`/`clang++` itself, a rename might make more sense than keeping the same name for
  an expanded job — but this is speculation, not a confirmed direction.
