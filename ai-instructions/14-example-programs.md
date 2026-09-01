# Az Example Programs

Complete, illustrative `.az` programs exercising most of the language and built-in libraries
together. These aren't from the original design conversation verbatim (aside from the very first
one, adapted) — they're constructed here to give the implementer concrete, realistic targets to
test the compiler against, and to show how features compose in practice. Treat these as **test
fixtures worth actually compiling and running** once the pipeline exists, not just documentation.

---

## 1. Hello World (minimal)

```
//hello.az
using console;

func main(): none
{
    console.write("Hello, world!\n");
}
```

Exercises: `using`, `console.write`, `main`, comments.

---

## 2. Basic types, variables, and fstring

```
//basics.az
using console;

func main(): none
{
    int age = 25;
    double price = 19.99;
    string name = "Az";
    bool active = true;
    char grade = 'A';

    console.write(fstring("{name} is version {age}, priced at {price}\n"));
    console.write(fstring("Active: {active}, Grade: {grade}\n"));

    var inferred = age;              // var infers int here
    console.write(fstring("Inferred: {inferred}\n"));

    const int MAX_USERS = 100;
    console.write(fstring("Max users: {MAX_USERS}\n"));
}
```

Exercises: all core scalar types, `fstring` with multiple interpolations, `var` inference, `const`.

---

## 3. Control flow — if/else, while, for, times, switch

```
//control-flow.az
using console;

func classify(int n): string
{
    if n < 0 {
        return "negative";
    } else if n == 0 {
        return "zero";
    } else {
        return "positive";
    }
}

func main(): none
{
    // if / else if / else
    console.write(fstring("{classify(-5)}\n"));
    console.write(fstring("{classify(0)}\n"));
    console.write(fstring("{classify(5)}\n"));

    // while, with implicit _i
    temp int i = 0;
    while i < 3 {
        console.write(fstring("while _i={_i}, i={i}\n"));
        i++;
    }
    tempdelete i;

    // C-style for
    for int j = 0; j < 3; j++ {
        console.write(fstring("for j={j}, _i={_i}\n"));
    }

    // for with an omitted clause
    int k = 0;
    for none; k < 3; k++ {
        console.write(fstring("no-init for, k={k}\n"));
    }

    // range-based for
    list<int> nums = [10, 20, 30];
    for var n : nums {
        console.write(fstring("range-for n={n}, index _i={_i}\n"));
    }

    // times
    3 times {
        console.write(fstring("times iteration _i={_i}\n"));
    }

    // switch, no fallthrough, default required behavior shown
    int day = 3;
    switch day {
        case 1 { console.write("Monday\n"); }
        case 2 { console.write("Tuesday\n"); }
        case 3 { console.write("Wednesday\n"); }
        default { console.write("Some other day\n"); }
    }
}
```

Exercises: `if`/`else if`/`else`, `while` with `_i`, C-style `for` (including `none` clause),
range-based `for` with `_i`, `times` with `_i`, `switch`/`case`/`default`, `temp`/`tempdelete`.

---

## 4. Functions — defaults, overloading, recursion

```
//functions.az
using console;

func greet(string name = "World"): none
{
    console.write(fstring("Hello, {name}!\n"));
}

func describe(int x): string
{
    return fstring("int: {x}");
}

func overload describe(double x): string
{
    return fstring("double: {x}");
}

func overload describe(string x): string
{
    return fstring("string: {x}");
}

func factorial(int n): long long
{
    if n <= 1 {
        return 1;
    }
    return n * factorial(n - 1);
}

func main(): none
{
    greet();               // uses default "World"
    greet("Az");            // overrides default

    console.write(fstring("{describe(5)}\n"));       // int overload
    console.write(fstring("{describe(5.5)}\n"));      // double overload
    console.write(fstring("{describe(\"hi\")}\n"));   // string overload

    long long result = factorial(10);
    console.write(fstring("10! = {result}\n"));
}
```

Exercises: default parameters, `overload` keyword, recursion, calling overloaded functions.

---

## 5. Multi-file program — imports, both `using` forms, collision-safe overload

```
//mathutils.az   (an imported library file -- NOT the entry file, no main() allowed here)
func square(int x): int
{
    return x * x;
}

func cube(int x): int
{
    return x * x * x;
}
```

```
//stringutils.az
func shout(string s): string
{
    return s;   // (placeholder -- real uppercase logic would go here once
                 // more string built-ins exist; this file exists to demonstrate
                 // selective import syntax)
}
```

```
//main.az   (the entry file)
using console;
using mathutils.az;             // whole-file import -- needs prefix
using stringutils.shout;        // selective import -- called bare

func overload square(double x): double   // legal: differs in signature + uses `overload`,
{                                          // even though mathutils.az also has a `square`
    return x * x;
}

func main(): none
{
    console.write(fstring("mathutils.square(4) = {mathutils.square(4)}\n"));
    console.write(fstring("mathutils.cube(3) = {mathutils.cube(3)}\n"));
    console.write(fstring("local square(2.5) = {square(2.5)}\n"));
    console.write(fstring("{shout(\"hi\")}\n"));
}
```

Exercises: whole-file `using` (with `library.` prefix), selective `using` (bare call), the
`overload` keyword resolving what would otherwise be a name collision across files, confirming that
`mathutils.az` correctly has no `main()`.

---

## 6. `global`, `temp`, `swap`

```
//global-temp-swap.az
using console;

func recordVisit(): none
{
    global int visitCount = visitCount + 1;   // legal: `visitCount` is usable here even though,
    console.write(fstring("Visit #{visitCount}\n")); // textually, it's declared later, inside main()
}

func main(): none
{
    recordVisit();   // works because `global` compiles to an accessor function wrapping a
                       // lazily-initialized value (see 01-compiler-architecture.md section 7.4) --
                       // every reference resolves to the same function call regardless of where
                       // the `global int visitCount = 0;` statement itself sits in the source

    global int visitCount = 0;
    recordVisit();
    recordVisit();

    temp int a = 5;
    temp int b = 10;
    console.write(fstring("Before swap: a={a}, b={b}\n"));
    swap a, b;
    console.write(fstring("After swap: a={a}, b={b}\n"));
    tempdelete a;
    tempdelete b;
}
```

**Note:** this example is deliberately a bit contrived to demonstrate hoisting — in real code you'd
normally declare a `global` before using it, for readability, even though the language doesn't
require that ordering.

Exercises: `global` hoisting/before-declaration use, `temp`/`tempdelete`, `swap`.

---

## 7. `file` — read, write, and filesystem utilities

```
//file-demo.az
using console;
using file;

func main(): none
{
    file out("output.txt", 'w');
    out.write("First line\n");
    out.write(fstring("Number: {42}\n"));
    out.close();   // explicit close, though scope-exit would also handle it

    if file.exists("output.txt") {
        console.write("output.txt exists\n");
    }

    file in("output.txt", 'r');
    while !in.eof() {
        string line = in.getline();
        console.write(fstring("Read: {line}"));
    }
    in.close();

    console.write(fstring("Extension: {file.extension(\"output.txt\")}\n"));

    file.rename("output.txt", "renamed.txt");
    console.write("Renamed output.txt -> renamed.txt\n");

    file.copy("renamed.txt", "backup.txt");
    console.write("Copied renamed.txt -> backup.txt\n");

    file.delete("backup.txt");
    console.write("Deleted backup.txt\n");
}
```

Exercises: `'w'`/`'r'` modes, `.write`, `.eof`/`.getline`, `.close`, `file.exists`, `file.extension`,
`file.rename`, `file.copy`, `file.delete`.

---

## 8. `time` — stopwatch and countdown timer (polling)

```
//time-demo.az
using console;
using time;

func main(): none
{
    console.write(fstring("Current time: {time.now(\"yyyy/MM/dd hh:mm:ss\")}\n"));
    console.write(fstring("Epoch: {time.epoch()}\n"));

    var sw = time.startStopwatch();
    // ... imagine work happens here ...
    double elapsed = time.elapsedStopwatch(sw);
    console.write(fstring("Elapsed: {elapsed} seconds\n"));
    time.stopStopwatch(sw);

    var countdown = time.startTimer(5.0);
    while !time.isDoneTimer(countdown) {
        // polling loop -- a real program would do other work here between checks
    }
    console.write("Countdown finished!\n");
    time.stopTimer(countdown);
}
```

Exercises: `time.now`, `time.epoch`, stopwatch start/elapsed/stop, timer start/isDone/stop (polling
model, no callbacks).

---

## 9. `logs` and `runtime_errors.log` interaction

```
//logs-demo.az
using console;
using logs;
using file;

func main(): none
{
    logs.load("app.log");
    logs.write("Application started");                 // defaults to Info
    logs.write("Low disk space warning", Warning);
    logs.write("Unrecoverable state", Error);

    logs.unload();

    logs.write("This message is silently dropped, and logs a WARNING to runtime_errors.log");

    // This deliberately triggers a runtime ERROR (path mismatch), terminating the program,
    // logged to runtime_errors.log -- included here to illustrate the failure path, not as
    // something a real program should do carelessly:
    // file.rename("app.log", "/some/totally/different/path/renamed.log");
}
```

Exercises: `logs.load`/`write`/`unload`, the three severity literals, the no-log-loaded case
(routes to `runtime_errors.log`), and (commented out, since it would terminate the program) an
example of what would trigger a fatal runtime `ERROR`.

---

## 10. `math` and `random`

```
//math-random-demo.az
using console;
using math;
using random;

func main(): none
{
    console.write(fstring("sqrt(16) = {math.sqrt(16)}\n"));
    console.write(fstring("pow(2, 10) = {math.pow(2, 10)}\n"));
    console.write(fstring("abs(-5) = {math.abs(-5)}\n"));
    console.write(fstring("max(3, 7) = {math.max(3, 7)}\n"));
    console.write(fstring("round(3.6) = {math.round(3.6)}\n"));
    console.write(fstring("log2(1024) = {math.log2(1024)}\n"));
    console.write(fstring("5! = {math.factorial(5)}\n"));
    console.write(fstring("PI = {math.PI}\n"));

    int dieRoll = random.randint(1, 6);
    console.write(fstring("Die roll: {dieRoll}\n"));

    double randomChance = random.randdouble(0.0, 1.0);
    console.write(fstring("Random chance: {randomChance}\n"));

    list<string> options = ["red", "green", "blue"];
    string picked = random.choose(options);
    console.write(fstring("Picked: {picked}\n"));
}
```

Exercises: most of `math`, both `random` numeric functions, `random.choose` over a `list<string>`.

---

## 11. `console.input` validated reading

```
//input-demo.az
using console;

func main(): none
{
    int age = console.input(int, "How old are you? ");
    console.write(fstring("You are {age} years old.\n"));

    string name = console.input();   // unvalidated
    console.write(fstring("Hello, {name}!\n"));

    double price = console.input(double);   // uses default error message on invalid input
    console.write(fstring("Price entered: {price}\n"));
}
```

Exercises: `console.input()` bare, `console.input(type)`, `console.input(type, customMessage)`.

---

## 12. `skip` / `skip here` — multi-level break out of nested `if` blocks

```
//skip-demo.az
using console;

func validateUser(int age, bool hasId, bool hasPayment): none
{
    if age >= 18 {
        if hasId {
            if hasPayment {
                console.write("User is fully validated, proceeding...\n");
                skip done;   // jumps out of all three nested if-blocks at once,
                              // straight to the label near the end of the function
            }
            console.write("Missing payment info\n");
            skip done;
        }
        console.write("Missing ID\n");
        skip done;
    }

    console.write("User is under 18\n");

    skip here done;
    console.write("Validation check complete.\n");
}

func main(): none
{
    validateUser(20, true, true);    // exercises the deepest skip
    validateUser(20, true, false);   // exercises the middle skip
    validateUser(20, false, true);   // exercises the outer skip
    validateUser(16, true, true);    // falls through to the label naturally, no skip needed
}
```

This demonstrates the core motivating use case: bailing out of several layers of nested `if`
validation checks without needing a `bool` flag variable threaded through every level, or
restructuring the logic into early-return helper functions. Every `skip done;` above is legal
because the label `skip here done;` sits in the function's top-level block, which is an ancestor of
every block containing a `skip done;` call, per the rule in `02-language-spec.md`.

---

## 13. `list`/`dict` element access — Python-style indexing

```
//indexing-demo.az
using console;

func main(): none
{
    list<int> nums = [10, 20, 30, 40, 50];
    console.write(fstring("First: {nums[0]}\n"));
    console.write(fstring("Last (negative index): {nums[-1]}\n"));
    nums[1] = 99;
    console.write(fstring("After nums[1] = 99: {nums[1]}\n"));

    dict<string, int> scores = {"alice": 90, "bob": 85};
    console.write(fstring("Alice's score: {scores[\"alice\"]}\n"));
    scores["carol"] = 70;   // missing key on WRITE -- auto-inserts, no error
    console.write(fstring("Carol's score: {scores[\"carol\"]}\n"));

    switch scores["alice"] {
        case 90 { console.write("Alice has a 90\n"); }
        default { console.write("Alice has some other score\n"); }
    }

    string status = "active";
    switch status {   // switch on a non-int type -- confirmed to work on any type
        case "active" { console.write("Status: active\n"); }
        case "inactive" { console.write("Status: inactive\n"); }
        default { console.write("Status: unknown\n"); }
    }

    // The following would each trigger a fatal runtime ERROR if uncommented:
    // console.write(fstring("{nums[100]}"));        // list index out of range
    // console.write(fstring("{scores[\"missing\"]}"));  // dict READ of a missing key -- errors
    //                                                     // (unlike dict WRITE, which auto-inserts)
}
```

---

## Suggested use of these examples

Recommend the implementer build these up incrementally as the compiler's own test suite (see
`16-testing-strategy.md`) — roughly in the order listed here, since each one exercises a strictly
larger slice of the language than the last. A working compiler should be able to transpile and
successfully compile (via `g++`/`clang++`) every example in this file to a real, runnable
executable that produces the described output.
