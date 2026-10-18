---
tags:
  - craftsim
  - opensim
  - moc
  - index
aliases:
  - CraftSim Overview
  - pakkio/craftsim
---

# CraftSim

**Start here.** Hub of the CraftSim documentation: what the project is, where each topic lives, and the few facts worth having on one page. Read [[Prefazione]] first if you want the story; come back here to find things.

> [!WARNING] This hub was rebuilt on 2026-10-17
> The previous `CraftSim.md` was lost: the file was found empty (0 bytes) and the empty version had already been committed. This page is a **reconstruction** from the other notes and from the repo, not a restore. The "Quick reference" below was re-read from the repo that day; descriptions of the topic pages are taken from the pages themselves. If you have the old version (Obsidian *File recovery*, MEGA version history), compare it against this one.

CraftSim is a fork of OpenSimulator on .NET 10. Its centre is **ZEngine**, a script engine that compiles LSL to CIL, plus a set of `os*` extensions (databases, queues, caches, machine learning, language models, typed decisions), content-addressed asset deduplication, an MCP server for driving the simulator from outside, and the CraftStorm viewers that connect to it. Repo: `pakkio/craftsim`. The viewers: `pakkio/craftstorm`, documented in [[CraftStorm]].

## 1. The engine and the language

- **[[ZEngine]]**: the LSL-to-CIL compiler, scheduler, cooperative yielding, compiled-script cache.
- **[[ZEXT Language Extensions]]**: reference for the syntax ZEngine adds on top of LSL (ternary, `??`, `|>`, slices, maps, f-strings, `switch`, `foreach`, the `->` async bind, `await { }`, timers, the preprocessor) and for `zengine desugar`.
- **[[ZEXT Sugar and Async - Worked Examples]]**: the same features as runnable scripts with real output, the lowered LSL, and the traps found.

## 2. The `os*` extensions

- **[[OSSL Extensions (AI, ML, OsDb, OsQ)]]**: master index of the extension families added to `IOSSL_Api`.
- **[[ZEXT Constructs - One Example Each]]**: 29 realistic, run examples, one per ZEXT construct, with what each exposed.
- **[[Added Functions Summary]]**: one line per added function, console command, MCP tool and module.
- Deep dives, one per family:
  - **[[Module - OsDb & Vector Search]]**: notecard-backed JSONL row store, indexes, k-nearest-neighbour search for RAG.
  - **[[Module - osAI & osJev]]**: OpenRouter chat and image models, typed decisions with Jev, SubQ and other decision models.
  - **[[Module - osML & Embeddings]]**: on-sim machine learning and text embeddings.
  - **[[Module - OsQ, OsCache & JsonStore]]**: message queues, TTL cache, JSON tree stores.
  - **[[Module - osOcr, osPlot, osHTTP, osLex & osYacc]]**: OCR, charts, HTTP client, lexer and parser.
  - **[[Module - Pathfinding, Experience, Materials & Avatar Utilities]]**: navmesh, experiences, materials, avatar helpers.
  - **[[Avatar, NPC & Region Modules]]**: NPCs, avatars, regions.
  - **[[Module - Server-Side Baking (SSB)]]**: the `UpdateAvatarAppearance` capability and server-side compositor, with the SL-viewer alignment table.
  - **[[Module - AISv3 Inventory API]]**: the `InventoryAPIv3` / `LibraryAPIv3` REST inventory capabilities, with the SL-viewer alignment table.
  - **[[Module - Navmesh & Pathfinding]]**: DotRecast navmesh, `llCreateCharacter` family, rebuild and incremental build, off-by-default trap.
- **[[OSSL Extension Examples - Complete Coverage]]**: runnable examples (osDb, osJev) with real output.
- **[[End-to-End Multi-Module LSL Architectures]]**: scripts that combine several modules.

## 3. Operations

- **[[Build, Configuration & Self-Hosting]]**: building, configuration, running a grid.
- **[[MCP Server & Live Diagnostics]]**: the MCP server, its 76 tools, authentication, the stdio bridge and SSH tunnel.
- **[[Asset Deduplication & Database]]**: content-addressed assets, object templates, the compiled-script cache, fallback servers.
- **[[SQLite in CraftSim]]**: SQLite as the default store, the System.Data.SQLite port, per-service files, WAL, traps.
- **[[Decision Models Benchmark]]**: eight decision models compared on 30 labelled tickets (accuracy, speed, cost).
- **[[Performance & Benchmarks]]**: what was measured about throughput, memory, physics.
- **[[Voice (WebRTC, Janus & SynthVoice)]]**: voice through WebRTC.

## 3b. Upstream comparison

- **[[Upstream OpenSim vs CraftSim - Divergence]]**: what upstream OpenSim does not do (SSB, AIS, display-name changes, navmesh) and what CraftSim added.

## 4. What is wrong or unproven

- **[[Known and Unresolved Problems]]**: open problems in craftsim and craftstorm, items that only looked open, and claims that were never measured.

## 5. Quick reference

Re-read from the repo on 2026-10-17 (`craftsim` at `512c3f319b`); `origin/master` was checked again on 2026-10-18 and is one commit further (`181fb0ae86`, ZEngine tokenizer/constants fix, see [[ZEXT Language Extensions]]), with no other change to the documented surface.

| Item | Value |
| :--- | :--- |
| Version | `1.1.0` (`OpenSim/Framework/VersionInfo.cs:32-33`, flavour `Dev`) |
| Runtime | .NET 10 (`<TargetFramework>net10.0</TargetFramework>`); `global.json` pins SDK `10.0.100` with `rollForward: latestFeature` |
| Default script engine | `DefaultScriptEngine = "ZEngine"` (`bin/OpenSim.ini`) |
| ZEXT | `[ZEngine] ZEXT` defaults to `true`; `#include` needs `[ZEngine] EnableInclude = true` |
| Regions in the tracked `Regions.ini` | `Nexus` (1024x1024, UDP 9000) and `pakkio2` (256x256, UDP 9001) |
| Build and test | `./runprebuild.sh` on a fresh clone, then `./compile.sh`, `./test.sh` or `./check.sh`; test projects in `tests/OpenSim.Tests` and `tests/CraftSim.SmokeTests` |
| Recorded test result | 477/479 + 2/2 on 2026-10-01 (one load-sensitive test), **not re-run** in the documentation passes; the `DecisionClient*` tests (36) passed on 2026-10-17 |
| Repo docs | `README.md` (changelog and the Extensions Reference), `CLAUDE.md` (orientation), `CRAFTSIM.md`, `BUILDING.md`, `SELF_HOSTING.md`, `JANUS-SETUP.md`, `TEST-PLAN-VOICE.md`, `zengine_whitepaper.tex` |

Top-level layout of the repo:

```text
OpenSim/          simulator source (ZEngine under Region/ScriptEngine/ZEngine)
addon-modules/    UsersLog, OpenSimSearch
bin/              runtime: ini files, native libraries, scripts (opensim.sh, preflight.sh, backup-db.sh)
tests/            OpenSim.Tests, CraftSim.SmokeTests
tasks/            demonstration and test LSL scripts (tasks/jev, tasks/zext)
scripts/          mcp-craftsim-bridge.js, build helpers, sample LSL
doc/              integration notes, DECISION-MODELS-BENCHMARK.md, whitepaper generators
```

## 6. How these notes were checked

Each topic page says against which commit it was read and marks what could not be proved from the code (*unverified*). Where `CLAUDE.md`, `README.md` or the whitepaper disagree with the code, the page follows the code and notes the difference. The documentation passes mostly read code; they ran very little, which is why [[Known and Unresolved Problems]] lists what was never exercised.
