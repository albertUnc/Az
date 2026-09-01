# Built-in Library: `file`

Import with `using file;`, per standard import rules.

## Design note: `file` is a compiler-recognized built-in type, not a preview of user-facing OOP

Unlike everything else in the language so far, `file` is used with object-style syntax — you
declare a variable of type `file` and call methods on it with `.` (`.write()`, `.getline()`, etc.).
**This does not mean OOP/classes exist in v1.** `file` is a special built-in type the compiler
understands natively (in the same category as the `time` stopwatch/timer handles — see
`05-library-time.md` — though `file` additionally has real dot-method call syntax, which handles
didn't). Under the hood, the generated C++ uses real `std::fstream`/`std::ifstream`/`std::ofstream`
objects; Az just exposes a simpler, unified surface over them. This is not evidence that general
user-definable classes are available — they aren't, until the planned future OOP version.

Modeled loosely on C++'s `fstream` (not Python's file-handling idioms) — matches the rest of the
language's general "C++ shape, minus the annoying parts" philosophy.

---

## Opening a file

Two equivalent forms — a constructor-style declaration, and a function-style alternative that does
the same thing:

```
file idkName("path", 'a');
// exactly equivalent to:
file idkName = loadFile("path", 'a');
```

Both take the same two arguments:
1. **path** — `string`, the file path
2. **mode** — a single `char`: `'a'` (append), `'w'` (write — clears/truncates the file first),
   or `'r'` (read)

**A file must be opened in exactly one mode.** There is no multi-mode file (e.g. simultaneously
read+write) in v1.

**The mode argument must be a literal `char` at the point the file is opened** (`'a'`, `'w'`, or
`'r'` written directly — not a variable or expression that merely evaluates to one at runtime).
This is required so the compiler can statically know, for every `file`-typed variable, which
methods are legal to call on it — which is what makes the compile-time mode-enforcement rule below
actually enforceable.

---

## Mode enforcement — compile-time error, not runtime

Calling a method that doesn't match the file's opened mode is a **compile-time error**, e.g.:

```
file f("log.txt", 'r');
f.write("hello");   // COMPILE ERROR: calling .write() on a file opened in mode 'r' is not possible
```

Method-to-mode legality:

| Method | Legal in mode(s) |
|---|---|
| `.write(value)` | `'w'`, `'a'` |
| `.getline(): string` | `'r'` |
| `.read(): string` | `'r'` |
| `.eof(): bool` | `'r'` |
| `.close()` | any mode |

---

## Writing

### `.write(value)`

Same semantics as `console.write` — accepts **any type**, auto-converts to string, emits the same
compile-time warning if the value isn't already `string`/`char`.

```
file logFile("log.txt", 'a');
logFile.write("Something\n");
logFile.write(42);   // valid, auto-converted, triggers the usual non-string warning
```

Only legal on files opened in `'w'` or `'a'` mode.

---

## Reading

Only legal on files opened in `'r'` mode. Reading never removes/erases data from the file — it only
advances the file's internal read position.

### `.getline(): string`

Returns the current line and advances the read position past it. Intended to be used in a loop
alongside `.eof()`:

```
file f("data.txt", 'r');
while !f.eof() {
    string line = f.getline();
    // ... do something with line ...
}
f.close();
```

### `.read(): string`

Returns **everything left to read from the file, from the current read position to the end** — not
necessarily the whole file from the start. If nothing has been read yet, this is the same as "the
whole file." If some lines have already been consumed via `.getline()` first, `.read()` returns only
what remains after that point.

### `.eof(): bool`

`true` once the read position has reached the end of the file (no more data left to read). Intended
as the loop-continuation check for `.getline()`, as shown above. (An earlier draft of this design
considered treating an empty string as an implicit "false" in a `while` condition, but that was
rejected in favor of this explicit check — it would have been the only place in Az where a `string`
behaves as a boolean, breaking the language's general "no implicit cross-type behavior" rule seen
elsewhere, e.g. `string + int` being a hard compile error.)

---

## Closing

### `.close()`

Manually closes the file. Legal regardless of mode.

**Files also auto-close** when their variable goes out of scope, or when explicitly cleaned up via
`tempdelete` (if declared `temp`) — matching how every other variable's lifetime already works in
Az (see `02-language-spec.md` section 3). `.close()` exists for when you want to release the file
early, without waiting for scope exit — the same relationship `tempdelete` has to normal scope-based
cleanup elsewhere in the language.

Implementation note: since `file` already wraps a real C++ stream object, this auto-close behavior
is likely close to "free" — `std::fstream`'s own destructor already closes the underlying file
handle on destruction, so the generated C++ may not need much beyond normal RAII/scope-based
destruction, same idea as the `AzTemp` RAII wrapper described in
`01-compiler-architecture.md` section 7.3.

## Design note: `file` is both a type and a library namespace — intentional, not a conflict

`file` plays two roles, same way other built-ins do but combined here for the first time:
- As a **type**, for declaring file-handle variables: `file f("path", 'r');`
- As a **library namespace**, for the free-standing filesystem utility functions below:
  `file.exists("path")`, `file.delete("path")`, etc. — these are not methods on a `file` instance,
  they're static/library-level functions, called the same way `console.write(...)` or
  `math.sqrt(...)` are. Context disambiguates: a `file` appearing in a variable declaration position
  is the type; a `file.functionName(...)` call is the library.

---

**Opening a file that doesn't exist yet, in ANY mode (including `'r'`), creates it.** This applies
uniformly across `'r'`, `'w'`, and `'a'` — matching the combined spirit of C++'s `fstream` and
`std::filesystem` (auto-creating on open is normal `fstream` behavior for write modes; Az extends
the same courtesy to `'r'` mode too, rather than erroring). A freshly auto-created file is empty, so
reading from it (`.getline()`/`.read()`) immediately reports `.eof(): true` — there's no data, but
no error either.

**What `file` will *not* create: missing parent directories.** If the path's containing directory
doesn't exist, opening in *any* mode is a runtime **`ERROR`**, written to `runtime_errors.log` (see
`12-runtime-diagnostics.md`), and the program terminates — this is the one remaining case that
fails. Creating missing directories automatically was deliberately left out (this is exactly what
`file.createDirectories(path)` exists for — call it first if you're not sure the directory exists).

There's no silent fallback for the missing-directory case — a program that wants to avoid it should
call `file.createDirectories(...)` (for the directory) and/or `file.exists(...)` (to check) ahead of
time; the constructor itself doesn't do any implicit directory-creation or recovery on your behalf.

---

## Filesystem utility functions (library-level, not instance methods)

These operate on paths directly (as plain `string` arguments) and don't require an open `file`
instance — they're for filesystem bookkeeping (checking, creating, renaming, moving, deleting,
inspecting paths) independent of reading/writing file contents.

### `file.exists(path: string): bool`
True if a file or folder exists at `path`.

### `file.createDirectories(path: string): none`
Creates all directories in `path` that don't already exist. Silently does nothing for any part of
the path that's already created (matches the common `mkdir -p` / `std::filesystem::create_directories`
behavior).

### `file.parentPath(path: string): string`
Returns the parent directory portion of `path`. If `path` has no parent (e.g. it's already a root
or bare filename), returns an **empty string**, and also writes a `WARNING` entry to
`runtime_errors.log` (see `12-runtime-diagnostics.md`) — the program continues running normally,
the empty string is still returned as the function's result.

### `file.extension(path: string): string`
Returns the file extension **including the leading dot** — e.g. `file.extension("path/file.mp3")`
returns `".mp3"`, not `"mp3"`. This is deliberate: the dot-inclusive form is what you want when
directly reusing the result to build a new filename (e.g. concatenating it onto a different base
name), which is the motivating use case.

### `file.rename(target: string, newName: string): none`
Renames a file or folder. `newName` is normally **just the new file/folder name** (no path needed —
`target`'s existing path is reused). A full path is also accepted for `newName`, but in that case
**every path component except the final name must exactly match `target`'s existing path** — if any
earlier part of the path differs, this writes an `ERROR` entry to `runtime_errors.log` (see
`12-runtime-diagnostics.md`) and the program terminates. (Using just the bare name avoids this whole
class of mismatch and is the recommended form.) Applies to both files and folders.

### `file.delete(path: string): none`
Deletes a file, or an entire folder (recursively, including all of its contents), at `path`.

### `file.move(target: string, destination: string): none`
Moves a file or folder from `target` to `destination`. Both are full paths.

### `file.copy(target: string, destination: string): none`
Copies a file or folder from `target` to `destination`, leaving the original at `target` intact.
Both are full paths. Applies to both files and folders (folder copy is recursive, same as `delete`
and `move`).

### `file.encrypt(path: string, key: string = "F(rxYfi@8PS3b3Nw"): none`
### `file.decrypt(path: string, key: string = "F(rxYfi@8PS3b3Nw"): none`

Obfuscates/de-obfuscates a file's contents **in place on disk** — neither function returns
anything; the file at `path` is loaded, transformed, and rewritten as a side effect.

- Algorithm: **repeating-key XOR**, with the result represented as **hex**, applied **line by
  line**. `decrypt` reverses exactly what `encrypt` does, using the same key.
- `key` is optional, defaulting to `"F(rxYfi@8PS3b3Nw"` if not provided. (If a custom key needs to
  contain a character that would otherwise break the string literal, use Az's standard universal
  `\` escape — see `02-language-spec.md` section 8 — same as any other string.)

**Important framing note, not a behavior change:** this is **obfuscation, not real security**.
Repeating-key XOR is a well-known, easily-reversible scheme (trivial to break via known-plaintext or
frequency analysis, especially against structured file formats) — appropriate for lightweight
purposes like deterring casual tampering with save files or config data, but must not be documented
or marketed as protecting genuinely sensitive information. Build it exactly as specified; just don't
let downstream users mistake it for encryption in the cryptographic-security sense.

---

## Still open / not covered by this design pass

These weren't discussed and shouldn't be assumed:
- **Binary file support** — this design is implicitly text-based (`.write()` takes stringifiable
  values, `.getline()`/`.read()` return `string`). Binary read/write was not addressed.
- **Directory listing** — reading the *contents* of a directory (enumerating files/subfolders) was
  not discussed. `createDirectories`, `delete`, `move`, `copy`, `rename` all operate ON folders as
  whole units, but nothing lets a program enumerate what's inside one.
- **Error behavior when the path is invalid** for the original `("path", mode)` constructor — e.g.
  opening a nonexistent file in `'r'` mode, or a path in a directory that doesn't exist in `'w'`/
  `'a'` mode. Not specified. (Note: this is distinct from `file.rename`'s mismatch case, which *is*
  now specified as a runtime error — this remaining gap is specifically about the constructor/
  `loadFile` path.)
- **General runtime warning/diagnostic mechanism** — a couple of design points in this file (e.g.
  the original idea for `parentPath`'s no-parent case) considered emitting a runtime warning
  distinct from a hard error, similar in spirit to C++'s `std::cerr`. Az doesn't have any defined
  concept yet for a "non-fatal runtime diagnostic" channel, separate from fatal runtime errors and
  compile-time errors/warnings. This would be a language-wide feature (useful well beyond `file`),
  not something to bolt on ad hoc just for this library — worth a dedicated design pass later.
