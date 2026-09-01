# Built-in Library: `console`

Import with `using console;` (built-ins go through the same `using` mechanism as user-written `.az`
files — see `02-language-spec.md` section 7).

This was the first built-in library designed, and sets the general pattern other built-ins should
follow: hide platform/annoyance complexity from the Az programmer, and implement anything with
naturally generic logic as a real C++ template in the runtime rather than duplicated per-type code.

---

## `console.write(value)`

Prints `value` to standard output.

- Accepts **any type** — not restricted to `string`.
- Non-string/non-char values are **automatically converted** to their string representation, using
  the same conversion machinery as `string(...)` (see `02-language-spec.md` section 8).
- If the value passed is not already `string`/`char`, emit a **compile-time warning** (not an
  error) — identical policy to `fstring(...)`'s `{}` segments, and for the same reason (catching
  accidental large/unintended dumps, e.g. printing a whole `list` by mistake).

```
console.write("Hello world!\n");
console.write(42);              // valid; auto-converted; triggers a compiler warning
```

---

## `console.input(): string`

Reads a line of input from the user, no validation, returns it as `string`.

```
string name = console.input();
```

## `console.input(type): T`

Reads input, **looping until the entered text can be validly converted to `type`**, then returns a
value of that type. `type` here is one of the literal type keywords: `int`, `double`, `bool`,
`string`. This is not real user-facing generics — it's compiler-recognized special syntax for this
one built-in (see implementation note below).

```
int age = console.input(int);
```

On invalid input, prints the default error message and re-prompts:
```
Invalid input type! Enter int
```
(This message is itself generated via `fstring("Invalid input type! Enter {inputType}")`, where
`inputType` is the literal name of the requested type as a string, e.g. `"int"`.)

## `console.input(type, customMessage): T`

Same as above, but `customMessage` (a `string`) is shown instead of the default error message on
invalid input.

```
int idk = console.input(int, "Enter a number!\n");
```

### Implementation note (see `01-compiler-architecture.md` section 7.6 for full detail)

`console.input` must **not** be implemented as four separate, hand-duplicated overload
implementations. Because Az v1 has no user-facing generics but the underlying C++ code the compiler
generates is free to use real templates, implement this as a single templated runtime function:

```cpp
template <typename T>
T az_console_input();  // no-args version routes to std::string specialization, or a separate overload

template <typename T>
T az_console_input(const std::string& customErrorMessage);
```

The Az compiler recognizes the special call pattern `console.input(<type literal>, ...)` and emits
the correctly-instantiated template call (`az_console_input<int>(...)`, etc.) — the "type literal"
argument is a compile-time construct, not a runtime value being passed around.

Behavior inside the template: print prompt (if any) → read a line → attempt conversion to `T` → on
failure, print the error message (default or custom) and retry → on success, return the value.

---

## `console.run(command): string`

Executes `command` (a `string`) as a shell/terminal command — equivalent in spirit to C++'s
`system()`, but should be implemented more safely/cleanly under the hood (e.g. via `popen`/pipe
capture rather than a bare `system()` call, so output can actually be captured).

- **Returns the command's output as a `string`.**

```
string result = console.run("ls -la");
console.write(result);
```

---

## `console.clear()`

Clears the terminal screen.

- **Must be OS-aware.** This is not simply sugar for `console.run("clear")` — on Windows, the
  correct command is `cls`, not `clear`, so a naive passthrough would silently fail/error on
  Windows. Implement this with its own dedicated, platform-detecting logic in the runtime (compile
  time `#ifdef`/OS detection, or runtime detection — compile-time is preferable since the target
  platform is generally known at compile time).

```
console.clear();
```
