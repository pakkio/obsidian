---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
aliases:
  - for yield
  - for comprehension
  - Scala for
---

# ZEXT `for (x <- xs) yield e`: Scala's for-comprehension over lists

← Back to [[ZEXT Syntax Sugar Index]] | Single-generator form: [[ZEXT Small Sugar - def on let defer assert print in]] §8 | Engine: [[ZEngine]] | Added 2026-11-20

Scala's syntax, exactly, over LSL lists: generators `x <- list`, guards `if cond`, `yield`. No value definitions (`y = e`) in the header.

```lsl
list a = [1, 2, 3];
list b = [10, 20];

list ys = for (x <- a) yield x * 2;                         // map:      2, 4, 6
list s  = for (x <- a; y <- b if x != 2) yield x + y;       // flatMap + filter: 11, 21, 13, 23
list q  = for { x <- [1, 2, 3, 4] if x % 2 == 0 } yield x * x;   // braces, ';' optional: 4, 16
list w  = for (line <- llParseString2List("a b,c d", [","], []);
               word <- llParseString2List(line, [" "], [])) yield word + "!";   // a!, b!, c!, d!

for (x <- [1, 2]; y <- ["a", "b"]) llOwnerSay((string)x + y);   // no yield = foreach: 1a 1b 2a 2b

list d = for (x: integer <- llJson2List(j)) yield x * 10;   // typed generator
```

| Piece | Meaning |
| :--- | :--- |
| `x <- list` | generator; write `<-` with no space (`x < -y` is still a comparison) |
| `x: integer <- list` | generator with its element type written |
| `if cond` | guard: filters at the point it is written, any number, after any generator |
| `;` | separator; optional in the `{ }` form |
| `yield e` | the result is the list of every `e`; a list-valued `e` is flattened in (LSL has no nested lists) |
| no `yield` | the body statement runs for every combination (Scala's `foreach`) |

Later generators are evaluated again for each element of the earlier ones, so one may depend on another (the `word <- …(line…)` line above), as in Scala.

## Where it can go

A `for … yield` is an expression: declarations, assignments, call arguments, `return`, `if` and `switch` tests all work (`llOwnerSay(llList2CSV(for (x <- xs) yield x + 1));`). The loops are placed in front of the statement that contains it, so three places are refused with a message:

- a `while` / `do` / `for (;;)` condition or update: it would be computed only once;
- a global initialiser: there is no statement to put the loops in front of;
- a `->` chain step: it would run before the step's input exists.

Inside an `&&`, `||` or `?:` operand it is computed even when that operand is not evaluated.

## Element types

List items carry their types only at run time, but a generator variable needs a static type. It is taken from the annotation, or read off the source:

- a list literal of one kind of literal (`[1, 2, 3]`, `["a", "b"]`);
- `llParseString2List`, `llParseStringKeepNulls`, `llCSV2List` (strings);
- slices, `llList2List`, `llListSort`, `llDeleteSubList`, `llList2ListStrided`, `llListReplaceList` and `+` of such lists;
- a list built by an earlier `for … yield` whose yielded value has an evident type (`list ys = for (x <- a) yield x * 2;` then `y <- ys`);
- a list variable whose every declaration is such a literal and which is **never assigned anywhere in the script** (one `xs += …` anywhere and its type is no longer trusted).

Otherwise it is a compile error asking for the annotation: `cannot tell the element type of the list 'x' iterates over; write it: 'x: integer <- ...'`. It never guesses.

## How it is lowered

In the parser (`ZLSLScalaFor.cs`), into nested indexed loops, one per generator, with the guards as `if`s; with `yield`, into a generated local `__zfy<n>` that the expression is replaced by:

```lsl
list s = for (x <- a; y <- b if x != 2) yield x + y;
// becomes, in front of the statement:
list __zfy0 = [];
{ list _lst = a; for (integer _idx = 0; _idx < llGetListLength(_lst); ++_idx) { integer x = _lst[_idx];
    { list _lst2 = b; for (...) { integer y = ...;
        if (x != 2) __zfy0 = __zfy0 + [x + y]; } } } }
list s = __zfy0;
```

So the runtime and `zengine desugar` see only loops (`ZDesugarTest.TestScalaForIsEquivalent` runs both and compares). Tests: `ZScalaForTest` (12).

See also: [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]] (`->` chains, the other monadic-looking sugar: a bind over async replies).
