---
tags:
  - craftstorm
  - webgl
  - threejs
  - rendering
  - batching
  - bvh
  - geometry
aliases:
  - CraftStorm Rendering
  - Static Batching
  - MeshBVH
---

# CraftStorm Web - Architecture, Rendering & Batching

← Back to [[CraftStorm]] | Related: [[CraftStorm Web - Performance, Budgets & Diagnostics]], [[CraftStorm Web - Avatars, Animesh, Movement & Audio]], [[CraftStorm Bridge - LLUDP, J2K, Mesh, Anim & MCP]], [[CraftSim]]

Rendering architecture, coordinate math, procedural geometry builders, instanced/static draw-call batching and off-thread BVH raycasting of the browser viewer in `C:\Users\pakkio\w\craftstorm\web\`. Every number and `path:line` below was re-read from source; paths are relative to `web/` and line numbers refer to the checked-out `app.js` (12,778 lines).

> **Verified against code (commit `bba1a8b8f6`, 2026-10-16)**, app build string `v2.4.2 (2026-10-15)` (`app.js:37`, `package.json` version `2.4.2`). Line numbers drift on every edit; grep the symbol if a number is off.

---

## Parts

This note was split because it had grown too long. Read the parts in order or jump to the one you need.

- [[CraftStorm Web - Architecture, Rendering & Batching - Layout, coordinates and draw calls]]: module layout, coordinate system, instancing and static batching.
- [[CraftStorm Web - Architecture, Rendering & Batching - Geometry, culling and raycasting]]: procedural geometry, cull pass, MeshBVH.

## 7. Known gaps / unverified
- Draw-call savings figures (~12 ms for ~800 draws, 408 batches for 711 faces at 64 m) come from comments in `static_batch.js`, measured on a weak GPU; not re-measured.
- `WEBGL_multi_draw` usage is by three.js `BatchedMesh` internals; not verified in vendor code (unverified).
- Batched instances follow their source only once per cull pass (0.25 s), hence moving objects are excluded rather than synced per frame.
- Static batches are not culled per-instance by distance (only `perObjectFrustumCulled` inside BatchedMesh); `DRAW_DISTANCE` hides the source mesh through `.visible`, which `sync` propagates to the instance via `effectiveVisible`.

---

## Corrections vs previous doc
- Coordinate mapping and `WEAK_GPU` regex were right but regex also contains `software`; `renderPixelRatio` starts at 0.5 on **all** GPUs.
- Shadows are off by default **everywhere** (opt in with `?shadows=1`), not "heavy regions only".
- Static batch **grouping comment** says 64 m but the real `BATCH_CELL_SIZE` is 256; the old doc's constant was right, but it cited `L125-128`/`L132-145`/`app.js:11647` - real locations are `batchableMaterial` (`static_batch.js:125`), `getBatch` (`:132`), `syncStaticBatch` at `app.js:5610` (call site in the cull `finally`, `:11647`).
- Instancing: old doc listed "spheres, toruses" - torus is **excluded**; sphere_hemi/cone/cone_hemi/cylinder_hemi/prism/pyramid/tetrahedron are included; cell size 64 m is in X/Y of the SL position.
- Alpha-mask: the marker is a PNG tEXt chunk and Content-Type `image/png; alpha=mask`, with `alphaTest = 0.5` (not just "alphaTest"); blended PNGs get `alphaTest 0.05`.
- Volume builds are **not** always synchronous/worker-less: `geo_worker.js` handles `kind: 'volume'` for >= 4000 estimated tris; and sculpt normals come from `sculpt_build.js`, not defined inside `geo_worker.js`.
- Mesh URL is `/mesh/<id>?fmt=v3[&lod=low]`, not `?lod=high|medium|low|lowest&v=v3`.
- `MAX_HEAVY_PER_QUERY = 8` is **not applied** anywhere (dead export). Collision uses its own `HEAVY_TRIANGLES = 5000` gate.
- Old text implied `?nobatch`/`?noinstancing` did not exist; both do.
- Three.js "0.186.0" -> the vendored revision string is `'186'`.
- Line references in the old note (e.g. `app.js:11647`, `12724-12778` for both volume and box) were off or mixed; replaced above.
