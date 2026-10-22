---
tags:
  - craftsim
  - navmesh
  - pathfinding
  - npc
  - lsl
aliases:
  - navmesh
  - pathfinding
  - DotRecast
  - llCreateCharacter
---

# Module - Navmesh & Pathfinding

← Back to [[CraftSim]] | Related: [[Avatar, NPC & Region Modules]], [[Module - Pathfinding, Experience, Materials & Avatar Utilities]], [[Upstream OpenSim vs CraftSim - Divergence]], [[Performance & Benchmarks]]

Second Life has "pathfinding": the region builds a **navigation mesh** (navmesh) of where a character can walk, and scripts use it to move objects around obstacles with `llNavigateTo`, `llPursue` and friends. CraftSim implements this with **DotRecast** (a .NET port of Recast and Detour). Code: `OpenSim/Region/OptionalModules/World/Pathfinding/` (five files, about 1,900 lines). Read from `origin/master` at `181fb0ae86`.

> [!INFO] Overview page
> [[Module - Pathfinding, Experience, Materials & Avatar Utilities]] covers pathfinding in one section among others. This page is the full reference for the navmesh itself.

## 1. Upstream OpenSimulator

Upstream has **no navmesh and no character API**: none of `llCreateCharacter`, `llNavigateTo`, `llPursue` or DotRecast exist in the source (checked on the June 2025 baseline tag and again on upstream master, HEAD `58c2118`, 2026-10-16). Its only pathfinding code is read-back of `OBJECT_PATHFINDING_TYPE` and `OBJECT_CHARACTER_TIME` in `llGetObjectDetails`, which is stub-level. A script calling `llNavigateTo` fails to compile there. Everything below is fork-only (commits `a733ddd015` navmesh, `16fb60e5c2` crowd steering, `caa58111a2` LSL functions, `e5370c460f` OSSL functions, all 2026-08-30; incremental and toggleable in `257c08f116`, 2026-09-02).

## 2. Is it on? (read this first)

> [!WARNING] The ini comment and the code disagree
> `bin/OpenSimDefaults.ini` says `;;Enabled = true` with "default: true". The code is `config.GetBoolean("Enabled", false)` and the comment above it says "Defaults OFF". **The code wins: pathfinding is off unless `[Pathfinding] Enabled = true` is set.** The reason in the code: a navmesh rebuild is a real cost that a region doing bulk population (an OAR import) or with no scripted characters should not pay unasked.

Ways to turn it on or off:

| Way | Effect |
| :--- | :--- |
| `[Pathfinding] Enabled = true` | initial build at region load, then rebuilds on scene edits |
| console `pathfinding enable` / `pathfinding disable` | per selected region, no restart; disable drops the navmesh and stops rebuilding (useful before `load oar`) |
| `osNavMesh(TRUE/FALSE)` from a script | same, from LSL; `osIsNavMesh()` reads the flag |
| `osNavMeshRadius(radius)` | opt-in by use: it sets `Enabled = true` itself |

Event handlers and console commands are wired even when disabled so that enabling later needs no region restart; while disabled the handlers do nothing.

## 3. How the navmesh is built

1. **Geometry** (`PathfindingGeometryBuilder`): world-space triangles for the **terrain** (two triangles per 1 m cell at full resolution) plus **static objects**. Attachments and vegetation are excluded. This matches the existing `OBJECT_PATHFINDING_TYPE` heuristic; explicit per-linkset types (walkable, obstacle, exclusion) are not implemented because the scripting API to set them (`llSetLinksetProperties` equivalent) does not exist.
2. **Recast** (`PathfindingNavMesh.Build`): coordinates are converted from OpenSim Z-up to Recast Y-up (`PathfindingCoords`), then `RcBuilder` with the **watershed** partition produces a poly mesh and a detail mesh, which become a `DtNavMesh` (`DtNavMeshBuilder.CreateNavMeshData`). It returns `null` for "no walkable geometry" or on failure; callers treat that as "no navmesh", not as an error.
3. **Swap**: the new `DtNavMesh` and a `DtNavMeshQuery` replace the old under a lock, and `PathfindingCharacterManager.SetNavMesh` is called.

### Build settings (`[Pathfinding]`, all optional)

| Key | Default | Meaning |
| :--- | :--- | :--- |
| `CellSize` | 0.3 m | build resolution; smaller is more accurate and much slower |
| `CellHeight` | 0.2 m | vertical resolution |
| `AgentRadius` | 0.4 m | used for **erosion** (how close to a wall is walkable) and as the default character shape |
| `AgentHeight` | 1.8 m | |
| `AgentMaxClimb` | 0.4 m | tallest step |
| `AgentMaxSlope` | 45 degrees | steepest walkable slope |

Fixed in code: `RegionMinSize 8`, `RegionMergeSize 20`, `EdgeMaxLen 12`, `EdgeMaxError 1.3`, `VertsPerPoly 6`, `DetailSampleDist 6`, `DetailSampleMaxError 1`.

**Cost.** A 256 x 256 region at 0.3 m is on the order of **several hundred thousand Recast voxels** and about 130,000 terrain triangles before any object (the code comment says so). The ini advises raising `CellSize` on large regions. The code names `DotRecast.Detour.TileCache` and a coarser terrain sampling stride as the known, **not attempted** follow-ups. No rebuild timing was found in the repo, so the real cost on the 1024 x 1024 `Nexus` region is **unmeasured**.

## 4. Keeping it current

- Scene events (`OnObjectAddedToScene`, `OnObjectBeingRemovedFromScene`, `OnSceneObjectPartUpdated`, `OnTerrainTainted`) mark the navmesh dirty.
- A **2-second debounce timer** (`RebuildDebounceMs = 2000`) then runs a **whole-region** rebuild on a background task (`Util.FireAndForget`), so a burst of edits costs one rebuild.
- `SuspendAutoRebuild` / `ResumeAutoRebuild` (counted) let bulk operations hold the rebuild until they finish.
- **Incremental build** (`osNavMeshRadius`, console `pathfinding rebuild-radius <x> <y> <z> <radius>`): adds only the terrain cells and object parts within the radius that are not already recorded, keeping a running accumulation, then rebuilds from the total. Repeated calls pay only for new area; it is meant for large regions where a full rebuild is too slow. `osClearNavMesh()` / `ClearNavMesh` discards the accumulation. Terrain patches from separate calls are not vertex-shared, which Recast tolerates because it rasterises absolute coordinates.
- **A rebuild drops every moving character** (`SetNavMesh` recreates the crowd). A script whose character was walking simply stops and must call `llNavigateTo` etc. again. The code comment says this mirrors SL's own behaviour; that was not compared.

## 5. Characters and steering (`PathfindingCharacterManager`)

A "character" is any scene object group moved **kinematically**: direct position and rotation writes plus an explicit terse update, the same way `KeyframeMotion` moves a linkset, with **no physics engine involved**. Steering uses DotRecast's **`DtCrowd`** (default speed 3.5 m/s, acceleration 8 m/s squared). It has its own tick timer per region.

| LSL function | Behaviour |
| :--- | :--- |
| `llCreateCharacter(options)`, `llUpdateCharacter`, `llDeleteCharacter` | register, change, remove; options are the `CHARACTER_*` constants (taken from the SL viewer's `keywords_lsl_default.xml`) |
| `llNavigateTo(pos, options)` | walk to a point along the navmesh |
| `llPatrolPoints(points, options)` | needs at least two waypoints; `PauseAtWaypoints` option |
| `llWanderWithin(origin, dist, options)` | picks random points inside the box; each axis is at least 1 m |
| `llPursue(target, options)` | follows a target, `PURSUIT_OFFSET` supported, re-targets as it moves |
| `llEvade(target, options)` | **simplified**: keeps moving away from the pursuer's live position |
| `llFleeFrom(pos, dist, options)` | moves away from a point by at least `dist` (at least 1 m) |
| `llExecCharacterCmd` | stop and similar commands; the code notes `DtCrowd` has no eased stop or parabolic jump, so those are approximated |
| `llGetClosestNavPoint(point, options)` | nearest point on the navmesh |

Other OSSL: `osPathfindingRebuild()`, `osPathfindingHasNavMesh()` (true only when a navmesh is baked), `osNavMesh`, `osIsNavMesh`, `osNavMeshRadius`, `osClearNavMesh`. `osResetRegion()` also drops the accumulation. Console: `pathfinding status`, `enable`, `disable`, `rebuild`, `rebuild-radius`, and `pathfinding test-character <object-uuid> <x> <y> <z>` (a validation command that drives any existing object).

## 6. Alignment with Second Life

| Aspect | Second Life | CraftSim |
| :--- | :--- | :--- |
| Function names and option constants | `llCreateCharacter` family, `CHARACTER_*`, `PURSUIT_*`, `GCNP_*` | same names; constants copied from the viewer's keyword file, not from memory |
| Navmesh source | region navmesh from terrain and pathfinding-typed objects, baked by the server | terrain plus static objects, rebuilt on edit; **no per-object walkable / obstacle / exclusion types** |
| `llEvade` | looks for a hiding spot out of the pursuer's sight | **only runs away**; no line-of-sight queries |
| Rebuild invalidates characters | yes | yes (characters are dropped) |
| Movement | physics-based characters | **kinematic**, no collision with physics bodies, no avatar-style physics |
| Path quality (shapes, corners, edge cases) | reference | **not compared** |

> [!WARNING] Stale and unverified
> - The class comment in `PathfindingModule.cs` says character steering is "Phase 3/4" and this phase "only maintains a queryable DtNavMesh". `PathfindingCharacterManager` exists and is wired, so the comment is out of date.
> - The test comments refer to `PathfindingDotRecastSpikeTest` and `PathfindingCoordsTest`; whether they are in the build and pass was **not checked**.
> - Behaviour against a real viewer or against SL for path shapes was **not verified**. Whether NPC avatars (`osNpcMoveToTarget`) can use the navmesh was **not checked**; they have their own movement, see [[Avatar, NPC & Region Modules]].
