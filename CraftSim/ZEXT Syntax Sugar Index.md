---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - index
aliases:
  - ZEXT sugar
  - ZEngine syntax sugar
  - braceless LSL
---

# ZEXT Syntax Sugar: index (2026-10 syntax pass)

← Back to [[CraftSim]] | Older reference: [[ZEXT Language Extensions]] | Engine: [[ZEngine]]

A pass over ZEngine's language surface, finished 2026-10-22: LSL you can write **without braces or semicolons**, with real **async arrows**, **JSON-checked strings**, **heredocs**, and a handful of small conveniences, plus a **rewritten `zengine desugar`** that turns all of it back into stock LSL and an **equivalence test** that proves the two forms behave the same. Everything here is ZEXT only (`[ZEngine] ZEXT = false` rejects it).

> [!SUMMARY] Where to start
> - Want the look of it? [[ZEXT Sugar Showcase - Annotated]] (one script, nearly everything, and the stock LSL it becomes).
> - Writing braceless code? [[ZEXT Braceless Indent Mode]].
> - Waiting for a reply or a timer? [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]].
> - Need to ship to an engine that only speaks LSL? [[ZEXT Desugar - Stock LSL Output]].
> - Do you trust the desugarer? [[ZEXT Desugar Verification Report]] says what was tested, what was found, what is not covered.
> - Want to see it in a script that does real work? [[ZEXT Sugar and Async - Worked Examples]] §9: `tasks/doc_reader_indent.lsl` reads the first document in an object and says how long it took, using `#indent`, `def`, f-strings and `->` with a timeout (2026-11-07).

## The notes

| Note | Covers | Runnable example (repo, `tasks/desugar_suite/`) |
| :--- | :--- | :--- |
| [[ZEXT Braceless Indent Mode]] | `#indent`: blocks by `:` and indentation, newline = end of statement, optional condition parentheses, `elif` / `unless` / `until` | `sugar_01_indent_braceless.lsl` |
| [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]] | `f() -> x` (single, typed, destructured, timeout `else`, `await` with per-bind timeouts, chains `f(x) -> g(_) -> r`), `["k" => v]` maps, `(…) => { }` closures, `timer ->` | `sugar_02_async_arrows.lsl`, `sugar_03_fat_arrow.lsl`, `sugar_10_inline_timers.lsl`, `level2_named_timer.lsl` |
| [[ZEXT JSON Literals (j-strings)]] | `j"…"`, `j'…'`, `j"""…"""`, `jr'…'`, `j<< ID`: strings checked as JSON at compile time | `sugar_04_json_strings.lsl` |
| [[ZEXT Heredocs]] | `<< ID … ID`, any non-space delimiter, raw body | `sugar_05_heredocs.lsl` |
| [[ZEXT Classes and Generics]] | `class C { fields; C(...) {...} methods }`, value semantics, `class Box<T>`, `T maxOf<T>(...)` templates (2026-11-20) | — |
| [[ZEXT Scala-style Classes - Inheritance, Case Classes, Implicits]] | `extends B with T`, `trait`, `abstract`, `super`, class parameters `class P(float x)`, `case class`, `implicit` values/parameters/conversions, `implicit class` extension methods (2026-11-21) | — |
| [[ZEXT For Comprehensions (Scala)]] | `for (x <- xs; y <- ys if c) yield e`, `for { … } yield e`, `for (x <- xs) stmt` (2026-11-20) | — |
| [[ZEXT Small Sugar - def on let defer assert print in]] | `def`, `on`, default and named arguments, `let`, `defer`, `assert`, `print`, chained comparison, `in`, list comprehension, `0b` `0o` `1_000` | `sugar_06_small_sugar.lsl` |
| [[ZEXT Sugar - Strings, Operators and States (annotated)]] | f/r/triple/single strings, `%`, `*`, slices; `?:` `??` `\|>` bitwise assignment, `boolean`; state arguments | `sugar_07…09` |
| [[ZEXT Desugar - Stock LSL Output]] | what each construct becomes, generated names, helper functions, known differences, what is refused | all of the above |
| [[ZEXT Desugar Verification Report]] | the evidence: suite, corpus sweep, bugs found and fixed, gaps | `ZDesugarTest` |
| [[ZEXT Sugar Showcase - Annotated]] | one 114-line braceless script and its 459-line stock-LSL form | `tasks/sugar_showcase.lsl` |

## The arrows at a glance

| Arrow | Meaning | Note |
| :--- | :--- | :--- |
| `call() -> x` | suspend until this request's reply, bind it (a monadic bind in all but name). `call() -> x : 90.0 else:` adds a timeout; a real use with timing is in the arrows note and [[ZEXT Sugar and Async - Worked Examples]] §9 | [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]] |
| `f(x) -> g(_) -> r` | chain: each call waits for the previous, `_` is its result; a trailing `: t else` covers every step (2026-11-20) | same, §1.1 |
| `timer -> 0.5:` | inline timer handler | same |
| `class C { … }`, `Box<T>`, `maxOf<T>(…)` | classes (value semantics: one variable per field) and C++-style generic templates (2026-11-20) | [[ZEXT Classes and Generics]] |
| `class C(...) extends B(...) with T`, `trait`, `case class`, `implicit` | Scala-style inheritance (linearization, stackable traits), class parameters, case classes, implicits and extension methods (2026-11-21) | [[ZEXT Scala-style Classes - Inheritance, Case Classes, Implicits]] |
| `for (x <- xs if c) yield e` | Scala for-comprehension over lists: several generators, guards, no value definitions (2026-11-20) | [[ZEXT For Comprehensions (Scala)]] |
| `["k" => v]` | map literal (a flat list) | same |
| `(args) => { … }` | closure, last argument of `llSetTimerEvent` / `llListen` / `llSensor` | same |
| `x \|> f()` | pipeline | [[ZEXT Language Extensions]] |
| `-> type` after `def f(…)` | return type | [[ZEXT Small Sugar - def on let defer assert print in]] |

## The one-line summary of every construct

`#indent` · `elif` `unless` `until` · `def f() -> t` · `on touch_start()` · `f(x, b: 2)` and `b = 2` defaults · `let x = e` · `defer s` · `assert c, "m"` · `print(a, b)` (owner only; for public chat define `def say(...)`, see §9 of the worked examples) · `f"…{expr}…"` · `0 < x <= 9` · `x in list` · `[e for T v in l if c]` · `0b101 0o17 1_000` · `j'{…}'` · `<< END … END` · `->` · `=>` · `timer ->`.

## Where the code is

| Piece | File (`OpenSim/Region/ScriptEngine/ZEngine/`) |
| :--- | :--- |
| indentation, `elif`/`unless`/`until` | `ZIndentTransform.cs`, hooked in `ZLSLTokenizer.Tokenize` |
| JSON literals, heredocs, number literals | `ZLSLTokenizer.cs` (`MarkJson`, `TryReadHeredoc`, `ReadNumber`) and `ZIncludeResolver.cs` (keeps the preprocessor out of heredoc bodies) |
| `def` `on` `let` `defer` `assert` `print` `in`, defaults, comprehension, chained compare | `ZLSLParser.cs` (plus `ZTypeInfer.cs` / `ZCILCodeGen.cs` for `let` and `in`) |
| desugarer | `ZLSLDesugar.cs`, and `ZLSLDesugarFlatten.cs` for nested `->` / `await` |
| tests | `Shared/Tests/ZSugarTest.cs`, `ZSugarRunTest.cs`, `ZDesugarTest.cs`; scripts in `tasks/desugar_suite/` and `tasks/sugar_showcase*.lsl`; the working list in `tasks/sugar-todo.md` |

## Running it

```text
zengine run   tasks/desugar_suite/sugar_01_indent_braceless.lsl --touch
zengine desugar tasks/sugar_showcase.lsl --out showcase.stock.lsl
zengine desugarcheck --all               # original vs lowered, every suite script
dotnet test tests/OpenSim.Tests/OpenSim.Tests.csproj --filter "TestDesugarSuiteIsEquivalent|ZSugar"
```

On a Windows dev box: build `OpenSim.sln` (not only the test project) before scene-based tests, and run `prebuild` from PowerShell (Git Bash rewrites `/target`). See `CLAUDE.md` in the repo.

## What changed in older notes

[[ZEXT Language Extensions]] (§4 and the corrections list), [[ZEXT Constructs - One Example Each]], [[ZEXT Sugar and Async - Worked Examples]], [[Known and Unresolved Problems]] and [[CraftSim]] said the desugarer refused nested `->`, string `%`/`*`, and left a dangling named-timer call; those statements are corrected and point here.
