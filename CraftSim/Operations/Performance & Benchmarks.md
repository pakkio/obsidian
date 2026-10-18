---
tags:
  - craftsim
  - performance
  - benchmarks
  - gc
  - physics
---

# Performance & Benchmarks

← Back to [[CraftSim]] | Related: [[ZEngine]], [[Build, Configuration & Self-Hosting]], [[Asset Deduplication & Database]], [[Decision Models Benchmark]] (osJev model accuracy, latency and cost)

CraftSim's runtime work targets script throughput, frame pacing, GC footprint, async I/O and native physics. Only numbers that exist in repo files are quoted below, each with its source.

> [!NOTE] Verified against code (commit `9fea6a0332`, 2026-10-16)
> I re-read the benchmark tables in `CRAFTSIM.md`, `README.md`, `tasks/stress-results.txt`, `doc/ubode/*` and the code paths for frame pacing, thread pool, GC config and native libs. I did **not** re-run any benchmark; all figures are historical measurements. The harness `bench_heavy.py` that produced the 100-NPC table is **not in the repository** (`find` returns nothing) so it cannot be re-run from the checkout.

---

## 1. The 100-NPC / 1,000-script stress test

Source: `CRAFTSIM.md` section 6 (README.md "Heavy Benchmark" repeats the net9 column). Workload: a heavy OAR with 100 NPCs and 1,001 sleep/touch scripts, measured with `show stats`. **Builds measured were .NET 8 (Server GC) and .NET 9 (Workstation GC); it has not been re-measured on .NET 10** (README says so explicitly).

| Metric | Vanilla OpenSim | CraftSim .NET 8, Server GC | CraftSim .NET 9, Workstation GC |
| :--- | ---: | ---: | ---: |
| Startup (s) | 1.6 | 2.0 | 1.5 |
| Load + spawn (s) | 66.0 | 66.0 | **44.0** |
| SimFPS / dilation | 55 / 1.0 | 55 / 1.0 | 55 / 1.0 |
| Root agents / active scripts | 100 / 1001 | 100 / 1001 | 100 / 1001 |
| Script events/s | 333 | 485 | 496 |
| RSS base (MB) | 159.7 | 176.5 | 160.9 |
| RSS peak (MB) | 208.6 | **1,671.7** | 465.9 |
| CPU % avg | 0.6 | 1.3 | 1.25 |

Derived deltas (computed from the table, matching the repo docs):
- .NET 9 WS-GC vs .NET 8 Server-GC: load+spawn **-33.3%**, peak RSS **-72.1%**, base RSS -8.8%, startup -25.0%, events/s only **+2.3%** (485 -> 496).
- .NET 9 vs **vanilla**: events/s **+49.0%** (333 -> 496), load+spawn -33.3%, **peak RSS +123.4%** (208.6 -> 465.9 MB), CPU roughly double. The "+49%" is vs vanilla, the "-72%" is vs the earlier Server-GC build: they are different comparisons and must not be read as one.
- net9 column = mean of two runs (peak RSS 464.8 and 466.9 MB). Vanilla did not collapse: it also held 55 FPS.
- Why Server GC lost: it creates a heap + GC thread per logical core (16 on the test host) for a workload dominated by one scene heartbeat and 4 script workers. `bin/OpenSim.runtimeconfig.json` has `System.GC.Server=false` (+ `HighMemoryPercent=50`). `bin/opensim.sh` no longer exports `DOTNET_gcServer` (it silently forced Server GC until 2026-07-15; measured effect on an idle region ~37 MB, 431 -> 394 MB RSS, per README).
- Newer, lighter comparison (README, whitepaper PDF): vanilla 0.9.3.0 vs CraftSim on .NET 10, single empty region: ~168 MB vs ~180 MB idle RSS. Full numbers are only in `zengine_whitepaper.pdf` (**not checked**). Caveat from README: `tasks/simtest.py`'s test-region UUID collides with the real `Nexus` region, so a naive run measures production content.

### Other measured runs in the repo
`tasks/stress-results.txt` (2026-06-20): **100 NPCs solo** held 55 SimFPS, dilation 1.00, RSS 160 -> 182-184 MB (~240 KB/NPC); lightweight appearance, no scripts/attachments. **179 combined** (100 NPCs + 79 pCampBot avatars; 25 connects, 68 timeouts): 55 FPS throughout, peak 501 MB RSS (~2.8 MB/agent), 1,272 `[XINVENTORY]` log lines, zero inventory errors. `tasks/engine_benchmark.md` describes the YEngine-vs-ZEngine procedure (`zengine run tasks/engine_compare.lsl --touch --wait 5000`, >=5 runs, medians) but contains no results. Rich-inventory / attachment-cloning NPC scaling is listed as unbenchmarked (CLAUDE.md "Known Follow-ups").

---

## 2. Core runtime & frame pacing (code-verified)

- **Frame timer** (`OpenSim/Region/Framework/Scenes/Scene.cs:1868-1905`): after each heartbeat, `Thread.Sleep` for the bulk (`remaining - 0.5 ms`, only when `remaining > 1.5 ms`), then a `Thread.SpinWait(50)` loop until `framestart + frameTimeMS` using `Util.GetTimeStampMS()`. This replaced coarse sleep granularity drift. (It is a Sleep + spin tail, not a pure Stopwatch timer as the old note implied.)
- **Thread pool** (`OpenSim/Region/Application/Application.cs:142-190`): bounds are `workerThreadsMin = 500`, `workerThreadsMax = 1000`, `iocpThreadsMin = 1000`, `iocpThreadsMax = 2000`; the code **clamps the runtime's *max* thread counts into those ranges** and calls `ThreadPool.SetMaxThreads(...)`. It does not raise the pool *minimums* (the min variables are the lower clamp for the max). Runtime change is possible with console `debug threadpool set worker|iocp min|max <n>` (`ServerBase.cs:434-485`).
- **Asset HTTP handlers**: cached `XmlSerializer` statics (`AssetServerGetHandler.cs:51-53`) and no extra outer `Task.Run`; the handler awaits `m_AssetService.GetDataAsync` (lines 76-101).
- **Async inventory**: `IInventoryService.cs` has 23 `Task` signatures (rough grep count; old doc said ~22) and `XInventoryServicesConnector` implements `GetMultipleFoldersContentAsync` (762) / `GetMultipleItemsAsync` (1056). CRAFTSIM.md section 2 explicitly says benefit is workload-dependent and **no "8x async" figure was measured**. Startup/shutdown ordering paths (`LoadPrimsFromStorage`, `SaveTerrain`) stay synchronous by design (per old note/CLAUDE.md; not re-derived).
- **ZEngine scheduling knobs** (`bin/OpenSim.ini:1040-1056`, read in `ZEngine.cs:270-278`): `[ZEngine] NumWorkers = 4`, `MaxEventSliceSeconds = 30`, `MaxWorkerQuantumMilliseconds = 5` (fairness quantum at ZIR block boundaries; 0 disables). The old note's `NumThreadScriptWorkers = 4` is a **YEngine** key (`XMREngine.cs:213`, default 2) and does not configure ZEngine. `llSleep` is a flag write checked between ZIR blocks (~6 ns/op vs ~3,100 ns for an exception throw, per CLAUDE.md measurement).
- **Other config**: `[NPC] MaxNumberNPCsPerScene = 150`; each region `MaxAgents = 250`; `[Statistics]`/`[Monitoring]` enabled in the tracked ini (`SlowAssetThreshold 100`, `SlowDatabaseThreshold 50`, `SlowClientThreshold 200`, `SlowSceneThreshold 100`).

---

## 3. Native physics libraries

### ubODE (default; `OpenSimDefaults.ini`: `physics = ubODE` line 302, `meshing = ubODEMeshmerizer` line 274)
- `bin/lib64/libubode-x86_64.so`: rebuilt with GCC 12 `-O3 -march=x86-64-v2`; README: "~13% faster physics stepping on a 2000-body benchmark", needs `GLIBC_2.14` + SSE4.2; stock copy kept as `libubode-x86_64.so.pre-optimized-backup` (both present in `bin/lib64`). The 13%/2000-body figure is quoted from README only; **no raw data for it was found** **(unverified)**. `x86-64-v3` was measured slower (README).
- **Staged, not installed: the "1.1.0" candidate** (`artifacts/ubode/libubode-x86_64.so` + `.rollback` + `SHA256SUMS`; `cmp` shows it differs from `bin/lib64`). `doc/ubode/README.md` states `bin/lib64` has not been replaced and the live simulator not restarted. Install = stop, copy over `bin/lib64/libubode-x86_64.so`, start via `opensim.sh` (do not overwrite a mapped library in place). Build: `scripts/build-ubode-candidate.sh <new-dir>` (autotools, flags `-O3 -march=x86-64-v2 -mtune=generic`, threading off).
  - Synthetic scene, pinned to 1 CPU, median of 5, 1,000 steps (`doc/ubode/README.md`): 150 bodies 1,378.2 -> 1,089.3 ms (**-21.0%**), 300 bodies 3,420.4 -> 2,913.2 ms (**-14.8%**), 600 bodies 7,835.1 -> 6,546.1 ms (**-16.5%**). "Native scene time, not whole-simulator performance."
  - Stability: 200/200 seeds ran, bodies sank in 10 seeds (5%) vs 9/200 seeds (4.5%) for production: no proven stability change; ASan 80/80 seeds clean (`stability-200-seeds.json`, `asan-80-seeds.json`, `timings.json`).
  - Failure containment: C++ exceptions caught in `dCollide`/`dSpaceCollide(2)`/`dWorldQuickStep`; a failed QuickStep **stops that region's physics** with an error log; non-finite prims/avatars are restored. Segfaults and native memory corruption can still kill the process.
  - NPC crowds (`doc/ubode/npc-startup.md`, assembly/ubODE version `1.1.0`, `VersionInfo.cs:32`): crowds of >=48 characters use a 2D bbox sweep. Per-step collision cost before/after: 150 scattered 0.200/0.054 ms (-73%), 600 scattered 1.014/0.190 (-81%), 1,200 scattered 2.095/0.371 (-82%); groups of six -51%/-71%/-75%; packed 150: 0.622 -> 0.509 ms (-18%). Repeated-mesh startup with a shared 3,200-triangle mesh: 300 actors 10,136 -> 40 ms; 1,200 actors 41,748 -> 35 ms (1 native build instead of N). These are harness numbers, "not whole-region speedups"; live-region timing is still to be measured.
  - Harness: `tests/ubode/performance/PerformanceChecks.csproj` (`... PerformanceChecks.dll bin artifacts/ubode/libubode-x86_64.so [check|trajectory]`), `tests/ubode/managed/RecoveryChecks.csproj`.

### BulletSim (optional)
- `bin/lib64/libBulletSim.so` (README: `std::map/set` -> `unordered_map/set` on hot paths, `-O2 -DNDEBUG`, `vsnprintf` overflow fix) needs `GLIBC_2.38` and the `libwrap_memcpy.so` preload set by `opensim.sh`; `libBulletSim.so.pre-optimized-backup` (GLIBC_2.4) is kept for old hosts. `preflight.sh` warns only when `physics = bulletsim`. Native build details are README-only **(unverified against source; binaries only)**.

---

## 4. Reading the numbers honestly
- Historical .NET 8/.NET 9 data; .NET 10 heavy-load re-measurement is outstanding.
- "No universal speedup": README states the gains are workload-dependent.
- Raw-speed claims for ZEngine vs YEngine are qualitative (cooperative `llSleep`, touch responsiveness); the "10,000 scripts" figure was never measured (CRAFTSIM.md).

---

## Corrections vs previous doc
- Delta column was ambiguous: "+48.9% vs Vanilla" is vs vanilla (repo says +49.0%); vs .NET 8 the event-rate gain is +2.3%. The doc omitted that **peak RSS is +123% vs vanilla** and that CPU roughly doubles.
- Benchmark runtime was presented next to ".NET 10"; the table is .NET 8/9 only, and `bench_heavy.py` is absent from the repo.
- Thread pool: old text said minimums were raised; code clamps *max* threads into 500-1000 worker / 1000-2000 IOCP.
- Frame pacing is Sleep + `SpinWait(50)` tail, not "Stopwatch target-based".
- `NumThreadScriptWorkers = 4` -> wrong key for ZEngine (`[ZEngine] NumWorkers = 4`).
- ubODE: the 13% claim has no raw data in-repo; the repo's documented measurements are the 150/300/600-body table (-14.8% to -21.0%), plus the **uninstalled 1.1.0 candidate** and its NPC-crowd/mesh results, none of which the old note mentioned.
- Added stress-results (100 solo, 179 combined), thread-pool console command, ZEngine scheduling keys.

---

## 5. Is ZEngine more stable with thousands of scripts, and is there more room for 100 avatars + 100 NPCs?

**Short answer: the repo does not demonstrate either claim.** What exists, and what does not:

| Claim | What was measured | Verdict |
| :--- | :--- | :--- |
| Stable with ~1,000 scripts | 1,001 scripts + 100 NPCs: **55 SimFPS, dilation 1.0 on CraftSim and on vanilla alike** (table in section 1). | Stable, but **vanilla was equally stable**: it "did not collapse" (`CRAFTSIM.md` section 6). No stability advantage shown. |
| Stable with thousands (10,000) | Never run. `CRAFTSIM.md`: the "10,000 scripts" figure "was never measured and is not claimed". | **Unmeasured.** |
| Faster | Script events/s 333 (vanilla) vs 496 (CraftSim net9), +49%. | Real, measured, historical (.NET 9). Throughput, not headroom. |
| Cheaper in memory | Peak RSS 208.6 MB (vanilla) vs 465.9 MB (CraftSim net9). | **Worse** than vanilla by 123%; CPU roughly doubles. |
| Engine A/B (YEngine vs ZEngine, same build) | `tasks/engine_benchmark.md` defines the procedure and holds **no results**. | **Not done.** The 333 vs 496 comparison is vanilla vs the whole fork, not engine vs engine. |
| 100 avatars + 100 NPCs | Never run. Nearest: 100 NPCs solo (55 FPS, 182 MB) and 100 NPCs + 79 bot avatars requested = 179 agents (55 FPS, peak 501 MB), but only **25 bot connects succeeded and 68 timed out**, so real avatar load was far below 79. No scripts and a lightweight appearance in that run. | **Not demonstrated.** And no vanilla run exists for either scenario to compare against. |

The configured ceilings (`[NPC] MaxNumberNPCsPerScene = 150`, region `MaxAgents = 250`) allow 100 + 100, but a limit is a setting, not a measurement. The "40 NPC limit" once reported was a config default, not a framework limit.

### Mechanisms that could plausibly help (design reasoning, not benchmarks)

- **No interpreter**: scripts are compiled to CIL, so per-event cost is native-code speed; this is the likely source of the events/s gain.
- **Sleeping scripts hold no thread**: `llSleep` is a flag write checked between ZIR blocks (~6 ns/op against ~3,100 ns for an exception), and a timer re-enqueues the script. A thousand sleeping scripts cost memory, not workers.
- **Bounded handler time**: events are split at block boundaries with a fairness quantum (`MaxWorkerQuantumMilliseconds = 5`), so one heavy handler cannot hold a worker or the frame loop indefinitely.
- **One compile per distinct source**: the compiled-script cache is keyed by SHA-256 of the source, so 1,000 copies of one script compile once.
- **Per-script cost is not free**: compiled delegates stay resident in RAM, which is part of the 466 MB peak.

### Risks that cut the other way (all in `CLAUDE.md`)

- A CIL compile failure means the script **does not run**; there is no interpreter fallback.
- The compile cache is **never evicted**; a region that rezzes many one-off scripts grows without bound over a long uptime.
- `ZEngineGlobalsTest`: 7 of 10 tests failed when first run (integer, string, vector and mixed globals read back as -1); an open bug in global-slot initialisation, not yet investigated.
- Exception handling differs between the typed fast path and `registry.Invoke` by design.

### What would settle it

Run `tasks/engine_benchmark.md` for YEngine and ZEngine on the same .NET 10 build, then the real target load: 100 real avatars (bots that actually connect; the earlier run timed out on login at a 1 s delay) plus 100 NPCs plus the script set, with `show stats` for SimFPS, dilation, events/s, RSS and CPU, repeated at least five times. Until then the supportable statement is: **ZEngine is faster per event and held a flat 55 FPS at 1,001 scripts + 100 NPCs, as did vanilla; more headroom for avatars and NPCs is not shown.**