# Built-in Libraries: `math` and `random`

Both imported via `using math;` / `using random;` (or selectively, e.g. `using math.sqrt;`), per the
standard import rules in `02-language-spec.md` section 7.

---

## `math`

| Function | Notes |
|---|---|
| `math.sqrt(double x): double` | square root |
| `math.pow(double base, double exp): double` | exponentiation |
| `math.abs(x)` | overloaded: `int` → `int`, `double` → `double`, `long long` → `long long` |
| `math.min(a, b)` | overloaded for `int`/`double`/`long long` (both args same type) |
| `math.max(a, b)` | overloaded for `int`/`double`/`long long` (both args same type) |
| `math.round(double x): int` | rounds to nearest integer |
| `math.floor(double x): int` | rounds down |
| `math.ceil(double x): int` | rounds up |
| `math.log(double x): double` | natural logarithm (base *e*) |
| `math.log2(double x): double` | logarithm base 2 — the one you want for "loop `log(n)` times instead of `n`" (halving something repeatedly down to 1 takes `log2(n)` steps — binary search, divide-and-conquer algorithms, etc.) |
| `math.log10(double x): double` | logarithm base 10 |
| `math.factorial(int n): unsigned long long` | `n!` — see overflow/negative-input notes below |
| `math.PI` | constant, `double` |
| `math.E` | constant, `double` (Euler's number) |

**`math.factorial` behavior:**
- Input must be a non-negative `int`. **`n < 0` is a runtime `ERROR`** (writes to
  `runtime_errors.log`, program terminates — see `12-runtime-diagnostics.md`), since factorial is
  mathematically undefined for negative numbers.
- Return type is **`unsigned long long`**, not plain `long long` — since a valid factorial input is
  never negative (already rejected above) and the result is therefore never negative either, there's
  no reason to spend a sign bit on the return type. (Note: this doesn't actually raise the overflow
  ceiling below — see next point — it's purely a correctness/clarity choice, not a range increase.)
- **Even `unsigned long long` isn't unlimited** — it overflows starting at `21!`. (`20!` ≈
  2.43×10¹⁸ is the largest factorial that fits in either a signed *or* unsigned 64-bit integer;
  `21!` ≈ 5.11×10¹⁹ exceeds both signed `long long`'s max (~9.22×10¹⁸) and unsigned `long long`'s
  max (~1.84×10¹⁹) — there's no factorial value that happens to fit in unsigned but not signed, the
  jump from `20!` to `21!` skips straight over that gap.) Rather than silently returning a wrapped-
  around/garbage number for `n > 20`, this is also a runtime `ERROR` (`"factorial(n) would overflow
  unsigned long long for n > 20"`), same mechanism as the negative-input case. Real arbitrary-
  precision ("big integer") support would remove this ceiling entirely, but that's a much larger
  feature and explicitly out of scope for v1.

**Deliberately excluded (not just undiscussed):** trigonometric functions (`sin`, `cos`, `tan`,
etc.) are **not** part of `math` — explicitly not wanted.

Implementation note: since these all map almost directly onto `<cmath>` functions, this library
should be a very thin wrapper layer in the runtime — little original logic needed beyond handling
the `int`/`double` overloads for `abs`/`min`/`max`.

---

## `random`

| Function | Notes |
|---|---|
| `random.randint(int min, int max): int` | inclusive on **both** ends |
| `random.randdouble(double min, double max): double` | inclusive on **both** ends |
| `random.choose(list<T> l): T` | returns a random element from the list |

Implementation note: `random.choose` is naturally generic over the list's element type `T`. Per the
general built-in implementation principle (see `01-compiler-architecture.md` section 7.6), this
should be implemented as a real C++ template in the runtime
(`template<typename T> T az_random_choose(const std::vector<T>& list)`), not duplicated per element
type.

No specific requirements were given regarding the underlying RNG (e.g. seeding behavior,
`<random>` vs `rand()`, thread-safety). Recommend using C++'s `<random>` facilities
(`std::mt19937` seeded from `std::random_device`) over legacy `rand()`, since it's not meaningfully
more complex to implement and avoids `rand()`'s well-known quality/distribution issues — but this
wasn't an explicit requirement, just a reasonable default.
