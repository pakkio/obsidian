---
tags:
  - craftstorm
  - webgl
  - threejs
  - rendering
  - batching
  - bvh
  - geometry
---

# CraftStorm Web - Architecture, Rendering & Batching - Geometry, culling and raycasting

> Part of [[CraftStorm Web - Architecture, Rendering & Batching]]. Sections: procedural geometry, cull pass, MeshBVH.

## 4. Procedural geometry

### Stock prims and SL box (`app.js:12724-12778`)
`makeSLBoxGeometry(sx, sy, sz)` takes SL axes and emits 6 faces in Linden TE order: 0 top (+Z), 1 side -Y, 2 side +X, 3 side +Y, 4 side -X, 5 bottom (-Z); `addGroup(i*6, 6, i)`. Winding is derived from `(bl-tl) x (tr-tl) . normal > 0`, not hard-coded (4 faces used to wind inward, hidden by `DoubleSide`).

### Swept volumes (`assets/volume_build.js`)
Ported from `LLProfile`/`LLPath` (`llvolume.cpp`). Key exports: `NGON_RADIUS` (`:12`), `ngonProfile` (`:14`), `profileFor` (`:40`; `profileCurve & 0x0F`: 1 square offset -0.375, 2/3/4 triangles, 5 half-circle offset 0.5 angScale 0.5, default circle), `matchRings` (`:79`, resamples outer/hole rings by union of radial angles via `rayHitPolygon` for mixed square/round hollows), `hollowProfileFor`, `buildPath`, `volumeResolution` (`:210`: `segs = 24` (12 low), circular path steps `max(6, round(24*revolutions))`, twisted 16 (8 low), else 1), `estimateVolumeTris` (`:223`, deliberate over-estimate), `buildVolumePositions` (`:234`, non-indexed triangle soup), `computeSoupNormals`, `finishVolumePositions` (`:461`).
- **Routing**: `VOLUME_WORKER_MIN_TRIS = 4000` (`app.js:5968`): below that the build is synchronous (sub-ms to few ms); above, it goes to `geo_worker.js` (`kind: 'volume'`). The comment in `geo_worker.js` header still says volume builds stay synchronous - the code in the same file (and `app.js:5974`, `:6393`) proves otherwise.

### Sculpts (`assets/sculpt_build.js`, `sculpt_grid.js`)
- `sculptGridResolution(width, height, lod)`: LOD -> 16x16; otherwise `sizeS = clamp(height, 4, 128)`, `sizeT = clamp(width, 4, 128)` (so max 128x128 = 16,384 verts).
- `sculptGridPositions`: stitching = `sculptType & 0x07` (1 sphere, 2 torus, 3 plane, 4 cylinder), flags `0x80` invert and `0x40` mirror (`reverseHorizontal` XOR logic, triangle winding reversed when `mirror !== invert`). Pixel mapping: three-X = `(R/255-0.5)*scale[0]`, three-Y = `(B/255-0.5)*scale[2]`, three-Z = `-(G/255-0.5)*scale[1]`. `computeIndexedNormals` and `boundingSphereOfPositions` live in `sculpt_build.js` (not in the worker) and the worker calls them.
- Routing: `SCULPT_WORKER_MIN_VERTICES = 2048` (`app.js:4490`); below, the build is synchronous.

### Workers (`app.js:4398-4560`)
- **`geo_worker.js`**: two jobs `kind: 'sculpt' | 'volume'`; **`GEO_WORKER_COUNT = 2`** workers, jobs distributed round-robin (`postGeoJob`, `:4504`); a dead worker is dropped and the sync path is used if all die. Result buffers are transferred zero-copy.
- **`mesh_worker.js`**: input `{faces, scale, bind, bindN}`; concatenates all faces into `pos/norm/uv/idx` (Uint16 or Uint32 when >65,535 verts), applies optional column-major 4x4 `bind` + 3x3 `bindN` (static bind-pose bake of rigged meshes) and axis swap (`sx=scale[0], sy=scale[2], sz=scale[1]`), returns `groups = [indexOffset, count, faceIndex]` and a bounding sphere.
- **Mesh asset fetch**: `GET /mesh/<uuid>?fmt=v3` (`MESH_FORMAT = 'v3'`, `app.js:~40`, must match `MESH_CACHE_VERSION` in `bridge.py`) for the full mesh, and `GET /mesh/<uuid>?fmt=v3&lod=low` (`app.js:4327`) for the distance LOD. Not `?lod=high|medium|...&v=v3` as the previous doc said.
- **Mesh LOD swap** (`updateMeshAssetLod`, `app.js:4372`, constants `:4313`): static meshes beyond `MESH_LOD_FAR = 35` m switch to the low LOD, back inside `MESH_LOD_NEAR = 28` m (hysteresis); `MESH_LOD_MAX_INFLIGHT = 4`; rigged or no-real-low-LOD meshes are recorded in `meshLowUseless`.

---

## 5. Cull pass and screen-size gating (`updatePrimCulling`, `app.js:11496-11660`)
- Timer: `CULL_INTERVAL = 0.25` s (`:11280`), doubled to 0.5 s at `shedLevel >= 1` (`:12650`). `CULL_BUDGET_MS = 3` (`:7379`); time is checked every 32 roots (`(cullCursor & 31) === 0`) and the pass resumes at `cullCursor` next frame. Keys are snapshotted in `cullKeys` at pass start.
- Per root: linkset children are forced visible (+ mesh LOD update); instanced proxies stay visible; others get `vis = dist <= DRAW_DISTANCE`, frustum test (`_cullFrustum.intersectsObject`) when childless, then the **tiny-object gate** (`estimateScreenPx`, hide below `TINY_HIDE_PX = 2`, show above `TINY_SHOW_PX = 4`, scaled by `tinyPxScale / lodFactor`, `assets/screen_lod.js`). `TRI_BUDGET = 1.5e6` default (`?triBudget=<millions>`, `:11486`): `updateTriBudget(tris)` multiplies `tinyPxScale` by 1.12 (cap 24) above budget and divides by 1.03 (floor 1) below `0.7*budget`.
- LOD for stock prims: `wantDetail = dist <= LOD_DISTANCE * lodFactor * lodSizeScale(radius) ? 'high' : 'low'`, and cylinders/spheres/trees/rocks/cones/torus/tube/ring are rebuilt via `makePrimGeometry`.
- Collision ignores `.visible`, so culling never makes walls walk-through.

---

## 6. MeshBVH raycast acceleration (`assets/bvh.js`, `bvh_worker.js`, `raycast_budget.js`)
- **Library**: `MeshBVH`, `acceleratedRaycast` from `vendor/three-mesh-bvh.module.js` (0.9.15). Options (`meshBVHOptions`, `bvh.js:95`): `{ indirect: true, targetLeafSize: BVH_MAX_LEAF_TRIS (8), maxDepth: BVH_MAX_DEPTH (32) }`.
- **Why indirect**: a non-indirect MeshBVH reorders the geometry's index buffer in place, breaking multi-material `groups` and the main thread's own index. `buildSerializedBVH(positions, index)` (`:183`) returns `serialized` including `indirectBuffer`; `attachSerializedBVH` (`:197`) attaches to the live geometry with `setIndex: false`. Fixed in commit `e4ae6cbacf`.
- **Thresholds**: `BVH_MIN_TRIS = 5000`, `BVH_MAX_TRIS = 300000` (above that: cached `null`, brute force), `BVH_MAX_HITS_PER_MESH = 32`.
- **Queue**: `queueGeometryBVH(geometry)` (`:296`) posts to `bvh_worker.js` (module worker, created lazily in `getBvhWorker`); if no Worker it falls back to the main thread, one build per `requestIdleCallback` slice (`timeout: 500`). Call sites in `app.js`: `:4279` (sculpt), `:4368` (mesh LOD 'high'), `:4929` (mesh). Stats `bvhBuildStats = {buildMs, workerMs, queued, fallback}`. The "~250 ms for 65k tris" figure is from the `bvh_worker.js` header comment (not benchmarked here: unverified).
- **Eligibility** (`bvhEligible`): only meshes whose materials are all `DoubleSide`; single-sided meshes stay on stock three.js raycast so face culling is exactly as authored.
- **`intersectObjectsAccelerated(raycaster, objects, recursive, opts)`** (`:393`): drop-in for `intersectObjects`; heavy+eligible meshes go through the BVH if built (else count a `fallback`, queue a build and brute-force that mesh), the rest through three. Used for hover/click picking (`app.js:9729, 9980, 11096`) and collision (`collision.js` `slideMove`).
- **Broadphase**: `filterPickRoots(entries, camPos, maxDist)` (`raycast_budget.js:48`, used at `app.js:11092` with `TOUCH_RANGE`) returns `null` when nothing was filtered. `collision.js` has its **own** `HEAVY_TRIANGLES = 5000` sphere gate in `collectCandidates` (XZ spatial hash, vertical slab check, then a 3D sphere gate for heavy meshes).
- **Dead exports**: `MAX_HEAVY_PER_QUERY = 8`, `isHeavyGeometry` and `distSqToSphere` in `raycast_budget.js` are only referenced by tests (`browser_flows.test.mjs`); nothing in `app.js`/`collision.js` enforces the "8 heavy meshes per collision query" cap the previous doc claimed.

---

---
Back to [[CraftStorm Web - Architecture, Rendering & Batching]].
