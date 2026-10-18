---
tags:
  - craftsim
  - craftstorm
  - known-issues
  - unresolved
  - backlog
aliases:
  - Known problems
  - Open issues
---

# Known and Unresolved Problems

← Back to [[CraftSim]] | Related: [[Added Functions Summary]], [[Decision Models Benchmark]], [[Asset Deduplication & Database]], [[Voice (WebRTC, Janus & SynthVoice)]], [[Performance & Benchmarks]]

What is still open, what only *looks* open, and what was claimed but never measured. Written **2026-10-17** from the repo docs (`README.md`, `CLAUDE.md`, `doc/INTEGRAZIONE-CRAFTSIM2026.md`, `web/HANDOVER_TODO.md`), the commit history of `craftsim` (`47fb914f1c`) and `craftstorm` (`bba1a8b8f6`), and the checks listed under each item. Nothing here was fixed by writing it down.

Legend: **Open** = still a problem. **Workaround** = open, with a switch or a known way round. **Unverified** = nobody has proven it either way. **Closed** = looked open, is not.

## 1. craftsim (the simulator)

### 1.1 Open or with a workaround

| Area | Problem | Status | Source |
| :--- | :--- | :--- | :--- |
| Decision models | `DecisionClient` does not retry. `perplexity/pplx-decider-v1.2` answers `HTTP 429 ... retry after 1s`: 12 of 30 calls in the 2026-10-17 benchmark, even with 1.2 s between calls. A script calling the default model gets an `ERROR:` string now and then | **Open** (scripts must retry; a back-off loop is in `tasks/jev_decision_eval.lsl`) | [[Decision Models Benchmark]] |
| Decision models | `subq-snap-preview` is text only; images and `osJevBox` return `ERROR:` because SubQ's image contract was not implemented | **Open** | `CLAUDE.md` |
| Decision models | SubQ's API reports no `cost`; any cost figure is list price × tokens | **Open** (cosmetic) | benchmark |
| Decision models | `scripts/jev_tool_picker.py` is a separate TypeSafe client and was not moved to `DecisionClient` | **Open** | `CLAUDE.md` |
| Decision models | `osJevScore` and `osJevNoul` return `-1.0` on any error and cannot say why (429, no key, bad model look identical); with `pplx-decider-v1.2` as the default, a first call in a burst can come back `-1.0` | **Workaround**: retry (done in `tasks/jev/jev_score.lsl`) | `tasks/jev/` runs |
| Decision models | `osJevBox` is approximate: for a red square at (0.25-0.50, 0.125-0.375) Jev 1.14 returned (0.27-0.48, 0.10-0.53) with confidence 0.53 | **Open** (model quality, one image) | `tasks/jev/jev_box.lsl` |
| Decision models | Default `pplx-decider-v1.2` is committed (`47fb914f1c`) but the running sim still has the old default until it is rebuilt and restarted; the new unit-test cases were not run | **Open** | session 2026-10-17 |
| Voice | A text IM can ring like a call: under WebRTC voice the viewer sends `start p2p voice` for every P2P conversation, and the request cannot tell a call from an IM | **Workaround**: `[WebRtcVoice] P2PRing = sessionstart` (default), `provision` or `none`. **Not tested with a viewer** | `CLAUDE.md` |
| Voice | The addon `os-webrtc-janus` compiles and produces its DLLs, but voice was never exercised end to end (needs a viewer and the synthvoice server) | **Unverified** | integration doc §11.3 |
| Voice | The internal `WebRTCVoiceModule` and the addon register the same capabilities and `VoiceServerType`; if both are on, the last one to register wins. The internal module stays in the tree with `enabled = false`; removing it is left to its author | **Workaround** (keep one off) | integration doc §3.5, §11.4 |
| PostgreSQL | The `ON CONFLICT (id) DO UPDATE` upsert and the `IsAlreadyStored` `SELECT` compile, but that SQL was not executed against a live PostgreSQL in the integration pass, and no test covers it (SQLite is covered, MySQL is `[Explicit]`) | **Unverified** | integration doc §11.3 |
| Asset fallback | The real HTTP fallback was never exercised; `EnableAssetFallback = false` by default. Backlog in `tasks/todo.md`: obsolete `HttpWebRequest`, an unused field, Info-level log line on every call, no circuit breaker for servers that just failed | **Open** | integration doc §11.4 |
| Warp3D | Nothing tests `Warp3DImageModule`; it is only known to compile | **Unverified** | integration doc §11.3 |
| Tests | `ZResetLoopTest.Test27CopiesWithoutBrakeSaturateWorkers` fails under the load of the full suite (measures ~1.75 cores against a threshold of > 2) and passes alone. Suggested fix: compare braked vs unbraked, not an absolute core count. Recorded run: 477/479 + 2/2 on 2026-10-01, not re-run since | **Open** | integration doc §11.4, `CLAUDE.md` |
| Pathfinding | Off by default in code; the `OpenSimDefaults.ini` comment saying "default: true" is wrong. `PU_FAILURE_UNREACHABLE` and `PU_FAILURE_NO_NAVMESH` are never raised, so scripts cannot react to them | **Open** | [[OSSL Extensions (AI, ML, OsDb, OsQ)]] |
| ZEXT desugar | `zengine desugarcheck` reports **FAIL** on a script using `boolean`: the original prints `TRUE` (f-string), the lowered one prints `1`, since `boolean` becomes `integer` | **Open** (lowering is not output-identical for booleans) | `tasks/zext/sugar_lowerable.lsl` |
| ZEXT desugar | Lowering named timers leaves a call to `__zSetNamedTimer(...)` that the output does not define, so the result is not self-contained stock LSL; string `%` and `*` cannot be lowered at all (refused with an error) | **Open** | `tasks/zext/timers.lsl`, `sugar_values.lsl` |
| ZEXT | A variable named `key` (or another type name) in a destructuring assignment gives ten unrelated-looking errors starting "list unpacking currently requires explicit type declarations" | **Open** (the message points at the wrong cause) | `tasks/zext/sugar_control.lsl` |
| ZEXT | **Negative list index returns an empty value**: `scores[-1]` gives `""` / 0 in ZEngine 1.1.0, while `zengine desugar` lowers it to `llList2Integer(scores, -1)`, which returns the last element. String indexes (`s[-1]`) are fine | **Open** (bug) | `tasks/zext/probe3.lsl`, construct 7 |
| ZEXT | `decimal` cannot be mixed with a float literal (`subtotal * 0.22` is a compile error) and `c == 0.30` is false for a decimal `0.10 + 0.20`; the message says to cast, but a literal in a `decimal` variable is the practical way | **Open** (limitation) | construct 2 |
| ZEXT | State parameters are in scope in `state_entry` only; other events see "undefined variable". A transition from a state to itself with new arguments did not re-enter it | **Open** (limitation, undocumented before) | construct 16 |
| ZEXT | String `%` understands `%d %i %s %f %%` only; `%5d`, `%.2f`, `%x` are left unchanged without any error | **Open** (limitation) | construct 15 |
| ZEXT | Escapes are still processed inside `"""..."""` (`C:\temp` becomes `C:` + tab + `emp`); only `r"..."` keeps backslashes | **Documented behaviour** | construct 12 |
| ZEXT | `#include` and `#STEP` have no runnable example: neither works in the throwaway prim of `zengine run` (no inventory, interactive dialog) | **Unverified** | `ZEXT Constructs - One Example Each` |
| Dedup | Object templates do not dedupe when `KeyframeMotion` is active at different positions (positions are inside the hashed motion JSON) | **Open** (documented trade-off) | [[Asset Deduplication & Database]] §2 |
| Dedup | SQLite writes have no explicit transaction and no identical-write skip; MySQL `replace INTO` resets `create_time` / `access_time` on every non-skipped rewrite; a missing blob reads back as an empty array, not an error; Hypergrid-proxied foreign assets are not counted; `FSAsset` / `XAsset` stores are not deduplicated | **Open** (by design, but easy to trip over) | same, §1, §5 |
| Dedup | No console command or MCP tool reports the ZEngine compile cache (size, hits, refcounts) | **Open** | same, §6 |
| Inventory | NPC scaling with rich inventories and attachment cloning is unbenchmarked (needs a viewer-captured appearance). The optional tier-2/3 async migrations (scene inventory ops, `LocalInventoryServiceConnector`, `HGInventoryBroker`, `InventoryTransferModule`) are available but not done | **Open** (optional) | `README.md` "Known Follow-ups" |
| Repo | `CLAUDE.md` and `README.md` differ between the `craftsim` and `CraftSim2026` trees and were never reconciled; Robust / PostgreSQL changes that were expected from production do not exist on this machine | **Open** | integration doc §0.4, §11.4 |

### 1.2 Unverified claims in the docs

- **"~275× faster" `osDbInsertMany`** (500 rows in 0.027 s vs 7.42 s): plausible from the one-persist design, never re-measured.
- **"383 `IOSSL_Api` methods / 17 families"** and **"over 100 new"**: stale; the interface now has ~471 declarations (406 distinct names), and "new vs upstream" cannot be checked without diffing against upstream.
- **"0 errors, 0 warnings", 477/479 tests**: recorded results from 2026-10-01, not re-run in the Obsidian passes.
- **Positional audio via wolfvoice** and the `SLData` rationale: not provable from this repo's C#; the voice server is another repo.
- **`CLAUDE.md` says version 1.0.4**; the code is `1.1.0` (`VersionInfo.cs`).
- **Production host layout** (`<grid-host>:<grid-port>`, regions `PKC` + `pakkio2`): from `CLAUDE.md` only.

### 1.3 Closed (looked open)

| Item | Why it is closed |
| :--- | :--- |
| This vault's `OSSL Extension Examples - Complete Coverage` note was cut off at 4096 bytes mid-statement, had no `osJev` examples, and its osDb scripts called `osDbCreate` with one argument (it takes two) | Rewritten 2026-10-17; the osDb and `osJev` scripts in it were run on the live sim (`tasks/jev/`). Other families in it are still "not re-run" |
| `global.json` pin made `dotnet build OpenSim.sln` exit 155 on this machine (integration doc §10.7, marked **[APERTO]**) | Commit `2026-10-01 "build: accept any 10.0.x SDK so the tree builds on 10.0.400"` |
| Default decision model `perplexity/pplx-decider-v1.1-27b` returned `404 No endpoints found` | Default moved to `pplx-decider-v1.2` (`47fb914f1c`, 2026-10-17) |
| Object-asset dedup lost whole linksets on load (shared placeholder UUID) | Fixed 2026-08-26 (`eb5dcc4d34`) |
| `PGSQLAssetData.Delete` was a no-op, so `assets sweep` never reclaimed space on Postgres (P1); legacy rows never migrated (P6); KeyframeMotion state lost on every store/load (P7) | Fixed 2026-07-05 |
| Compile cache never evicted (P2) | Refcounted eviction |
| `JsonStore` silently dropped every JSONL record after the first | Fixed 2026-09-29 (loads JSONL as an array) |
| AISv3: an inventory item PATCH carrying `hash_id` did not link the uploaded asset | Fixed 2026-10-12 (`16818e14af`) |
| Pooled guest login created no inventory skeleton | Fixed 2026-09-02 (`9e3de293c7`) |

## 2. craftstorm (Firestorm viewer + `web/` viewer)

### 2.1 `web/HANDOVER_TODO.md`: mostly done, the file is just stale

An earlier summary said "12 known defects still unresolved". **That was wrong**: it was written from the document alone. The document dates from 2026-09-08 and was never updated, but the same day commit `97d7b00815` ("v200.0: handover robustness - transport, material, queue, BVH, budgets") landed most of it, with tests. Checked on 2026-10-17:

- `node --test tests/handover_fixes.test.mjs`: **13/13 pass**.
- The Python half, `tests/test_handover_fixes.py`, could **not be run** on the server: `bridge.py` imports Pillow and `PIL` is not installed there. That is the environment, not a failing test; it still needs a run where Pillow exists.

| # | Item | State |
| :--- | :--- | :--- |
| 1 | Stale WebSocket callbacks | **Done**: `this.socket !== socket` guards (`runtime/bridge_client.js:27,33,39`); test |
| 2 | Voice rejoin | **Done**; test |
| 3 | Voice capability calls blocking movement | **Done**; Python test (not run here) |
| 4 | Bridge flush self-rescheduling | **Done**; Python test (not run here) |
| 5 | Killed ids left in the visibility set | **Done**; Python test (not run here) |
| 6 | Mutation of cached shared materials | **Probably done**: no direct `material.map =` assignment left; **no test** |
| 7 | Texture request state | **Done**: `reqState` `queued / inFlight / staged / deferred / failed` (`app.js:1517-2162`) |
| 8 | Texture upload staging | **Done**: `pendingTextureUploadByTexture` (`app.js:1827`) |
| 9 | Cache budgets and ownership | **Done**: byte-aware budgets, `?texBudgetMiB=256&geoBudgetMiB=128` (`app.js:1349-1375`), counters in the HUD |
| 10 | Concurrent sculpt reads | **Done**: `pendingSculptGeometryByKey` (`app.js:4020`) |
| 11 | Dense instance groups | **Done**: `liveCount`, packed instances (`app.js:5572-5767`) |
| 12 | BVH distance at small scale | **Done**; test (scale 0.05 sphere) |
| 13 | Worker failure leaves promises pending | **Partly**: `onerror` handlers exist (`app.js:4447`, `:4578`) and queue depth / oldest-job age are in the diagnostics (`:10255`); **no test** that every pending job settles |
| 14 | BVH built off the render path | **Done**: worker-built BVH with synchronous fallback; 5 tests |
| 15 | Node vs Python bridge parity | **Open**: `bridge.js` now has texture edge/format parity code (`:1830-2038`, "Parity with bridge.py parse_texture_params"), but I found **no committed route-parity test and no recorded decision** on which runtime is supported (`npm start` still runs `bridge.js`, whose header calls it a "Node.js port of bridge.py") |

Follow-ups at the end of the file: HTTPS capability URLs and preserved query strings are covered by tests; per-session asset routing is implemented and tested, but `LAST_LOGGED_IN_SESSION` still exists as a backwards-compatible alias (`bridge.py:337`); a new login now clears the old scene (`app.js:7261`); GPU read-backs are skipped "while hot" but the diagnostics tick still takes pixel probes and thumbnails otherwise (`app.js:11880-11940`) - whether that runs by default in normal play is **unverified**.

### 2.2 Other open points

| Area | Problem | Status |
| :--- | :--- | :--- |
| Inventory | The web viewer does not use AIS (no `InventoryAPIv3` / `AISv3` in `bridge.py` or `app.js`); the inventory window (Ctrl+I) fetches through `FetchInventoryDescendents2` only. The server side (AISv3 since 2026-08-31) is ready | **Open** |
| Animesh | Suspected missing in craftstorm. The code has it end to end (flag parsing `bridge.py:3778`, `ObjectAnimation` capability `:1200`, rig mount and playback `app.js:7537-8005`, 65 animesh commits in the Firestorm tree). Not reproduced; test coverage is the flag parser and the animation maths (`anim_frame / rest / root`), nothing for worn animesh of other avatars or sim restarts | **Unverified**: needs the failing region / object |
| Tests | Python suite cannot run on the `craft` host (Pillow missing) | **Open** (environment) |

## 3. Notes about the notes

- `Module - osAI & osJev.md` in this vault was an **empty file** until 2026-10-17 although other notes linked to its "section 6"; it was rewritten from the repo docs and live tests.
- Both repos contain files that are not mine and not committed (`rebuild-pk.sh`, `rebuild-pk2.sh`, `scripts/brain.lsl`, `bin/OpenSim.ini` edits in `craftsim`); they were left alone.
- On 2026-10-17 two notes were emptied by scripted edits on the network drive (the write failed silently after the file was truncated): `CraftSim.md` (the hub) and `Operations/Voice (WebRTC, Janus & SynthVoice).md`. The empty versions were committed and pushed before it was noticed. Both were **rebuilt** from the repo, not restored, and say so at the top. Edits to the vault now use a write that is checked for size afterwards.
