---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
---

# ZEXT Language Extensions

← Back to [[CraftSim]] | Related: [[ZEngine]], [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[MCP Server & Live Diagnostics]], [[End-to-End Multi-Module LSL Architectures]]

> [!NOTE] Verified against code (commit `9fea6a0332`, 2026-10-16)
> Checked against `ZLSLParser.cs`, `ZLSLTokenizer.cs`, `ZTypeInfer.cs`, `ZIncludeResolver.cs`, `ZFunctionInliner.cs`, `ZLSLDesugar.cs`, `ZEngineInstance.cs`, `ZEngine.cs`, the `ZEngineZextTest` suite and `tasks/desugar_suite/`. Unverified items are marked.

**ZEXT** is the set of non-stock-LSL syntax ZEngine accepts. `[ZEngine] ZEXT = true` (C# default `ZEngine.cs:275`, also `bin/OpenSim.ini` `ZEXT = true`) allows everything; `ZEXT = false` is the *portability gate*: the constructs in table 1 become compile errors of the form
`line L:C: <what> is a ZEngine extension, not standard LSL (set [ZEngine] ZEXT = true to allow it)` (`ZLSLParser.cs:29-30`). The flag is passed `ZEngine.m_allowExtensions` → `ZIncludeResolver`, `ZLSLParser`, `ZTypeInfer`, `ZCILCodeGen`.

---

## 1. ZEXT-gated constructs (error when `ZEXT = false`)

Each row cites the parser site that raises the error.

| Construct | Example | Error text / source |
| :--- | :--- | :--- |
| `boolean` / `decimal` types | `boolean ok = TRUE;`, `decimal price = 1.25;` | `the 'boolean' type` / `the 'decimal' type` (`ZLSLParser.cs:1599`) |
| Lowercase literals | `integer ok = true;` | `lowercase 'true' (use TRUE)` / `'false' (use FALSE)` (`:1427,1434`) |
| Ternary | `m = a > b ? a : b;` | `the ternary '?:' operator` (`:973`) |
| Null-coalescing | `string n = name ?? "guest";` | `the null-coalescing '??' operator` (`:989`). Lowered to `cond ? lhs : rhs` where "missing" = not `""`, not `NULL_KEY` (`00000000-…`), not the JSON_INVALID sentinel; the test runs on the pre-cast value so `(integer)llJsonGetValue(..) ?? 100` works (`:993-1006`) |
| Pipeline | `text \|> llToUpper()` | `the pipeline '\|>' operator` (`:950`). RHS must be a call; LHS is inserted as **first** argument (`:957-960`) |
| Compound bitwise/shift assign | `flags \|= 4; x <<= 1;` (`&= \|= ^= <<= >>=`) | `compound assignment '<op>'` (`:933`, `IsExtAssignOp`) |
| Index | `s[0]`, `s[-1]`, `list[i]` | `index '[i]' syntax` (`:1199`) |
| Range slice | `s[0:3]`, `s[1:]`, `s[:2]` | `range slice '[i:j]' syntax` (`:1192`). Missing start → 0, missing stop → `int.MaxValue`. **Stop is inclusive** for strings and lists (fixed 2026-09-18 per CLAUDE.md: `"abcdefgh"[0:3]` = `"abcd"`; `ZCILHelpers.StrSlice` mirrors `llGetSubString`) |
| Map literal | `["k" => v]` | `the map '=>' arrow syntax` (`:1501`); becomes a strided list |
| Adjacent string literals | `"ab" "cd"` | `adjacent string literal concatenation` (`:1572`) |
| Python-style strings | `r"C:\temp"`, `"""multi"""`, `f"hi {name}"`, `rf"…"` | message comes from the token (`Token.IsExt/ExtDesc`, `:1419`) |
| **String `%` formatting** *(missing from old doc)* | `"x=%d" % [n]` | `'%' string formatting is a ZEngine extension` (`ZTypeInfer.cs:439`); with ZEXT on, RHS must be a `list` |
| **String repetition `str * n`** *(missing from old doc)* | `"ab" * 3` | `string repetition 'str * n' is a ZEngine extension` (`ZTypeInfer.cs:449`); with ZEXT on, needs an integer multiplier |
| State parameters / arguments | `state run(integer speed) {…}`, `state run(5);` | `State parameters` (`:75`), `State arguments` (`:230`) |
| **Any preprocessor line** | `#define`, `#include`, `#if`, `#TRON`, `#STEP`… | `the preprocessor ('#…') is a ZEngine extension` (`ZIncludeResolver.cs:378-392`); `#include` additionally needs `[ZEngine] EnableInclude = true` |

Covered by `OpenSim/Region/ScriptEngine/Shared/Tests/ZEngineZextTest.cs` (boolean/decimal/ternary/lowercase/compound/index tests) and `ZIncludeResolverZextTest.cs`. Whether every row above has its own test is **(unverified)**.

### Always available (not gated by ZEXT)
`switch/case/default`, `break`, `continue` (contextual keywords, `ZLSLParser.cs:~1687`), `foreach (type x in list)` (`KW_FOREACH`/`KW_IN` are real keywords in `ZLSLTokenizer.cs:18,88`, so `foreach`/`in` cannot be identifiers), list destructuring `[a, b] = expr;` (`IsListDestructuring`, `:302`), closure syntax `(type p) => …` / `() => …` (`ParseClosure`, `:1139,1148`; semantics **(unverified)**), the async-bind arrow `f() -> x`, `await { … }`, and `timer -> N: { … }` / `timer "name" -> N: { … }` (`:269-273`). Use `zengine desugar` (section 4) to lower all of these to stock LSL where possible.

---

## 2. Asynchronous programming

### 2.1 Async bind `->`
Parsed in `ParseAsyncBind` (`ZLSLParser.cs:624`). Left side **must be a function call** (error: *left of '->' must be a function call, e.g. osMLpredictAsync(...) -> result;*). Bindings:
- single: `-> name` or `-> type name` (typed form allowed);
- multi: `-> (a, b)` or `-> [a, b]` (interchangeable) — the payload is CSV-split in order;
- optional `: <timeoutExpr>` and optional `else <stmt>` run when the timeout elapses (`:672-679`).

```lsl
llHTTPRequest("http://example.com", [], "") -> string body : 5.0 else {
    llOwnerSay("HTTP request timed out!");
    return;
}
```
Runtime (`ZEngineInstance.cs`): `ZIROp.AsyncYield` ends the block exactly like `Yield`; `SuspendKind.AsyncAwait` resumes via **`TryConsumeMatchingEvent`** (`:558`) — the old doc's name `TryConsumeMatchingDataserverEvent` does not exist. It scans the event queue and removes **only** the event whose name equals `m_pendingAsyncEvent` (default `"dataserver"`) and whose first parameter equals the pending request key, re-enqueueing everything else. It is **not dataserver-only**: `http_response` and `listen` are supported (for `listen`, the key is `listen:<channel>:<avatar>` and the payload is `Params[3]`; an implicit listen handle is removed on match — `:503-575`). Fan-out uses `TryConsumeMatchingFanOutEvents` (`:487`). Bound names are strings.

`->` works in event bodies and inside user functions because `ZFunctionInliner` inlines such functions ([[ZEngine]] §2). Recursive suspending functions are a compile error.

### 2.2 `await { … }`
`ParseAsyncFanOut` (`:273`): the block contains several `call -> name;` statements, all fired at once; resumes when all slots are satisfied or the `: timeout else {…}` fires (`ZIROp.AsyncFanOutYield/AsyncFanOutBindLoad`).
```lsl
await {
    osAIChatAsync(model, p1, 200, []) -> r1;
    osAIChatAsync(model, p2, 200, []) -> r2;
} : 15.0 else { llOwnerSay("timeout"); }
```

### 2.3 Inline and named timers
- `timer -> 1.0: { … }` (`ParseTimerInline`, `:269`) emits `llSetTimerEvent(1.0)` at the statement and hoists the body into the state's `timer()`; several unnamed bodies append in source order; **mixing an unnamed inline timer with an explicit `timer()` handler in the same state is a compile error**.
- `timer "name" -> N: { … }` hoists the body into a synthesized `__ztimer_<name>` handler armed by compiler-internal `__zSetNamedTimer`; named timers **do** coexist with an explicit `timer()` and with each other at independent cadences, scheduled by `ZEngine.m_namedTimerHeap` (`ZEngine.cs:167`). Intervals are clamped by `MinTimerInterval` (0.5 s code default, 0.01 in `bin/OpenSim.ini`). Named-timer state is persisted in `.state.json` (`namedTimers`).
- Inline timers outside event/state scope are rejected (`RejectInlineTimer`, `ZLSLParser.cs:58,562`).

---

## 3. Preprocessor and interactive debugging

- **Directives** (`ZIncludeResolver.cs`, 1027 lines): `#include "name"` (prim inventory), `#include <asset-uuid>` (creator must be the script owner, or in `IncludeTrustedCreators`), `#define`/`#undef` (object-like and function-like macros, `#`/`##`; `ZMacroExpander`), `#if`/`#ifdef`/`#ifndef`/`#else`/`#endif` (`ZConditionalEvaluator`). Includes resolve recursively. Inspect with `zengine include show <asset-uuid>` or `zengine include show <prim-uuid> <item-name>`. A reset of a script that uses `#include` re-resolves and recompiles it (old version kept on failure).
- **`#TRON` / `#TROFF`** — paired regions; both directives take no arguments (`ZIncludeResolver.cs:535,548` error "directive takes no arguments"). `ZTraceInjector` wraps each statement with `llOwnerSay` of `script:line  <source text> [var=value…]` and emits `script:line  EVENT name(param=value…)` as the first statement of each handler. Test: `ZTraceInjectorTest`.
- **`#STEP` / `#STOP`** — paired region; `ZStepInjector` inserts `__zCheckpoint(line, depth, text)` before statements. At a hit, the owner gets an `llDialog` with buttons `STEP, NEXT, GLOB, LOCAL, RESUME, HELP, CMD` (`ZEngineInstance.cs:1211`) on a private channel `-1000000 - ((uint)ItemID.GetHashCode() % 900000)` (`:1199`). Chat commands on that channel: `p <var>`, `set <var>=<val>`, `listbreaks`, `deletebreak <line>` (`:1294-1350`). STEP pauses at every checkpoint including inlined calls; NEXT steps over inlined calls (depth tracking); RESUME runs the rest of the invocation; CMD arms a one-shot channel-0 listen. Dynamic breakpoints from outside the script are possible via `extraBreakLines` (see `ScriptDebugTools` in [[MCP Server & Live Diagnostics]]).

---

## 4. Desugaring to stock LSL (`ZLSLDesugar.cs`)

```text
zengine desugar <script-file> [--out <file>]  |  zengine desugar --heredoc [--out <file>]
zengine desugarcheck <script-file> [--touch] [--say <ch> <msg>] [--wait <ms>] [--verbose]
zengine desugarcheck --all [dir ...] [--wait <ms>] [--verbose]
```
Output is **generated source; comments/formatting are not preserved**. The result is stock LSL, not YEngine-flavoured (ternary and bitwise compound assignment are also expanded). Lowering rules (checked in `ZLSLDesugar.cs`):

| ZEXT construct | Lowering |
| :--- | :--- |
| `boolean` | `integer` |
| `c ? a : b` | temp `__zte<n>` + `if`/`else` (`:591`) |
| `a ?? b`, f-strings, raw/triple strings, adjacent strings, `foreach`, `=>` maps | already normalised to ordinary AST by the parser, so they appear as plain LSL |
| `x \|> f(args)` | `f(x, args)` (parser already inserted the arg) |
| `x \|= y` etc. | `x = x \| y` |
| `s[i]` / `s[a:b]` | `llGetSubString(s, i, i)` / `llGetSubString(s, a, b)`; list forms use `llList2<Type>` / `llList2List` (needs type inference first; inliner is deliberately **not** run) |
| loops with `break`/`continue` | labels + `jump` (continue label at end of body) |
| `switch/case/default` | jump-dispatch chain + labels in source order (preserves fall-through) |
| `->`, `await {}`, named/inline timers | **Level-2 CPS state machine**: global `__zstate`, dispatcher + synthesized `dataserver`, `http_response`, `timer` handlers. **Only top-level statements of an event-handler body**: errors for async-bind inside a user function, a loop, or an `if`/`else` (`CheckIllegalAsyncInStmt`, `ZLSLDesugar.cs:~746-760`). Hard errors rather than approximations |

`desugarcheck` runs original and lowered scripts through the same throwaway-prim harness as `zengine run` and diffs say/shout/whisper output after masking UUIDs, ISO-8601 timestamps and timing figures. Test corpus: `tasks/desugar_suite/` (`break_continue_nested_loops.lsl`, `foreach_ternary.lsl`, `fstrings_python_strings.lsl`, `index_expr.lsl`, `level2_asyncbind_http.lsl`, `level2_asyncbind_timeout.lsl`, `level2_named_timer.lsl`, `string_list_slicing.lsl`, …) and `ZDesugarTest` (behavioural-equivalence assertions). CLAUDE.md reports 57 corpus files desugar and recompile with `ZEXT=false` and `ZDesugarTest` has 18 tests; I did not re-run them **(unverified)**.

Because the whole `os*` API is engine-agnostic (YEngine binds it by reflection), only ZEXT *syntax* ties a script to ZEngine; a desugared script should run on stock-LSL engines.

---

## Corrections vs previous doc

- **Missing gated constructs**: `"fmt" % [list]` string formatting and `str * n` repetition are ZEXT-gated (`ZTypeInfer.cs:431-452`) but were not in the table.
- **`TryConsumeMatchingDataserverEvent`** does not exist; the method is `TryConsumeMatchingEvent`, and selective resume handles `dataserver`, `http_response` **and** `listen` (not only dataserver).
- The "always-available" list was incomplete: list destructuring `[a,b] = …` and closure syntax exist; `foreach`/`in` are hard keywords.
- Debug channel formula in the old note (`-1000000 - ItemID.GetHashCode() % 900000`) omitted the `(uint)` cast used in code (`ZEngineInstance.cs:1199`); button list now exact (`HELP` was missing).
- `#TRON`/`#TROFF`/`#STEP`/`#STOP` take no arguments (explicit error), and **any** `#` line is rejected with ZEXT off (not only `#define`/`#include`).
- Desugarer: CPS lowering also rejects async-bind inside `if`/`else` (previous note said loops/functions/if all rejected — consistent — but the exact error sources were not cited); `??`, f-strings, maps and `foreach` never reach the desugarer as distinct nodes.
- "Null-coalescing → empty check (blank string, NULL_KEY, or sentinel)" confirmed and made precise (pre-cast operand, `JSON_INVALID` sentinel).
- Tuple destructuring example `osDbFindVectorNearestAsync(...)` was invalid: no such function exists (only `osDbFindVectorNearest`, `…Scores`, `…ByJson`, all synchronous). Use a real async function such as `osDbFindAsync` / `osMLpredictAsync` with `-> (a, b)`.
