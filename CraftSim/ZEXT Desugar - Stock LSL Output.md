---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - desugar
  - reference
aliases:
  - zengine desugar
  - desugar
  - stock LSL output
---

# ZEXT Desugar: turning ZEngine LSL back into stock LSL

← Back to [[ZEXT Syntax Sugar Index]] | Reference: [[ZEXT Language Extensions]] (§4 is the older summary, superseded by this page) | Engine: [[ZEngine]] | Evidence: [[ZEXT Desugar Verification Report]]

`zengine desugar <script-file> [--out <file>]` rewrites a ZEXT script into **stock LSL**: no ternary, no `->`, no f-strings, no braceless blocks, nothing that needs `[ZEngine] ZEXT = true`. It exists so a script written with the sugar can still run on an engine that only knows LSL, and so you can *read* what the sugar means. `zengine desugarcheck <file> | --all` runs original and lowered through the same throwaway-prim harness and diffs what they say.

> [!NOTE] What "stock LSL" is checked against
> The lowered text is recompiled by ZEngine itself with `ZEXT = false` (the gate that rejects every ZEXT construct). It has **not** been fed to an independent LSL compiler (Second Life, YEngine). The only non-LSL function it relies on is `llOrd` (present in OpenSim/ZEngine), and only for string `case` ranges.

---

## 1. The pipeline

```text
source ──► tokenizer ──► parser (ZEXT on) ──► inline async functions ──► type inference
          · #indent → braces                  · closures → helper + event branch
          · elif/unless/until                 · state arguments → globals + loader
          · j"" heredoc literals              · timer -> / named timers → timer() handler
                                              · defer / default args / comprehension / let / in … → plain AST
       ──► hoist ?:-initialised globals ──► async lowering (straight-line split or flatten) ──► emit + helpers
```

Most sugar is already plain AST after parsing, so the emitter needs nothing for it. What the emitter and the async passes add is below. The emitter parenthesises aggressively (generated source, unambiguous beats pretty) and **drops comments and formatting** (the file says so in its header).

## 2. What each construct becomes

| ZEXT construct | Stock LSL |
| :--- | :--- |
| `#indent`, `elif`, `unless`, `until`, `def`, `on` | ordinary braces / `else if` / negated conditions / plain declarations (done in the tokenizer and parser) |
| default and named arguments | the omitted arguments are written out at every call |
| `let x = e` | `T x = e;` with the inferred type |
| `defer s` | `s` copied before every exit of the scope (block end, `return`, `state`, `break`, `continue`) |
| `assert c, m` / `print(a, b)` | `if (!(c)) llOwnerSay("assertion failed at line N: m");` / `llOwnerSay((string)a + " " + (string)b);` |
| `a < b <= c` | `(a < b) && (b <= c)` |
| `x in list` / `x in str` | `llListFindList(list, [x]) != -1` / `llSubStringIndex(str, x) != -1` |
| `[e for T v in s if c]` | `list r = []; foreach … { if (c) r = r + [e]; }` (in LSL: an indexed `for`) |
| `0b101`, `0o17`, `1_000` | decimal literal |
| `j"…"`, `j<< ID`, `f"…"`, `r"…"`, `"""…"""`, `'…'`, `<< ID`, `"a" "b"` | one ordinary quoted string / a `+` chain of `(string)` casts |
| `["k" => v]` | `["k", v]` |
| `s[i]`, `s[a:b]`, `l[i]`, `l[a:b]` | `llGetSubString(s, i, i)`, `llGetSubString(s, a, b)`, `llList2<T>(l, i)`, `llList2List(l, a, b)` (the accessor comes from the inferred type) |
| `"ab" * n`, `"fmt" % [args]` | calls to the generated `__zrepeat` / `__zfmt` (helpers are emitted first) |
| `c ? a : b` | a temporary and `if (c) t = a; else t = b;` placed before the statement |
| `a ?? b` | the same ternary (missing = `""`, `NULL_KEY` or the JSON-invalid sentinel, tested before a cast) |
| `x \|> f(a)` | `f(x, a)` |
| `x \|= y` … `>>=` | `x = (x \| y)` |
| `boolean`, `true`, `false` | `integer`, `1`, `0`; a boolean that meets a string prints `TRUE`/`FALSE` (`__zbs…` temp) |
| `switch` / `case a..b` | a jump-dispatch chain with labels in source order, so fall-through is preserved; a string range uses `__zscmp` |
| `break` / `continue` | `jump` to a label after / at the end of the loop body |
| `foreach (T v in l)` | an indexed `for` over a copy of the list |
| closures (`llSetTimerEvent`, `llListen`, `llSensor`) | already a helper function plus an `if (cb_id == n)` branch of the event (parser) |
| `state worker(args)` | arguments stored in globals, loaded at the top of `state_entry()` of the target (parser) |
| `->`, `await` directly in the handler body | straight-line split: `__zstate` + synthesized `dataserver` / `http_response` / `timer` handlers |
| `->`, `await` in `if` / loops / `switch` / functions | **flattened dispatch function** (see [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]]) |
| `timer -> N:`, `timer "x" -> N:` | timer() handler; named timers keep a due time per name |
| `?:` in a global initialiser | the global is assigned once, behind `__zginit`, at the start of the default state's `state_entry` |

### Conditions with `?:` in loop headers and `&&` / `||`

A `?:` needs statements, but a `while` condition or a `for` update runs on every pass, so the loop is rebuilt instead of refused:

```text
while (c ? a : b) body        →   while (1) { <prolog>; if (!(cond)) jump brk; body }   // continue lands at the end of body
do body while (c ? a : b);    →   while (1) { body; @cnt; <prolog>; if (!(cond)) jump brk; }
for (i; c ? a : b; u) body    →   i; while (1) { <prolog>; if (!(cond)) jump brk; { body @cnt; } u }
x && (c ? a : b)              →   integer t; if (x) { <prolog>; if (r) t = 1; else t = 0; } else t = 0;   // short circuit kept
```

## 3. Names the output introduces

| Name | Where | Meaning |
| :--- | :--- | :--- |
| `__zrepeat`, `__zfmt`, `__zscmp` | top of file | helper functions (only emitted when used) |
| `__zte<n>`, `__zsc<n>`, `__zbs<n>` | locals | ternary result, short-circuit result, boolean-as-string temp |
| `__zswt<n>`, `__zswc<n>_<i>`, `__zswend<n>` | locals / labels | switch test value, case labels, end label |
| `__zcnt<n>`, `__zbrk<n>` | labels | continue / break targets |
| `__zstate` | global | which async request the script is waiting on (0 = none) |
| `__zreq_<K>`, `__zreq_<K>_<i>` | globals | the request key of async op K (and slot i of an `await`) |
| `__zgot_<K>_<i>` | globals | "slot i has arrived" flags of an `await` |
| `__zl<n>_<name>` | globals | a local / parameter / bound name that lives across a suspension |
| `__zrun<n>_<event>(integer __zpc)` | function | the flattened handler body: a `while (1)` over `if (__zpc == block)` arms |
| `__zdisp<n>` | label | end of that loop body (every jump back into the dispatcher) |
| `__zpend` | global | a `state x;` requested from inside a flattened body (the caller performs it) |
| `__zsw<n>` | global | switch test value inside a flattened body |
| `__zginit` | global | "global initialisers already run" flag |
| `__ztimer_<name>_int`, `_next`, `__ztimer_tick` | globals | named-timer interval, due time, and the finest tick armed |
| `__z_call_timer`, `__z_call_listen`, `__z_call_sensor`, `__ztimer_cb`, `__zlisten_cb`, `__zsensor_cb` | functions / globals | closure expansion |
| `__zstate_<state>_arg_<i>` | globals | state arguments |

## 4. Differences from the native behaviour (known and accepted)

1. **Named timers are polled.** Stock LSL has one timer, so the lowered form wakes at the finest named interval armed (`__ztimer_tick`) and fires each name when its due time passes (due times advance on a fixed grid). A timer whose interval is not a multiple of the finest one fires up to one tick late. An explicit `llSetTimerEvent` in the script competes with this single timer.
2. **A waiting handler drops new triggers** of itself (`if (__zstate == 0)`); native `->` leaves them queued. Event parameters become globals when the handler is entered, so `llDetected*` is only valid before the first suspension.
3. **An exception inside an API call** (e.g. the null reference the unit-test prim raises for a missing notecard) surfaces at a different point in the lowered form, because the call now sits in a generated function. The scripts that showed this in the corpus sweep already fail natively. Scripts that do not throw are unaffected.
4. **`%d` / casts of strings use LSL rules in the output** (`(integer)"12abc"` is 12), while ZEngine's own string-to-integer helper is strict (0). `"%d" % ["12abc"]` therefore differs.
5. **String ranges compare by code point** (`llOrd`); .NET compares UTF-16 units, which only differs for characters outside the Basic Multilingual Plane.
6. **Comments and formatting are not preserved.**

## 5. Still refused (an error, never an approximation)

- the `decimal(p,s)` type (no fixed-point type in LSL; mapping it to `float` would change rounding);
- a closure anywhere except the last argument of `llSetTimerEvent` / `llListen` / `llSensor` (ZEngine rejects it too);
- recursion through a function that suspends (the inliner cannot expand it).

Everything else in this family was refused at some point before 2026-10-22 and no longer is: `?:` in loop headers, `->` and `await` inside `if` / loops / `switch` / functions, string `*` and `%`, string `case` ranges, `?:` in a global initialiser.

## 6. A complete example

The showcase script uses nearly every construct at once; the source and the generated stock LSL are in [[ZEXT Sugar Showcase - Annotated]] and in the repo as `tasks/sugar_showcase.lsl` / `tasks/sugar_showcase.desugared.lsl` (kept in sync and compared by a test).

Regenerating a listing: `dotnet test tests/OpenSim.Tests/OpenSim.Tests.csproj --filter RegenerateSugarShowcase`; dumping desugared text and both run outputs for any files: set `DUMP_DIR` (and optionally `DUMP_FILES`) and run the `DumpDesugarAndRun` test.
