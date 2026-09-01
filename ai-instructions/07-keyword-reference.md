# Az Reserved Keyword Reference

Full list of reserved words compiled from the design conversation. This should be treated as
**confirmed but not necessarily exhaustive** — a couple of entries near the bottom are marked as
"implied but not explicitly discussed" (things like `true`/`false`), since every language obviously
needs them but they weren't the subject of explicit back-and-forth the way most of this list was.
Flag any additions made beyond this list clearly if the implementer needs to add anything not
covered here.

## Types
```
int
double
long long        (two-word type name)
signed
unsigned
bool
string
char
list
dict
var
none
file            (special built-in type — not a preview of user-facing OOP; see 06-library-file.md)
```

## Variable qualifiers / statements
```
const
constexpr
global
temp
tempdelete
swap
```

## Control flow
```
if
else
while
for
times
break
continue
switch
case
default
skip            (labeled multi-level break/goto -- see 02-language-spec.md Control Flow section)
here            (used only as part of `skip here <name>;` -- not meaningful standalone)
```

## Functions / program structure
```
func
return
using
overload        (marks a function declaration as an intentional overload of an existing name)
main            (not a keyword in the syntactic sense, but the literal required function name for
                 the program's single entry point — treat as reserved/special)
```

## Built-in conversion / string functions
```
fstring
string          (also a type name — context distinguishes usage: `string` as a type in a
                 declaration vs. `string(...)` as a function call)
int             (also a type name, same dual usage as `string` above)
double          (also a type name, same dual usage as `string` above)
LL              (conversion-only — short for `long long`, since `long long` itself is two
                 tokens and can't be used as a function name; LL(value) converts to long long)
```

## Literals — implied but not explicitly discussed in the design conversation
```
true
false
```
(Needed for `bool` to be usable at all; wasn't a point of explicit discussion because it's an
obvious, low-risk necessity, but flagging it here rather than letting it be silently assumed.)

## Implicitly reserved (not keywords, but special/auto-defined identifiers)
```
_i              — auto-available loop index inside while / for (both forms) / times loop bodies.
                  Should be treated as reserved within those contexts to prevent the user
                  accidentally declaring their own variable named _i and causing a conflict.
```

## Explicitly NOT included in v1 (do not add these as keywords yet)
```
and, or, not     — logical operator word-aliases were explicitly rejected; use && || ! only
elif             — explicitly rejected; use `else if` (two words) instead
owned, shared, weak, unique  — smart-pointer-style keywords, deferred to v2 along with
                                pointers/references generally
```
