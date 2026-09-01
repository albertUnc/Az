# Testing Strategy for the Az Compiler

Suggested approach for testing the transpiler as it's built. Not dictated by the original design
conversation — this is implementation guidance to help the build actually be verifiable at each
stage, structured to match the pipeline in `01-compiler-architecture.md`.

## 1. Per-stage unit tests

Test each pipeline stage in isolation before testing them together:

- **Lexer tests**: feed small source snippets, assert the exact token sequence produced. Specifically
  cover the tricky lexical rules: the universal `\` escape (`"\{"`, `"\}"`, `"\\"`, `"\n"` all
  producing correct literal characters), block comments spanning multiple lines, `long long` as two
  tokens recognized as one type, string vs. char literal delimiters never being interchangeable.
- **Parser tests**: feed token sequences (or source directly, through the lexer), assert the
  resulting AST shape. Cover every statement/expression form in `13-grammar.md`, especially the
  trickier ones: C-style `for` with `none` clauses, range-based `for`, `times` with both a literal
  and a variable, `switch` with no fallthrough, ternary vs. function-return-type `:` disambiguation,
  `fstring()`'s nested expression parsing.
- **Symbol table / collision tests**: construct multi-file import scenarios and assert collision
  detection fires (or doesn't) correctly per the `overload`-keyword rule in
  `02-language-spec.md` §7 — including the specific edge cases: same name without `overload`
  (must error), same name with `overload` and different signature (must succeed, including across
  files), same name with `overload` and *identical* signature (must still error).
- **Semantic analysis tests**: exactly-one-`main()` enforcement, no-`main()`-in-imports enforcement,
  no-top-level-statements enforcement, the `double`→`int` rounding rule, `string + non-string`
  rejection.
- **Codegen tests**: for a representative sample of Az constructs, assert the generated C++ is both
  syntactically valid *and* semantically correct — not just "compiles," but "does the right thing."
  The cleanest way to check this without manually inspecting generated C++ for every case: compile
  the generated C++ with `g++`/`clang++` and run it, then assert on the *program's actual output* —
  this doubles as an end-to-end test (see next section) and avoids brittleness from asserting on
  exact generated C++ text, which is likely to change as the codegen implementation evolves.

## 2. End-to-end tests (recommended primary test strategy)

For every example program in `14-example-programs.md` (and more added as the language grows):
1. Transpile it (`az-transpiler example.az`)
2. Compile the result with `g++`/`clang++`
3. Run the resulting executable
4. Assert on **stdout/stderr and exit code**, not on the intermediate `.cpp`

This is more robust than asserting on generated C++ text directly, since it tests what actually
matters (does the compiled program behave correctly) without over-constraining *how* the compiler
gets there internally.

### Suggested test harness shape (Python, since it's likely the easiest scripting glue regardless of
what the compiler itself is written in — this is just test tooling, not part of the shipped product)

```python
import subprocess
import sys

def run_az_test(source_file: str, expected_stdout: str, expected_exit_code: int = 0):
    cpp_file = source_file.replace(".az", ".cpp")
    binary_file = source_file.replace(".az", "")

    # 1. Transpile
    result = subprocess.run(
        ["az-transpiler", source_file],
        capture_output=True, text=True
    )
    assert result.returncode == 0, f"Transpile failed: {result.stderr}"

    # 2. Compile the generated C++
    result = subprocess.run(
        ["g++", cpp_file, "-o", binary_file, "-I", "runtime/"],
        capture_output=True, text=True
    )
    assert result.returncode == 0, f"g++ failed: {result.stderr}"

    # 3. Run and check output
    result = subprocess.run([f"./{binary_file}"], capture_output=True, text=True)
    assert result.returncode == expected_exit_code
    assert result.stdout == expected_stdout
```

## 3. Negative tests (things that should fail, and fail correctly)

Just as important as positive tests — for every entry in `15-diagnostics-catalog.md`, write a small
`.az` snippet that should trigger it, and assert:
- The correct exit code (`1` for `ERROR`, per `10-cli.md`)
- The diagnostic actually appears in `transpile-logs.txt` (or `runtime_errors.log` for runtime
  diagnostics) with roughly the right level/message
- For `ERROR`-level: no output binary is produced
- For `WARNING`-level: the build *does* still succeed and *does* still produce a working binary

Example negative test cases to include, one per row of the diagnostics catalog:
```
// should trigger E001
func main(): none {
    string s = "count: " + 5;
}
```
```
// should trigger E003
func dup(int x): int { return x; }
func dup(int x): int { return x; }   // no `overload` -> error
```
```
// should trigger E004
func dup(int x): int { return x; }
func overload dup(int x): int { return x; }   // `overload` but identical signature -> still error
```
```
// should trigger E018 (no matching skip here)
func main(): none {
    skip nowhere;
}
```
```
// should trigger E019 (jump into a never-entered scope)
func main(): none {
    if true {
        skip here inner;
    }
    skip inner;   // jump's block is NOT an ancestor of inner's declaring block
}
```
```
// should trigger R005 (list index out of range) at RUNTIME, not compile time
func main(): none {
    list<int> nums = [1, 2, 3];
    int x = nums[10];   // compiles fine; fails when actually run
}
```
```
// should trigger R006 (dict read of a missing key) at RUNTIME -- and a companion
// positive test confirming dict WRITE to a missing key does NOT error:
func main(): none {
    dict<string, int> d = {"a": 1};
    int x = d["missing"];   // should terminate with R006
}
```
```
// positive test: dict write to a missing key must succeed, not error
func main(): none {
    dict<string, int> d = {};
    d["new"] = 5;   // must NOT trigger R006 -- auto-insert on write
}
```

## 4. Regression tests

As bugs get found and fixed during development, add the fixing test case permanently to the suite
rather than only manually verifying the fix once — standard practice, called out here because a
transpiler is exactly the kind of project where subtle codegen regressions (e.g. accidentally
reverting the rounding-not-truncating rule) are easy to silently reintroduce.

## 5. Suggested test directory layout

Building on the structure proposed in `01-compiler-architecture.md` §9:

```
tests/
├── lexer_tests/
│   └── (small source snippets + expected token sequences)
├── parser_tests/
│   └── (small source snippets + expected AST shapes)
├── semantic_tests/
│   └── (positive AND negative cases per 15-diagnostics-catalog.md)
├── codegen_tests/
│   └── (source + expected program behavior, run through the full pipeline)
├── e2e/
│   └── (the programs from 14-example-programs.md, or symlinks/copies of them)
└── negative/
    └── (one file per diagnostics-catalog entry, organized by ID)
```
