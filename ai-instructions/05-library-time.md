# Built-in Library: `time`

Import with `using time;` (or selectively), per standard import rules.

`time` covers three things: getting the current time/date, a stopwatch (counts up, no target), and
a countdown timer (counts down from a set duration). Stopwatches and timers both support
pause/resume, and **multiple independent instances of each can exist simultaneously** — this is why
both return a "handle" that must be passed back into every subsequent call for that specific
instance, rather than there being one implicit global stopwatch/timer.

---

## Current time

### `time.now(format: string): string`

Returns the current date/time, formatted according to a token-based format string. Recognized
tokens:

| Token | Meaning |
|---|---|
| `yyyy` | 4-digit year |
| `MM` | 2-digit month — **uppercase**, to distinguish from `mm` |
| `dd` | 2-digit day |
| `hh` | 2-digit hour |
| `mm` | 2-digit minute — **lowercase**, to distinguish from `MM` |
| `ss` | 2-digit second |
| `fff` | 3-digit millisecond |

**Capitalization matters and is deliberate:** `MM` (month) and `mm` (minute) are two different
tokens that differ only in case — this is intentional, not a typo, and matches the same convention
several other date/time formatting systems use for exactly this reason (month and minute would
otherwise be genuinely ambiguous). Get the case right when using this format string.

Anything in the format string that isn't one of these tokens is treated as literal text and passed
through unchanged (allowing things like `"Today is dd/MM/yyyy"`).

```
console.write(time.now("yyyy/MM/dd hh:mm:ss.fff"));
```

### `time.epoch(): long long`

Returns the current time as seconds since the Unix epoch (1970-01-01T00:00:00Z).

```
long long timestamp = time.epoch();
```

---

## Stopwatch (counts up, no target)

Returns an opaque handle from `startStopwatch()` — see `01-compiler-architecture.md` section 7.7
for the recommended underlying implementation of handle types.

| Function | Notes |
|---|---|
| `time.startStopwatch(): var` | begins a new, independent stopwatch; returns its handle |
| `time.pauseStopwatch(handle)` | pauses it; elapsed time stops accumulating |
| `time.resumeStopwatch(handle)` | resumes after a pause |
| `time.elapsedStopwatch(handle): double` | seconds elapsed so far, **excluding any paused time**; can be called repeatedly without stopping the stopwatch |
| `time.stopStopwatch(handle)` | ends it and releases any associated resources |

```
var sw = time.startStopwatch();
// ... do work ...
double seconds = time.elapsedStopwatch(sw);
time.stopStopwatch(sw);
```

---

## Countdown timer (counts down from a fixed duration)

Same shape as the stopwatch, but counts down instead of up, and exposes a "done" check.

| Function | Notes |
|---|---|
| `time.startTimer(seconds: double): var` | begins counting down from `seconds`; returns its handle |
| `time.pauseTimer(handle)` | pauses the countdown |
| `time.resumeTimer(handle)` | resumes after a pause |
| `time.remainingTimer(handle): double` | seconds remaining |
| `time.isDoneTimer(handle): bool` | `true` once the countdown reaches zero |
| `time.stopTimer(handle)` | ends it and releases any associated resources |

**v1 uses polling only** — there is no callback/trigger mechanism for "do X automatically when the
timer hits zero." A program must manually check `isDoneTimer(handle)`, typically inside a loop.
Callback support (passing a function to run automatically at zero) was explicitly discussed and
deferred — it implies passing functions as values, a meaningfully bigger feature not part of v1.

```
var countdown = time.startTimer(10.0);
while !time.isDoneTimer(countdown) {
    // ... do other work, check periodically ...
}
console.write("Time's up!\n");
time.stopTimer(countdown);
```

---

## Implementation notes

- The handle type returned by `startStopwatch`/`startTimer` is **not** one of Az's normal value
  types — it requires its own runtime representation. See
  `01-compiler-architecture.md` section 7.7 for the recommended approach (a small runtime C++ class
  wrapping the actual timing state).
- **Confirmed behavior for a handle used after it's stopped:** calling `elapsedStopwatch`/
  `remainingTimer`/etc. on a handle after `stopStopwatch`/`stopTimer` was already called on it is
  **not an error** — it simply returns the frozen value the handle held at the moment it was
  stopped, rather than continuing to update.
- All stopwatch/timer time values are `double`, representing seconds, throughout — both inputs
  (e.g. `startTimer(seconds: double)`) and outputs (`elapsedStopwatch`, `remainingTimer`).
