---
tags:
  - craftsim
  - zengine
  - zext
  - desugar
  - verification
  - testing
aliases:
  - desugar verification
  - is desugar correct
---

# ZEXT Desugar Verification Report (2026-10-22)

← Back to [[ZEXT Syntax Sugar Index]] | The thing verified: [[ZEXT Desugar - Stock LSL Output]] | Known issues board: [[Known and Unresolved Problems]]

> [!SUMMARY] Verdict
> **On every script that can say something in the test harness, the desugared form behaves like the original.** In the equivalence suite 24 of 25 scripts produce output and all 24 are identical (the 25th needs the internet); across the whole 117-script corpus, 102 are equivalent, 10 say nothing without a viewer/inventory/network (so prove nothing), and 5 differ because they **throw inside the unit-test prim** (an exception-path difference, described below). The check found and fixed **seven real bugs** in the desugarer or the native engine (§3); one more exists as a documented divergence. What the evidence does *not* cover is listed in §4. Read that section before relying on the output for something that matters.

---

## 1. What "correct" means and how it was measured

Three kinds of evidence, each answering a different question.

| Question | Method | Strength |
| :--- | :--- | :--- |
| Does the desugared script **behave** like the original? | Run the original with ZEXT on, desugar it, recompile the output with **ZEXT off**, run that, compare everything the prim says (`say`/`shout`/`owner say`). `state_entry` and then `touch_start` are delivered to both. | strong for what is exercised, blind to what is not said |
| Does each sugar construct **parse to the right thing**? | Token-stream and AST comparison tests (`ZSugarTest`): the sugared form must tokenize to exactly the plain-LSL token sequence, or produce the expected nodes. | exact, but only for the shape |
| Is the output **well-formed stock LSL**? | The output is compiled by ZEngine with the ZEXT gate **off**, which rejects every extension construct. | catches leftover sugar, not LSL validity on other compilers |

The equivalence test is `ZDesugarTest.TestDesugarSuiteIsEquivalent` (every script in `tasks/desugar_suite/` plus the showcase, exact comparison). The wide net is `ZDesugarTest.CorpusEquivalenceSweep` (explicit, slow: every script in `tasks/` and `scripts/` that has a default state, 117 files, run twice, with UUIDs, generated ids and timing figures masked like `zengine desugarcheck` does). The scripts are heavily commented and documented in this family of notes; each note says which file it uses.

> [!INFO] The harness
> One throwaway prim in a unit-test region, no viewer, no avatars, no notecards in the prim's inventory, no network unless a script reaches out. A script waits until it has been silent for a "quiet" window (1.5 s in the suite, 1 s in the sweep, widened per script with a `// desugar-quiet: N` / `// desugar-timeout: N` header comment). Async sources used: `osMLpredictAsync("no-such-model", …)` (answers through `dataserver` with an error text) and `llHTTPRequest` to a closed local port (times out).

## 2. Results

### 2.1 The equivalence suite: 25 scripts, 24 identical, 1 silent

| Script | What it exercises | Output lines |
| :--- | :--- | :--- |
| `sugar_01_indent_braceless` | every control form braceless, continuation, `jump`/label, state change | 13 |
| `sugar_02_async_arrows` | `->` single/destructured/timeout/loop/function, `await` | 7 |
| `sugar_03_fat_arrow` | `=>` map, listen/sensor/timer closures | 5 |
| `sugar_04_json_strings` | `j'…'` `j"…"` `jr'…'` `j"""…"""` `j<<` | 6 |
| `sugar_05_heredocs` | delimiters, raw body, shift ambiguity | 5 |
| `sugar_06_small_sugar` | `def` `on` `let` `defer` `assert` `print` `in`, defaults/named, comprehension, chain, literals | 12 |
| `sugar_07_string_forms` | f/r/triple/single, `%`, `*`, slices, **negative list index** | 11 |
| `sugar_08_operators` | `?:` `??` `\|>` bitwise assignment, `boolean` | 9 |
| `sugar_09_state_arguments` | `state name(args)`, including as an unbraced `if` body | 5 |
| `sugar_10_inline_timers` | `timer -> N:` | 3 |
| `sugar_showcase` | nearly everything at once, real HTTP timeout | 19 |
| 14 pre-existing suite scripts | loops, switch, slicing, f-strings, named timer (pulses at 2/4/6 s), async + timeout, … | 1–8 each |

(`level2_asyncbind_http` needs the open internet and says nothing in the harness: it is the one silent script of the 25, among the 14 pre-existing ones.)

### 2.2 The corpus sweep: 117 scripts

```text
files=117  ok=100  silent=10  differ=7      (as run)
after explaining two of the seven:  equivalent 102 · silent 10 · differ 5
```

- **`ok`**: output identical after masking. Includes the 24 suite scripts, `craftsim_monad_pipeline_demo` (the `->`/`??` pipeline over a `foreach`, which exposed a real bug, §3), `pathfinding_demo`, the `jev_*`, `osq_*`, `osdb_*`, `json_*` families, and many more.
- **`silent`** (10): `car_script`, `cross_stress`, `inv_stress`, `jev_score`, `level2_asyncbind_http`, `level2_asyncbind_timeout`, `linkset_receiver`, `npc_stress_auto`, `sleep_touch`, `stress_object`. They need avatars, inventory, a vehicle, the network or a longer wait than the sweep allows, so nothing was said in either form. **They prove nothing either way.**
- **`differ`, explained and not desugar bugs**:
  - `test_vehicle`: the sweep's 3-second window cut the *native* run at 27 lines (it sleeps between phases); re-run with a long window both forms print the same 89 lines.
  - `osdb_demo`: the only difference is a generated run id inside notecard names (`osdb_demo_run_<10 digits>`), which the mask now covers.
- **`differ`, a genuine behaviour difference, five scripts** (`test_http_client`, `test_notecard_missing`, `test_notecard_reading`, `test_notecard_http_bridge`, `test_notecard_http_edge_cases`): they call an API on something that does not exist in the test prim (a notecard, a request target), and the **API throws a null-reference exception**. Natively the exception ends the script's event; in the lowered form the call sits inside a generated function and the failure surfaces at a different moment (the lowered run goes on to print the line of the next event and the error). Three repeated runs of `test_notecard_reading` gave the same pair of outputs each time, so it is deterministic, not a race. A script that does not throw is unaffected; a script that throws is not comparable. Documented in [[ZEXT Desugar - Stock LSL Output]] §4.

The first sweep (before the fixes below) had 91 ok / 16 differ. Of those 16, nine were only timing numbers, prim UUIDs or generated ids, one was the `->`-into-existing-variable bug (`craftsim_monad_pipeline_demo`), one a window cut-off (`pathfinding_demo`), and five are the exception-path group above.

### 2.3 Unit tests

| Class | Tests | What |
| :--- | :--- | :--- |
| `ZSugarTest` | 11 | token/AST shape of every construct; ZEXT gate; error messages |
| `ZSugarRunTest` | 7 | scripts run end to end (indent program, `in`, `let`, defer at every exit, defaults/named, comprehension, heredoc/JSON/numbers/print/assert) |
| `ZDesugarTest` | 36 (3 more are explicit tools: the sweep, the regenerator, the dumper) | original vs desugared: sugar scripts, `?:` in loop headers and short circuits, nested `->`, `await`, switch + async, string ranges, closures, helpers; the suite; the showcase file kept in sync |
| `ZEnginePythonStringsTest` | 17 | the string forms, `j` literals, heredocs |
| full `OpenSim.Tests` project | 650 | 639 pass; 3 are the explicit tools (not run by default); **8 fail, all environmental**: seven chart / image tests need `libgdiplus`, which this Windows box lacks (they pass on the Linux dev box), and `Test27CopiesWithoutBrakeSaturateWorkers` is load-sensitive (documented in `CLAUDE.md`). None involves the parser or the desugarer. |

---

## 3. Bugs the verification found (all fixed, each now pinned by a test)

| # | Where | Symptom | Cause | Fix |
| :--- | :--- | :--- | :--- | :--- |
| 1 | desugar | **Named timers never fired** after desugaring | the parser's lowering left a call to the compiler-internal `__zSetNamedTimer(...)` in the output; nothing set the due time the `timer()` handler tested. ZEngine itself still understood the call, so the old "does it compile" test passed. | real statements emitted (interval, due time, finest tick, `llSetTimerEvent`); due times stay on a grid |
| 2 | desugar | `boolean` printed `1` where the original printed `TRUE` | `boolean` was mapped to `integer`, but a boolean converted to a string prints `TRUE`/`FALSE` natively | the conversion is spelled out wherever a boolean meets a string, including call arguments (parameter types come from the registry) |
| 3 | desugar | `if (c) state s(args);` could not be desugared | state arguments make a compound statement that was not printable as an unbraced body | compound statements are printed in place |
| 4 | desugar | `-> summary` where `summary` already existed printed an empty value in nested async code | the flattened lowering declared a new shadowing variable; natively a bind stores into the existing handler variable | an existing variable is reused (payload converted if it is not a string) |
| 5 | **native engine** | `list[-1]` returned empty | `ZCILHelpers.ListGet/ListGetObj` rejected negative indices; `llList2<T>` and slices count from the end | negative index counts from the end |
| 6 | **native engine** | `"%d" % ["12"]`, `%f`, `%s` printed nothing for string list items | the formatter matched `string` but list items are the `LSL_String` wrapper | wrapper handled |
| 7 | parser | a closure outside any state crashed with a null reference (reported as "Parse error: Object reference…") | the closure expansion assumes a current state | clear error: *a closure can only be written inside an event handler of a state* |

Found and fixed earlier in the same pass while building the features (all with tests): `?:` in `while`/`do`/`for` headers was refused (loops are now rebuilt); a `?:` on the right of `&&`/`||` ran unconditionally ahead of the short circuit; a `?:` in a `for` initialiser was emitted after the declaration that needed it; `->`/`await` inside `if`, loops, `switch` and functions; string `*`, `%` and `case a..b:`; a `?:` in a global initialiser.

One divergence is **documented, not fixed**: ZEngine's own string→integer helper is strict (`(integer)"12abc"` is 0) while real LSL gives 12, so `"%d" % ["12abc"]` differs between the two forms ([[ZEXT Sugar - Strings, Operators and States (annotated)]]). Changing the helper would alter casts everywhere.

---

## 4. What is not covered (read this)

1. **No independent LSL compiler was run on the output.** "Stock LSL" means: compiles in ZEngine with the ZEXT gate off. It has not been fed to the Second Life compiler, YEngine or XEngine. Generated code uses constructs that are ordinary LSL (nested `if`, `jump` to labels, global state, functions), but a compiler with different limits (nesting depth, label scoping, script size) could object. The only non-core function used is `llOrd`.
2. **Silent scripts prove nothing** (10 of 117), and the harness can only observe chat output. A script whose effect is on prim properties, particles, physics or other objects is "equivalent" only in what it says.
3. **Five scripts differ on the exception path** (§2.2). No claim is made that exceptions behave the same.
4. **Timing is coarse.** Native named timers are scheduled exactly; the lowered form polls at the finest interval. The comparisons use windows of seconds; they would not notice a drift of a few hundred milliseconds. A waiting handler drops a second trigger in the lowered form; native queues it.
5. **No fuzzing.** Coverage is the constructs in typical contexts (each construct appears in at least one of: a handler, a loop, an `if`, a function, a nested `switch`, an async handler). Pathological nestings were not generated.
6. **Not exercised by the equivalence runs:** `#include`, `#TRON`/`#STEP`, `decimal`, persistence of script state across a region restart for the flattened handler (its variables are ordinary globals, but a restart in the middle of a suspension was not tested), multi-region or multi-prim interaction, and the `[ZEngine]` configuration variants.
7. **Characters outside the Basic Multilingual Plane** in a string `case a..b:` compare by code point in the lowered form and by UTF-16 unit natively.
8. **Event data after a suspension.** `llDetected*` and similar are only meaningful before the first suspension in the lowered form (and natively too, for events that deliver their data with the event).

## 5. Reproducing

```text
# the suite (about 3 minutes)
dotnet test tests/OpenSim.Tests/OpenSim.Tests.csproj --filter "TestDesugarSuiteIsEquivalent" --logger "console;verbosity=detailed"

# the wide net (about 6 minutes; explicit)
dotnet test tests/OpenSim.Tests/OpenSim.Tests.csproj --filter "CorpusEquivalenceSweep" --logger "console;verbosity=detailed"

# desugared text + native output + lowered output for any files
$env:DUMP_DIR='C:\tmp\out'; $env:DUMP_FILES='C:\path\a.lsl;C:\path\b.lsl'   # DUMP_QUIET / DUMP_TIMEOUT widen the window
dotnet test tests/OpenSim.Tests/OpenSim.Tests.csproj --filter "DumpDesugarAndRun"

# regenerate the committed showcase output
dotnet test tests/OpenSim.Tests/OpenSim.Tests.csproj --filter "RegenerateSugarShowcase"
```

On Windows: build `OpenSim.sln` first (the scene-based tests load the plugin assemblies by name), and run prebuild from PowerShell. Inside the simulator, `zengine desugarcheck --all` does the same comparison on a live region.

The working list behind all of this is `tasks/sugar-todo.md` in the repo.
