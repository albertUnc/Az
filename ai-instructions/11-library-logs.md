# Built-in Library: `logs`

Import with `using logs;`, per standard import rules.

This is a **runtime** library — for compiled Az programs to log their own activity while running.
Completely separate from `09-transpile-logging.md`, which covers the *transpiler's own* build-time
diagnostics. Different audience, different lifecycle: this one runs every time the compiled program
runs; that one runs only when the program is being built.

## Design note: single global log slot, not a handle system

Unlike `time` (which supports many simultaneous stopwatches/timers via handles) or `file` (which
supports many simultaneous open file instances via variables), `logs` has **exactly one "currently
loaded" log at a time**, tracked internally by the runtime — there's no `logs` type, no variable to
declare, and no handle passed around. `load()`/`write()`/`unload()` all implicitly operate on
whichever single log is currently active.

## Design note: `Info`/`Warning`/`Error` are compiler-recognized literals, not a general enum feature

Az has no general user-facing enum type yet. `Info`, `Warning`, and `Error` (as used in
`logs.write(message, Warning)`) are **special, compiler-recognized literals specific to this one
function** — the same pattern already used for `console.input(int)`, where `int` is a compile-time
construct the compiler recognizes, not a real runtime value being passed around (see
`03-library-console.md` and `01-compiler-architecture.md` section 7.6). This is **not** a preview of
general enums in Az — don't treat it as license to assume enums exist as a language feature.
Underlying implementation: a plain C++ enum in the runtime is a perfectly reasonable backing
representation, but that's purely an internal detail invisible to the Az programmer.

---

## `logs.load(path: string): none`

Loads the log file at `path` as the currently active log. If a log was already loaded when this is
called, the previous one is **automatically unloaded first** (no need to call `unload()` yourself
before loading a different file — calling `load()` again just switches which file is active).

- If a file already exists at `path`, it's opened in **append mode** — existing content is
  preserved, new entries are added after it.
- If no file exists at `path`, it's created.

Either way, immediately after loading, one line is automatically written to the file marking the
new session, using a fuller date+time stamp (distinct from `write()`'s time-only stamp, since this
marks a new session boundary, potentially on a different day than the previous entries):

```
[yyyy/MM/dd hh:mm:ss] Successfully loaded log
```
or, if the file didn't already exist:
```
[yyyy/MM/dd hh:mm:ss] Successfully created log
```

(Token meanings match `time.now`'s format tokens — see `05-library-time.md`. **The exact wording/
format of this auto-written line is a reasonable proposed default, not something explicitly
dictated beyond "date and time" + the two message variants** — confirm before treating it as
completely final if precision matters.)

```
logs.load("session.log");
```

## `logs.write(message: string, level = Info): none`

Writes one entry to the currently loaded log. `level` is optional and defaults to `Info` if
omitted; otherwise pass `Info`, `Warning`, or `Error` (see the compiler-recognized-literal note
above).

- Automatically prefixes the entry with the current time (time only, not date — see `load()` above
  for the fuller date+time stamp used at session-start instead) and the level, in the format:
  ```
  [hh:mm:ss] [LEVEL] message
  ```
- **Automatically appends a newline** — the caller does not need to include `\n` in `message`.

```
logs.write("Message", Warning);
// writes: [18:34:23] [WARNING] Message
```
```
logs.write("Started up");
// level defaults to Info — writes: [18:34:23] [INFO] Started up
```

### Behavior when no log is loaded — RESOLVED, routes through the runtime diagnostics system

If `write()` is called before any `load()` (or after `unload()`), the call **silently does nothing**
to the (nonexistent) log file — but a `WARNING` entry is written to `runtime_errors.log` (see
`12-runtime-diagnostics.md` for the general runtime diagnostics system this routes through), e.g.:
```
[WARNING] at 18:34:23 - logs.write() called with no log loaded, message discarded
```
The program continues running normally afterward — only this one `write()` call is dropped.

## `logs.unload(): none`

Unloads the currently active log, if one is loaded. After this, `logs` has no active log until
`load()` is called again (and any `write()` calls in between silently no-op, per above).

Calling `unload()` when nothing is loaded is not an error — it's simply a no-op.

```
logs.unload();
```

---

## `load()` failure behavior — RESOLVED

`logs.load(path)` opens/creates its target the same way `file` does (see `06-library-file.md`) —
which, per that spec, means the only realistic failure case is a **missing parent directory**
(opening a file that doesn't exist but whose directory *does* exist just creates it, same as
`file`). If that happens:

- A single `WARNING` is written to `runtime_errors.log` (**not** the user's own log file — it
  couldn't be created, so there's nowhere else for this to go) — this is the "automatically made"
  log referred to in `12-runtime-diagnostics.md`, distinct from the `logs` library's own file.
- The program **does not terminate.** `logs` is simply left in its "unloaded" state, exactly as if
  `load()` had never been called at all.
- Every subsequent `logs.write(...)` call then naturally falls through to the already-established
  "no log loaded" behavior above — silently discarded, with its own fresh `WARNING` written to
  `runtime_errors.log` **each time it happens.** This is what gives the effect of "every write/read
  attempt after a failed load quietly keeps failing on its own," without needing any special-cased
  logic beyond what's already defined for the no-log-loaded case.

This is expected to be **very rare in practice** — the recommended way to avoid it entirely is
calling `file.exists(path)` / `file.createDirectories(...)` before `logs.load(path)`, the same
general pattern recommended for `file` itself.

## Still open

- The exact wording/format of `load()`'s auto-written session-start line is a proposed default
  (see above), not explicitly dictated beyond the two message variants and "date and time."
