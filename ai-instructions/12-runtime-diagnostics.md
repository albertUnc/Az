# Runtime Diagnostics System (`runtime_errors.log`)

This is the **general-purpose mechanism for runtime errors and warnings produced by a *compiled
Az program while it's running*** — as opposed to `09-transpile-logging.md`, which
covers the transpiler's own *build-time* diagnostics, and `11-library-logs.md`, which is a
user-facing library the Az programmer explicitly calls (`logs.load`/`logs.write`/`logs.unload`) to
log their *own* application-level messages.

**Three genuinely separate systems, easy to confuse by name — keep them distinct:**

| System | When it runs | Who writes to it | File |
|---|---|---|---|
| Transpile-time logging (`09-transpile-logging.md`) | While `az-transpiler` is building the program | The transpiler itself | `transpile-logs.txt` |
| Runtime diagnostics (this file) | While the *compiled program* is executing | The Az **runtime** itself, automatically, for internal conditions (not something the Az programmer calls directly) | `runtime_errors.log` |
| `logs` library (`11-library-logs.md`) | While the *compiled program* is executing | The Az **programmer**, explicitly, via `logs.write(...)` in their own code | Whatever path they pass to `logs.load(...)` |

This file resolves a gap that had been left as an unfinished placeholder in two earlier specs
(`file.parentPath`'s "no parent" case in `06-library-file.md`, and `logs.write()`-with-no-log-loaded
in `11-library-logs.md`) — both should now be updated to route through this system instead of their
previous stopgap behavior (stderr printing, or "no additional signal"). See the end of this file for
the specific updates.

---

## Design: modeled directly on the transpile-time logger, with one deliberate difference

Same general shape as `09-transpile-logging.md`:

- **Two severity levels only: `ERROR` and `WARNING`** — deliberately **no `INFO` level at
  runtime** (unlike the transpile-time logger, which does have one). `ERROR` = something
  unrecoverable happened and the program terminates; `WARNING` = something risky/unexpected
  happened but the program keeps running normally. Purely informational logging is left entirely
  to the programmer's own use of the `logs` library (`11-library-logs.md`) — if they want an
  "info" record of their program's behavior, that's what `logs.write(message, Info)` is for. The
  runtime diagnostics system only exists for conditions the *runtime itself* detects going wrong,
  which by definition are never merely informational.
- **File wiped fresh on every run of the compiled program** — each execution's `runtime_errors.log`
  reflects only that run, not accumulated history across runs. (Same policy as
  `transpile-logs.txt` being wiped fresh on every transpile.)
- **Entry format**, adapted for a runtime context (no source file/line available at runtime the way
  the transpiler has it at build time — this uses a timestamp instead):
  ```
  [LEVEL] at hh:mm:ss - message
  ```
  Example:
  ```
  [WARNING] at 18:34:23 - logs.write() called with no log loaded, message discarded
  ```

**The one deliberate, explicit difference from the transpile-time logger: nothing is printed to
the terminal.** Both `ERROR` and `WARNING` go **only** to `runtime_errors.log`. This is intentional: unlike the transpiler (which always runs as a
terminal-attached command-line tool), a *compiled Az program* might be a GUI application, a
background service, or anything else with no meaningful terminal to print to — so the runtime
diagnostics system doesn't assume one exists, and stays purely file-based across the board.

## File location — CONFIRMED

Written next to the running executable (i.e. wherever the compiled program's binary actually
lives) — mirrors how `transpile-logs.txt` sits next to the source `.az` file being compiled.

## `ERROR` and program termination

An `ERROR`-level runtime diagnostic corresponds to something unrecoverable — the program logs the
entry and then **terminates** (mirroring how an `ERROR` at transpile time means the build simply
cannot complete). A `WARNING` is logged and the program continues running normally afterward,
exactly as before.

---

## What should actually write to this system

Any internal condition the Az **runtime itself** detects, not something the Az programmer manually
triggers. Concrete examples already implied by earlier specs, now formalized to route here:

- `file.rename(...)`'s path-mismatch case (see `06-library-file.md`) — **`ERROR`**, program
  terminates
- `file.parentPath(...)`'s "no parent exists" case — **`WARNING`**, program continues, function
  still returns the empty string as already specified
- `logs.write(...)` called with no log currently loaded (see `11-library-logs.md`) — **`WARNING`**,
  program continues, the write is simply discarded
- `logs.load(...)` failing (its target's parent directory doesn't exist) — **`WARNING`**, program
  continues, `logs` is left in "unloaded" state (see `11-library-logs.md`). Note this is a
  `WARNING`, not an `ERROR` — deliberately non-fatal, unlike the `file` constructor case below.
- The `file` constructor / `loadFile(...)` opening a path whose **parent directory doesn't exist**
  — see `06-library-file.md`. (Opening a path where only the *file itself* is missing, in any mode,
  is **not** an error — it auto-creates the file. Only a missing containing directory fails.) —
  **`ERROR`**, program terminates. The recommended way to avoid hitting this is calling
  `file.createDirectories(...)` and/or `file.exists(path)` first.
- `list` index out of range (read or write), and `dict` **read** of a missing key — see
  `02-language-spec.md` section 2a and `01-compiler-architecture.md` section 7.6b — **`ERROR`**,
  program terminates. (`dict` **write** to a missing key is not an error — it auto-inserts, per the
  same section — so it never reaches this system at all.)
- General candidates not yet formally spec'd anywhere else, but which will need this system once
  they're implemented: division by zero — not fully speced yet (see `08-open-questions.md`), but
  whatever its eventual behavior is, it should report through this system rather than inventing a
  separate one-off mechanism.

---

## Updates this resolves in other files

- **`06-library-file.md`, `parentPath`:** previously "returns an empty string... no additional
  signal" — now: also writes a `WARNING` entry to `runtime_errors.log`, per the table above.
- **`06-library-file.md`, `rename`:** previously "a runtime error" (unspecified mechanism) — now:
  writes an `ERROR` entry to `runtime_errors.log` and the program terminates.
- **`11-library-logs.md`, `write()` with no log loaded:** previously "recommended default: print a
  plain message to `stderr`... flagged as the seed of the broader general runtime-diagnostic
  mechanism gap" — now: that gap is filled by this file. Update the behavior to write a `WARNING`
  entry to `runtime_errors.log` instead of `stderr`. This item can be considered resolved, not a
  placeholder anymore.

---

## Still open

- A full, authoritative list of exactly which runtime conditions produce which level (division by
  zero, out-of-bounds access, etc.) doesn't exist yet beyond the examples above — not a complete
  enumeration, but the pattern (unrecoverable → `ERROR` + terminate, risky-but-continuable →
  `WARNING`) is established and should be applied consistently as more conditions get implemented.
