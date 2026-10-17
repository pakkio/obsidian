---
tags:
  - craftstorm
  - performance
  - benchmarks
  - diagnostics
  - webgl
  - gpu
aliases:
  - CraftStorm Performance
  - Performance Probe
  - CraftStorm Diagnostics
---

# CraftStorm Web - Performance, Budgets & Diagnostics

← Back to [[CraftStorm]] | Related: [[CraftStorm Web - Architecture, Rendering & Batching]], [[CraftStorm Web - Avatars, Animesh, Movement & Audio]], [[Performance & Benchmarks]]

CraftStorm Web follows a "50 ms rule" (`UI_YIELD_MS = 50`, `web/app.js:7372`: no synchronous task may hold the main thread longer than that; long work yields and resumes next frame) plus a per-frame CPU ledger and self-diagnosing telemetry. All numbers below were re-read from `web/app.js` (12,778 lines) and `web/assets/*`; line numbers drift, grep the symbol if one is off.

> **Verified against code (commit `bba1a8b8f6`, 2026-10-16)**, build string `v2.4.2 (2026-10-15)`. Tests that cover pieces of this: `tests/gpu_timer.test.mjs`, `tests/static_batch.test.mjs`, `tests/fly_speed.test.mjs`, `tests/browser_flows.test.mjs`, `tests/app_internals.test.mjs`.

---

## 1. Per-frame budget ledger

The comment above `MSG_BUDGET_MS` (just above `app.js:7319`) states the design: "messages 6 ms + deferred 4 ms + cull 12 ms + pick 8 ms = ~30 ms CPU, ~20 ms headroom for render". The cull comment/budget has since been reduced (see below), so the 12 ms figure is stale in the comment.

| Subsystem | Location | Budget / threshold | Mechanism |
| :--- | :--- | :--- | :--- |
| WebSocket parse | `runtime/bridge_client.js:38-66` | no budget (runs in `onmessage`); time summed in `recvMs` | JSON parse, expands `objects` and `object_transforms` batches (flat `[localId,x,y,z,qx,qy,qz,qw]` arrays) into individual queue items, reported as off-frame `ws` |
| Message queue | `runtime/message_queue.js` (FIFO with head cursor, compacts at `head >= 128`), drained via `drainMsgQueue` `app.js:7319-7328` | `MSG_BUDGET_MS = 6`, `MSG_MAX_PER_FRAME = 120` | `drain(handler, {budgetMs, maxMessages})` stops on either limit |
| Deferred frame work | `deferFrameWork` / `drainDeferredWork`, `app.js:7340-7365` | `DEFERRED_BUDGET_MS = 4`, `DEFERRED_MAX_PER_FRAME = 32` | fetch onloads, mesh settles, terrain neighbour refreshes, anisotropy migration |
| Cull / static batch pass | `updatePrimCulling`, `app.js:11496`; `CULL_BUDGET_MS = 3` (`:7379`) | 3 ms per slice, 0.25 s interval (`CULL_INTERVAL`, `:11280`) | clock check every 32 roots; resumes at `cullCursor`; shares the deadline with `staticBatcher.setDeadline()` |
| Hover picking | `PICK_BUDGET_MS = 8` (`:7383`) | 8 ms | aborts back to the old selection if exceeded; retried next throttled pass |
| Triangle budget | `TRI_BUDGET` `app.js:11486` | `1.5e6` default, `?triBudget=<millions>` (0 disables) | `tinyPxScale` x1.12 per update when over budget (cap 24), /1.03 when under `0.7*budget` (floor 1); thresholds in `assets/screen_lod.js`: hide < `2px*scale`, show > `4px*scale` |
| Texture uploads | `drainTextureUploads`, `app.js:2047` | `TEXTURE_UPLOADS_PER_FRAME = 8`, `TEXTURE_UPLOAD_QUEUE_MAX = 96` | decoded ImageBitmaps are staged and uploaded just before `renderer.render()` |
| Texture HTTP | `MAX_CONCURRENT_TEXTURE_FETCHES = 6` (`:1281`) | 6 concurrent | **not 16** as the old doc stated |
| Live-cache budgets | `queryBudgetMiB`, `app.js:1349-1359` | `TEX_BUDGET_BYTES = 256 MiB` (`?texBudgetMiB=`), `GEO_BUDGET_BYTES = 128 MiB` (`?geoBudgetMiB=`) | byte-aware LRU of unused resources; counters in diag (`cacheStats.geoEvict`, `geoFailedEvict`) |
| Persistent texture store | `assets/idb_texture_store.js` | IndexedDB `craftstorm-textures` v1, store `blobs`, key `` `${textureId}:${edge}:${format||'auto'}` ``; in-memory fallback `MEM_CAP = 200` | reloads read the blob from disk instead of HTTP; `R` key bypasses it (`skipStore`) |
| Texture wire format | `assets/texture_format.js` | `/texture/<id>[?edge=N][&format=webp]&cb=<ts>` | WebP when `detectWebPSupport()`; alpha textures stay PNG; the real `cb=Date.now()` cache-buster means URL-level HTTP caching is bypassed on purpose ("sticky 404s") |
| Movement heartbeat | `app.js:12114-12165` | `MOVE_SEND_MS = 100`, `MIN_PRESS_MS = 150` | `setInterval(sendMoveState, 100)` independent of rAF; press/release also send immediately (`setMoveKey`); a tap latches `keyLatchUntil` for 150 ms |
| Frame dt cap | `app.js:12201` | `Math.min(clock.getDelta(), 0.25)` | prediction keeps pace at 4 fps |
| Tab-hidden guard | `app.js:10209` | `visibilitychange` | `lastFrameStamp = 0; clock.getDelta()` so the first frame back does not report a multi-minute frame (one bridge log had a 5,608,338 ms sample) |

Load statistics (`animate`, `app.js:12172-12193`): `loadAvgMs` is an EMA of the rAF gap with factor `0.05` (starts at 16), `loadPeakMs` the peak since the last HUD tick, `gpuBoundEmaMs` an EMA (0.05) of `frameGap - framePhaseMs` clamped to 0..1000.

`perfStats` / `perfTotals` / `perfPeak` (`app.js:11322-11345`): phases recorded via `recordPerf(name, started)` are exactly `message, deferred, collision, cull, pick, spin, texanim, anim, upload, render, probe, hud` (EMA 0.15; `perfTotals` never reset). Off-frame ledger `offFrameMs`/`offFrameTotals()` (`:11356-11375`): `ws` (from `bridge.recvMs`), `meshDress`, `meshWorker`, `sculptWorker` (via `offFrame()` wrappers) and `bvh` (from `bvhBuildStats.buildMs`).

---

## 2. Adaptive resolution, load shedding and auto-rescue

### Dynamic resolution (`updateRenderScale`, `app.js:175-218`)
Skipped when a fixed resolution is chosen (HUD "Resolution" select, `craftstorm_resolution_v2` in localStorage, `'auto'` default) or while the perf probe runs.
- Starts at `renderPixelRatio = 0.5`, bounds `MIN_PIXEL_RATIO` (0.6 weak GPU / 0.75) and `MAX_PIXEL_RATIO` (1.0 weak GPU / 1.5, capped by devicePixelRatio).
- **Hitch snap**: a single frame > 500 ms jumps straight to the floor (`shedLevel >= 2 ? 0.5 : MIN_PIXEL_RATIO`) with a 0.5 s cooldown.
- Otherwise it works on an EMA (`renderScaleEmaMs`, factor 0.1 of `min(frameMs,200)`): `> 26 ms` is slow -> step down by 0.1 (cooldown 1 s); `< 19 ms` is good -> after 5 s accumulated (`renderScaleGoodT`) step up by 0.05 (cooldown 2 s); 19..26 ms is a dead zone. Values are rounded to 0.05.

### Two-level load shedding (`updateShedLevel`, `app.js:11425`)
| Level | Trigger | Effects (verified call sites) |
| :--- | :--- | :--- |
| 1 trim | `loadAvgMs > 40` or `loadPeakMs > 150` | cull interval x2 (0.5 s, `:12650`); hover `pickObject()` only every 3rd frame (`:12657`); far (> `LOD_DISTANCE = 40`) other avatars pose every 12th frame (`:12620`); teleport hint refresh 250 ms instead of 100 (`:9904`) |
| 2 severe | `loadAvgMs > 80` or `loadPeakMs > 300` or `lastDrawCalls > 2500` | flexible-prim sway paused (`:12226`); particle systems updated every 2nd frame (`:12662`); **all** other avatars pose every 12th frame; texture anisotropy 4 -> 1 via `deferFrameWork` (`appliedAnisotropy`, `:11454`); resolution floor 0.5 |
- Cool-down: both levels are held 2.0 s after the trigger clears (`shedCooldown`) before dropping to 0.
- **Avatars are never left in T-pose**: posing is throttled with `(frameCount + lid) % every`, but only when `ud._posedOnce` is already true (`:12621`), commit `87d061768f`. Self avatar, movement and collision are never shed.
- **Shadow rescue**: `loadAvgMs > 250` sets `shadowsShed`, `sun.castShadow = false` (`:11439`). Recovery needs `SHADOWS_DEFAULT` (i.e. `?shadows=1`), not safe mode, `shedLevel === 0`, `loadAvgMs < 30` for > 10 s (`:11442`). Because shadows are off unless `?shadows=1`, this path only matters for that mode.

### `autoReduceLoad` (`app.js:11399`)
Runs only after login and not during the perf probe. When `loadAvgMs > 100` accumulates for >= 3 s (`autoGuardLowT`) and the 6 s cooldown is over: if a fixed resolution is > 1x, set it to 1x (persisted, chat line "Very slow frames: resolution lowered to 1x."); otherwise lower `DRAW_DISTANCE` by 30 % (`x0.7`, rounded) down to `AUTO_DD_FLOOR = 32` m and send it to the bridge (`sendDrawDistance`). Never raises anything back. (The comment above the function still says "5 s below ~4 fps ... a quarter per step (10 s apart)"; the code is the 3 s / 0.7 / 6 s version.)

---

## 3. Instruments

### Disjoint GPU timer (`assets/gpu_timer.js`, 149 lines; created at `app.js:137`)
- `createGPUTimer(gl)`: needs `EXT_disjoint_timer_query_webgl2`; pool of **3** query slots; `state = {supported, gpuMs (EMA), pending, disjoint, samples}`.
- Query calls are on the **core `gl` context** (`gl.beginQuery(ext.TIME_ELAPSED_EXT, ...)`), only the enums come from `ext`. Slots whose `gl.isQuery()` is false are freed (avoids `INVALID_OPERATION` loops); a failed `beginQuery` does not strand a slot busy; a disjoint event drops in-flight work and keeps the old average. Tests in `tests/gpu_timer.test.mjs` (unsupported -> nulls, round trip, disjoint, stranded query, failed begin).
- Loop usage: `gpuTimer.poll()`, `beginFrame()` (`app.js:12696`) and `endFrame()` (`:12704`) around `renderer.render()`.
- `gpuDiagSnapshot()` (`:11301`) returns `{ms, exact, disjoint, supported, boundMs}`; when the extension is missing `ms` is `null` and the UI shows the `~` bound estimate.

### `P` performance probe (`runPerfProbe`, `app.js:10629`; keys at `:10175`)
`P` runs it, `Shift+P` (`copyLastPerfProbe`, `:10850`) re-copies the last report (the clipboard write after a multi-second run may be refused because user activation has expired). Commits `f24e1e54c9`, `bc2a3506f2`, `5b3574017a`.
1. **Three measurements**, each `WARM_MS = 600` + `SAMPLE_MS = 2500` of rAF gaps: `base` (as is), `lowRes` (pixel ratio x0.6, min 0.25), `noObjects` (everything except ground/water/sky/terrain/self avatar/avatars moved to layer 31 - layers, not `.visible`, because the cull pass rewrites `.visible`). Layer masks and pixel ratio are restored in `finally`.
2. Each result: `fps, ms, p95, renderMs, otherMs, phases (>=0.5 ms/frame), perCall peaks, offFrame (>=0.1 ms/frame), waitMs, minGapMs, slowFrames (>20 ms), frames, gpuMs, draws, ktris, pixelRatio`, computed by diffing `perfTotals` and `offFrameTotals()` over the window.
3. `drawCallCensus()` (`:10789`) walks the scene: `instanced`, `batched` (BatchedMesh), `avatar`, `skinned`, `transparent`, `single.{prim,sculpt,mesh,other}`, `multiMaterialObjs/Draws`, `staticDraws`, `staticMaterials`, top 12 owners by draws, plus `out.batching = staticBatcher.info()` (or `'off (?nobatch)'`).
4. **Verdict ladder** (`:10737-10766`): software renderer (`swiftshader|llvmpipe|softpipe|basic render|software`) -> "turn on hardware acceleration"; `fps >= 30 && p95 <= 40 && !shedLevel` -> "at target" (with "only just" if fps < 45 and > 50 % of frames > 20 ms); `resGain >= 1.25` -> GPU pixel-bound; `objGain >= 1.4` -> GPU geometry-bound (if exact GPU ms > 70 % of frame) or CPU draw-call bound; else the dominant non-render phase if > 25 % of frame, else "main thread busy outside the frame phases". Appends a shedding note when `shedLevel > 0` and a stutter note when `p95 > 40`.
5. Report is logged, sent as `{type:'diag', perfProbe}` to the bridge, copied to the clipboard and summarised in chat.

### `F9` 360-degree benchmark (`assets/benchmark.js`, 98 lines; `toggleBenchmark` `app.js:12070`)
`BenchRecorder`: `BENCH_DEFAULT_SECONDS = 10`, `BENCH_WARMUP_MS = 500`. The yaw is driven as `benchBaseYaw + orbitAngle()` (0..2PI over the run, `app.js:12209`) - the camera rotates in place, it does not translate. Samples frame gap, stage times (`message, collision, cull, pick, spin, texanim, anim, render` only), draws and triangles; `finish()` returns `frames, seconds, fps, frameMs{avg,p50,p95,max}, stages{avg,max}, draws, triangles`; `finishBenchmark()` adds `moved` (avatar moved during the run), `build`, `pixelRatio`, prints to console/chat and sends `{bench}` diag. F9 again cancels.

### Clipboard diag (`copyDiagToClipboard` button, `app.js:10285`; `collectDiagJson` `:10238`)
JSON with `build`, `url`, `query` (noaa, texBudgetMiB, geoBudgetMiB, safe, shadows, noinstancing, novolume), `load` (avgMs, peakMs, fps, shedLevel, shadowsShed), `phases` (perfStats EMAs), `render` (draws, tris, pixelRatio, gpu), `tex`, `geo` (mesh issue/settle counters, `estBytes`, `budgetBytes`, evictions), `workers` (mesh/sculpt/volume queue depth and oldest job age), `scene`, `player` (position, camera, measured `flySpeed {h,v,samples}`), `downloadsPaused`, `queues` (msg, deferred, propReqs), `voice`, `selected`, `collision`. **`staticBatcher.info()` is not in this JSON** - it is in the `drawCallCensus()` / probe output.

### Periodic bridge telemetry (`sendSceneDiag`, `app.js:11759`; timer `setInterval(sendSceneDiag, 3000)` at `:8662`)
Sends `{type:'diag'}` to the bridge every 3 s once logged in. "Hot" mode (`loadAvgMs > 120 || objectMeshes.size > 1500`) skips scene walks, pixel probe and thumbnail, so telemetry cannot cause the hitch; avatars always report cheap fields (mesh/visibility/ready/seat/yaw/animations). Non-hot extras: per-avatar `wornReport(lid)` (`:11688`), animesh `rigPoseDiag(mesh)` (`:11731`) with rig root rotation and mount info, `textureStateDiag()` (`:12012`; loaded/failed/pending/noImage/mapNoImage/noMap...), `runPixelProbe()` (`:11958`; 6x4 grid of 10x10 `readPixels` blocks right after render), and `nextFrameShot()`/`captureFrameShot()` (`:11927`; 128 px-wide JPEG at quality 0.6, only every 4th tick). Always: `perf` (stats + pixelRatio, draws, triangles, load, shed), `gpu` (`renderer.info.memory` textures/geometries/programs, `textureCacheSize`).

### Other hooks
`window.__craftstorm` (`app.js:5845`) exposes `instanceOf, instanceGroups, objectMeshes, objectState, primMaterialCache, INSTANCEABLE_SHAPES, volStats, ...` for tests and console. URL flags affecting performance: `?safe=1`, `?shadows=1`, `?noaa`, `?aa`, `?nobatch`, `?noinstancing`, `?novolume`, `?triBudget=`, `?texBudgetMiB=`, `?geoBudgetMiB=`, `?lights=<n>` (prim light pool `PRIM_LIGHT_SLOTS`, default 8, 4 on weak GPU, max 16, `:5054`).

---

## 4. Known gaps / unverified
- `sim_stats` / `time_dilation` in-place coalescing and the queue bypass for `region_handshake`, `login_status`, `error` described in the previous doc **do not exist** in `message_queue.js` or `bridge_client.js` (unverified whether the bridge coalesces on its side).
- `SHADOWS_DEFAULT` is false unless `?shadows=1`, so the shadow-rescue is dormant by default.
- The 3 s / 0.7 / 6 s auto-guard values are in code; its explanatory comment is stale.
- No automated test exercises `updateShedLevel`/`updateRenderScale` end-to-end (unverified; `app_internals.test.mjs` scope not fully reviewed).

---

## Corrections vs previous doc
- Budget constants for the message queue live in `app.js:7319` (`MSG_BUDGET_MS`, `MSG_MAX_PER_FRAME`), not in `runtime/message_queue.js`; the claimed `sim_stats`/`time_dilation` coalescing and bypass list is not in the code.
- Concurrent texture HTTP is **6** (`MAX_CONCURRENT_TEXTURE_FETCHES`), not 16. Upload cap 8/frame is right; `TEXTURE_UPLOAD_QUEUE_MAX = 96` added.
- Line refs were shifted (deferred work is at `app.js:7340`, not `2959`; cull at `11496` is right; movement at `12114`, not `12108`; `visibilitychange` at `10209`).
- `autoReduceLoad`: draw-distance step is x0.7 with 6 s cooldown, resolution first - matches the old text; the in-code comment (5 s / 4 fps / 10 s) is stale.
- Shadow recovery requires `?shadows=1`; the old text implied shadows were active by default.
- Dynamic resolution: old text said "step up after 5 s" but missed the EMA thresholds (26 ms down, 19 ms up), step sizes (-0.1 / +0.05), 500 ms hitch snap and the 0.5 floor under severe shed.
- `P` probe: it is a three-pass probe (base, low-res x0.6, no-objects) with 600 ms warm + 2.5 s sample each, which the old text did not describe; `Shift+P` copies the last report (correct).
- `F9` benchmark rotates yaw in place and records 8 stages (not the 12 phases of the probe).
- `collectDiagJson` does **not** include `staticBatcher.info()`; that is in `drawCallCensus()`. The tab-hidden guard, 3 s diag interval, hot thresholds (120 ms / 1500 objects), 6x4 10x10 pixel probe and 128 px thumbnail were confirmed.
- The "11,000-object" claim comes from code comments (`UI_YIELD_MS` note), not a measured test (unverified).
