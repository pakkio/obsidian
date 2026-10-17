---
tags:
  - craftsim
  - zengine
  - compiler
  - lsl
  - cil
---

# ZEngine

← Back to [[CraftSim]] | Related: [[ZEXT Language Extensions]], [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[Performance & Benchmarks]], [[MCP Server & Live Diagnostics]], [[Build, Configuration & Self-Hosting]]

> [!NOTE] Verified against code (commit `9fea6a0332`, 2026-10-16)
> Every claim below was re-checked against `OpenSim/Region/ScriptEngine/ZEngine/` (20 055 lines in 20 `.cs` files) and `bin/OpenSim.ini`. Line numbers are for that commit. Anything not checked is marked **(unverified)**.

**ZEngine** (`OpenSim/Region/ScriptEngine/ZEngine/`, class `ZEngine` in `ZEngine.cs`, per-script `ZEngineInstance` in `ZEngineInstance.cs`) is the script engine CraftSim ships as default (`bin/OpenSim.ini:284` `DefaultScriptEngine = "ZEngine"`; the upstream fallback in `bin/OpenSim.ini.example:275` is still `YEngine`). It compiles LSL straight to **.NET CIL** with `Reflection.Emit` (`ZCILCodeGen.cs`, 4029 lines); the JIT turns that into native code. There is **no bytecode interpreter**: `ZEngineInstance.CompileSourceUncached` (`ZEngineInstance.cs:2960-3005`) is commented "CIL path - the only execution path", and a CIL codegen exception becomes a compile error (the script does not run).

---

## 1. Source layout

| File | Lines | Role |
| :--- | ---: | :--- |
| `ZEngine.cs` | 2699 | Region module (`INonSharedRegionModule`, `IScriptModule`): config, console commands, compile cache, timer/sleep heaps, worker scheduler, checkpointing |
| `ZEngineInstance.cs` | 3029 | One per script item: event queue, suspend/resume (`SuspendKind`), debugger dialog, state save/restore, self-reset brake |
| `ZIncludeResolver.cs` | 1027 | Preprocessor: `#include/#define/#if…`, `#TRON/#TROFF/#STEP/#STOP` rewriting |
| `ZMacroExpander.cs` | 578 | `#define` object/function-like macro expansion (incl. `#`/`##`) |
| `ZLSLTokenizer.cs` / `ZLSLParser.cs` / `ZLSLAST.cs` | 642 / 1711 / 180 | Lexer, parser (ZEXT gate lives here), AST |
| `ZTraceInjector.cs` / `ZStepInjector.cs` | 224 / 136 | `#TRON` tracing, `#STEP` checkpoints |
| `ZFunctionInliner.cs` | 1134 | Inlines suspending user functions; loop rotation |
| `ZTypeInfer.cs` | 572 | Type inference/checking |
| `ZIRBuilder.cs` / `ZIRCompiler.cs` | 85 / 433 | ZIR ops + block splitting |
| `ZCILCodeGen.cs` / `ZCILHelpers.cs` | 4029 / 733 | CIL emission, runtime helpers |
| `ZLSLDesugar.cs` | 1434 | ZEXT → stock LSL lowering (`zengine desugar`) |
| `ZApiRegistry.cs`, `ZByteCode.cs`, `ZVM.cs`, `ZConditionalEvaluator.cs`, `ZCompileErrorFormatter.cs` | 494/363/114/375/63 | API reflection registry, `ZExecutionContext`/`ZSleepException`, value types, `#if` evaluator, error formatting |

---

## 2. Architecture and compilation pipeline

Real order (from `ZEngine.cs` → `ZEngineInstance.CompileWithCache` → `CompileSourceUncached` → `ZCILCodeGen.Compile`):

1. **Preprocess** (`ZIncludeResolver`, built at `ZEngine.cs:~807`/`1594`/`1817`/`2141`): resolves `#include "name"` (prim inventory) and `#include <asset-uuid>` (needs creator in `IncludeTrustedCreators` unless you created it), `#define`/`#undef`/`#if`/`#ifdef`/`#ifndef`/`#else`/`#endif` (macros via `ZMacroExpander`), and rewrites `#TRON/#TROFF/#STEP/#STOP` into line-preserving sentinel comments. With `ZEXT=false`, *any* line starting with `#` is a compile error (`ZIncludeResolver.cs:378-392`).
2. **Tokenize + parse** (`ZLSLTokenizer`, `ZLSLParser(tokens, allowExtensions)`, `ZEngineInstance.cs:2965-2966`). Parser errors abort; a script without `default` state, or with an event not in `s_knownEvents` (other than synthesized `__ztimer_<name>` handlers), is rejected. Sugar is normalised into core AST here (f-strings, `foreach`, `["k" => v]`, `timer -> N:` hoisting). `switch/break/continue` are contextual keywords (`ZLSLParser.cs:~1687`) so legacy `integer switch = 0;` still parses.
3. **`ZTraceInjector.Inject`** (`ZCILCodeGen.cs:69`) then **`ZStepInjector.Inject`** (`:76`) — before the inliner, so injected nodes go through the same passes as hand-written code. Trace statements are synthetic `llOwnerSay` calls formatted `script:line  <text>` and `script:line  EVENT name(param=value…)` (`ZTraceInjector.cs:167-215`). Step injection inserts `__zCheckpoint(line, depth, text)` (`ZStepInjector.CheckpointMarker`).
4. **`ZFunctionInliner`** (`ZCILCodeGen.cs:111`): `ComputeAsyncFuncs` (`ZFunctionInliner.cs:82`) is a fixpoint over user functions that contain `->`/`await{}` or call such a function; those are inlined at their call sites (the same closure covers `llSleep`, `m_sleepFuncs`). `if` conditions with a suspending call get a prolog; `while`/`do-while`/`for` conditions are **rotated** (`RotateWhile :526`, `RotateDoWhile :532`, `RotateFor :544`) using a synthesized first-iteration flag `__lf<n>` (`:534,:557`) so `continue` keeps exact semantics; `a && susp()` / `a || susp()` are lowered to explicit `if`. After inlining, the now-unreachable function declarations are removed from `prog.Functions` (otherwise an `AsyncYield` would land in an unsplit plain method).
5. **`ZTypeInfer`** (`ZCILCodeGen.cs:131`): types, implicit-conversion errors, ternary unification (a `list` branch forces the scalar branch to be boxed into a one-element list), decimal-vs-float mixing errors, and the ZEXT-gated string operators `"fmt" % [..]` and `str * n` (`ZTypeInfer.cs:431-452`).
6. **`ZIRCompiler`** (`ZCILCodeGen.cs:353`) splits each event handler into basic blocks. `ZIROp` (`ZIRBuilder.cs:6-22`): `Nop, EvalExpr, StoreLocal, StoreGlobal, Jump, JumpIfFalse, Yield (llSleep), Checkpoint (#STEP), AsyncYield, AsyncBindLoad, AsyncFanOutYield, AsyncFanOutBindLoad, Return, StateChange`. **`ZCILCodeGen`** emits one CIL method per block, returning the next block id; `RunBlocks` in `ZEngineInstance` drives them.

### Recursion (corrected)
Only **suspending** recursion is a compile error (`ZFunctionInliner.cs:636,1089`: *"recursive call to 'f', which suspends (llSleep or '->')"*). Ordinary recursive user functions compile and run; the runtime guards them with `ZRecursionLimitException` → shout `recursion limit exceeded (max 128 user-function calls per event)` (`ZEngineInstance.cs:1805-1807`).

### Runaway protection
`MaxEventSliceSeconds` is checked **between ZIR blocks**; the source comment (`ZEngineInstance.cs:614-616`) states a tight CIL loop *inside one block* (a user-function loop with no sleep/state change) is still unbounded.

---

## 3. Cooperative multitasking and `llSleep`

- `ZCILCodeGen.EmitFastApiCall` (`ZCILCodeGen.cs:3545`, `llSleep` case `:3560-3581`) compiles `llSleep(x)` to `CurrentInstance.BeginSleep(x)` + `Stsfld ZExecutionContext.YieldRequested = true` (`ZByteCode.cs:14`, `[ThreadStatic]`). `ZIROp.Yield` does the same at block level (`ZCILCodeGen.cs:2031-2038`). `RunBlocks` sees the flag and suspends; `TimerLoop` re-enqueues the instance when the deadline passes.
- `llSetTimerEvent` is also a fast-path call (`SetTimer`).
- The `throw new ZSleepException` in `ZApiRegistry.cs:184` is only reachable when an `llSleep` goes through `registry.Invoke` in CIL mode (uninlined path); `EmitFastApiCall` pre-empts it for any call with ≥1 argument. `ZEngineInstance.cs:1800` still catches it.
- **~6 ns/op vs ~3,100–3,200 ns/op (≈500×)** is from `CLAUDE.md`; I did not reproduce it **(unverified)**. `tasks/ZEngineRuntimeBench/Program.cs` is the dispatch micro-benchmark (1 000 000 iterations, 20 000 warm-up), not an exception-vs-flag comparison.
- **Scheduler**: a process-wide `GlobalWorkerScheduler` (`ZEngine.cs:2646`) with `ConcurrentQueue` + `SemaphoreSlim`; `EnsureWorkerCount(NumWorkers)` spawns threads `ZEngine-Worker-<n>`. One `ZEngine-Timer` thread runs `TimerLoop` (`:2461`) over three lazy-deletion `PriorityQueue` heaps: `m_timerHeap`, `m_namedTimerHeap` (`(ZEngineInstance, string)` keys), `m_sleepHeap` (`:160-167`); it waits until the earliest deadline (woken by `m_timerWake` on every push) and drains in timer → named → sleep order. Heaps are compacted when they exceed `max(1024, live*4+64)` (`:223`).
- **Worker quantum**: `MaxWorkerQuantumMilliseconds` (default 5) yields at block boundaries (`SuspendKind.Quantum`); `SuspendKind` is `{Sleep, Checkpoint, Quantum, AsyncAwait}` (`ZEngineInstance.cs:264`).

---

## 4. Compile cache (`m_compileCache`)

- `ConcurrentDictionary<string, CompileCacheEntry>` (`ZEngine.cs:113`), key = **SHA-256 hex of the resolved source** (`ComputeSourceHash`, `ZEngineInstance.cs:2948-2950`). A dynamic-breakpoint compile appends `// __zdynbreak:<lines>` to the hashed text so it gets its own entry (`:2902`).
- `CompileCacheEntry` (`ZEngineInstance.cs:3017`): `Key`, `Lazy<ZCompiledScript>`, `Refcount`. Instances hold the entry by direct reference (`BoundCacheEntry`), so eviction never invalidates a live script.
- `ReleaseCompileCacheEntry` (`ZEngine.cs:~1755`): `Interlocked.Decrement`; at 0 it calls `((ICollection<KeyValuePair<…>>)m_compileCache).Remove(new KeyValuePair(key, entry))` — an atomic remove-only-if-same-object. Wired to script removal, recompile (`TryRecompile`, `oldEntry`) and reset-with-include-refresh.
- `zengine run <file> --copies N` (max 500) reports how many compiled scripts the N prims share (expected: 1). Covered by `ZResetLoopTest`.

---

## 5. `llResetScript()` loop brake

`ZEngine.ResetInstance` (`ZEngine.cs:~1790`): `selfReset = ZExecutionContext.CurrentInstance == inst`; after `inst.Reset()`, if `selfReset && MaxSelfResetsPerSecond > 0 && inst.RegisterSelfReset(now, max, delay)` → `HoldAfterReset(delay)` (re-enqueued by the sleep heap) instead of `Enqueue`.

`RegisterSelfReset` (`ZEngineInstance.cs:1090-1120`): 1-second window counter; the `max+1`-th reset in a window sets `m_inResetLoop` and shouts once on the debug channel (*"llResetScript loop: the script reset itself more than N times in one second…"*). While in loop mode every self-reset returns true; the mode exits when the gap since the previous reset exceeds `2*delay+1` seconds. External resets (console, viewer, `llResetOtherScript`) are never braked. A plain `Reset()` calls `AllocTypedGlobals(rebuildApis: false)` (`:2100`), so APIs (and `LSL_Api.LoadConfig`) are not rebuilt. If the script used `#include`, a reset re-fetches and recompiles the source first (`TryRefreshIncludesOnReset`), keeping the old compiled version (with an in-world warning) if the new one fails.
Test: `ZResetLoopTest` (CLAUDE.md says `Test27CopiesWithoutBrakeSaturateWorkers` is load-sensitive: expects >2 busy cores, ~1.75 measured in a full-suite run, passes in isolation).

---

## 6. State persistence

- Format: `<ScriptCache>/<itemID>.state.json` (`m_cacheDir = ScriptEnginePath/ScriptCache`, `ZEngine.cs:314`; `SaveState` `ZEngineInstance.cs:2162-2167`, restore `:2474`). Contains globals, current state, suspend position, timers, `namedTimers`.
- Checkpointing runs on its **own thread** `ZEngine-Checkpoint` (BelowNormal priority, `ZEngine.cs:2372-2381`, `CheckpointLoop :2509`, `CheckpointStates`), every `StateCheckpointSeconds`; only instances that ran since their last checkpoint are rewritten (`TrySaveCheckpoint`). It is **not** a `Timer` object. `0` disables it; graceful shutdown still saves.

---

## 7. `[ZEngine]` configuration (`ZEngine.cs:269-300`, `bin/OpenSim.ini:1040-1080`)

The section falls back to `[YEngine]` if `[ZEngine]` is absent (`ZEngine.cs:267`).

| Key | C# default | `bin/OpenSim.ini` | Notes |
| :--- | :--- | :--- | :--- |
| `Enabled` | `true` | `true` | module on/off |
| `NumWorkers` | `4` | `4` | `GlobalWorkerScheduler` threads (process-wide, grows only) |
| `MinTimerInterval` | `0.5` (float s) | `0.01` | clamp for `llSetTimerEvent`/named timers; ini value is 10 ms |
| `MaxEventSliceSeconds` | `15` | `30` | raised so blocking `osAI*` calls can finish |
| `MaxWorkerQuantumMilliseconds` | `5` | `5` | `0` disables (→ `long.MaxValue`) |
| `MaxSelfResetsPerSecond` | `10` | *not set* (C# default) | `0` disables brake |
| `ResetLoopDelaySeconds` | `1.0` (min 0.1) | *not set* | |
| `StateCheckpointSeconds` | `60` | *not set* | `0` disables |
| `EnableInclude` | `false` | `true` | gates `#include` only |
| `IncludeTrustedCreators` | `""` | one estate-owner UUID `4e695ba5-e13c-42da-81e9-9a50f67a5b1b` | comma-separated; invalid tokens logged and ignored |
| `ZEXT` | `true` | `true` | see [[ZEXT Language Extensions]] |

Related: `[OSSL]` `Allow_*`/threat-level config in `bin/config-include/osslEnable.ini` (389 `Allow_os*` lines) decides which `os*` calls scripts may use; see [[OSSL Extensions (AI, ML, OsDb, OsQ)]].

---

## 8. Console commands (registered once, `ZEngine.cs:325-420`)

| Command | Purpose |
| :--- | :--- |
| `zengine include show <asset-uuid>` / `<prim-uuid> <item-name>` | print resolved `#include` chain with `>>> begin include <key> <<<` markers |
| `zengine run <file> [--touch] [--say <ch> <msg>] [--wait <ms>] [--copies <n>]` | compile & run on a throwaway non-persisted prim; `--wait` default 200 ms; `--copies` ≤ 500 |
| `zengine desugar <file> [--out <file>]` / `zengine desugar --heredoc` | lower ZEXT → stock LSL (`--heredoc` reads stdin until `//EOF`) |
| `zengine desugarcheck <file> [--touch] [--say…] [--wait] [--verbose]` / `--all [dir…]` | run original and desugared, diff output, PASS/FAIL (`--all` default dirs `tasks`,`scripts`, default wait 3000 ms, un-desugarable scripts are SKIPped) |
| `zengine vars [<item-uuid\|prim-uuid\|prim-name\|script-name>] [--all]`, `zengine show vars`, `show script vars` | dump globals/runtime state/memory metrics |

The MCP server exposes engine diagnostics too (`OpenSim/ApplicationPlugins/McpServer/Tools/ScriptEngineTools.cs`, `ScriptDebugTools.cs`, `ConsoleTools.cs`) — see [[MCP Server & Live Diagnostics]].

---

## 9. Gotchas verified in code

- **Per-script API host binding**: each `ZEngineInstance` owns `IScriptApi[] m_apis` (`ZEngineInstance.cs:103`, built lazily by `BuildApiSet`); never share one registry-level instance (that bug made every script act as the first prim's owner).
- **Exception paths differ**: typed fast path → exception propagates to `RunBlocks` catch-all (event ends, shout); `registry.Invoke` slow path swallows. `llDie`/`llRemoveInventory(self)` are caught as `SelfDeleteException` / `ScriptDeleteException` (`ZEngineInstance.cs:1810,1819`).
- **Async needs the right event admitted**: a script using `->` without a hand-written `dataserver` handler works only because `ZCILCodeGen.StmtContainsAsyncBind` adds the event name to `AnyStateEventNames` (from CLAUDE.md; code path not re-traced **(unverified)**).
- **The `os*` surface is engine-agnostic** (Shared API, bound to YEngine by reflection); only the syntax ties a script to ZEngine.
- Tests: `OpenSim/Region/ScriptEngine/Shared/Tests/Z*.cs` — `ZAsyncBindTest`, `ZConditionalCompilationTest`, `ZControlFlowTest`, `ZDesugarTest`, `ZEngineGlobalsTest`, `ZEnginePythonStringsTest`, `ZEngineStatePersistenceTest`, `ZEngineUpstreamLslFunctionsTest`, `ZEngineZextTest`, `ZIncludeResolverZextTest`, `ZIndexExprTest`, `ZLslCorpusCompileTest` (compiles every `.lsl` under `tasks/` and `scripts/`), `ZMacroExpanderTest`, `ZResetLoopTest`, `ZSuspendLoopTest`, `ZTraceInjectorTest`, `ZZeroArgUserFuncTest`.

---

## Corrections vs previous doc

- **"Direct or mutual script recursion remains a compile error"** — too broad. Only recursion across a *suspension* (`llSleep`/`->`) is rejected; plain recursion works up to 128 nested user-function calls per event.
- **`m_stateCheckpointTimer`** — does not exist; checkpointing is a dedicated thread (`ZEngine-Checkpoint`, `CheckpointLoop`).
- **Config table** claimed `MaxSelfResetsPerSecond`, `ResetLoopDelaySeconds`, `StateCheckpointSeconds` have explicit `bin/OpenSim.ini` values; they are **not present** in the ini (C# defaults apply).
- Trace/step injection is done inside `ZCILCodeGen.Compile` (after parse, before inliner), not as a separate engine stage.
- The 500× / 6 ns figures are repeated from `CLAUDE.md`, not independently measured; the bundled benchmark project does not contain that comparison.
- "Rezzing 1,000 copies compiles once" — mechanism is correct; checked evidence is `zengine run --copies` (≤500) and `ZResetLoopTest` (27 copies, 1 cache entry).
- Added: source layout, SHA-256 key details, thread names, heap compaction, runaway-protection limit, console command flags, `[YEngine]` config fallback, test inventory.
