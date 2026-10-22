---
tags:
  - craftsim
  - upstream
  - divergence
  - ssb
  - ais
  - navmesh
aliases:
  - upstream divergence
  - what upstream OpenSim does not do
---

# Upstream OpenSim vs CraftSim - Divergence

← Back to [[CraftSim]] | Details: [[Module - Server-Side Baking (SSB)]], [[Module - AISv3 Inventory API]], [[Performance & Benchmarks]], [[ZEngine]]

CraftSim is a fork of OpenSimulator. This page answers one question per feature: **does upstream do it, and if not, what did CraftSim add?**

> [!INFO] How this was checked
> The comparison point is the git tag **`baseline-upstream-202507`** in the craftsim repository (an upstream snapshot, last commit 2025-06-25). For each feature the source and the default configuration at that tag were searched; "upstream does not" below means *not present at that snapshot*. **Re-checked against current upstream on 2026-10-18**: a fresh shallow clone of `github.com/opensim/opensim` master (HEAD `58c2118`, 2026-10-16) gives the same result for all four features. No `UpdateAvatarAppearance`, `InventoryAPIv3`, `LibraryAPIv3`, baking code, `SetDisplayName` implementation, `llCreateCharacter`, `llNavigateTo`, `llPursue` or DotRecast anywhere in `.cs` or `.ini`; `LLClientView.cs` still has the `bit 0 signals server side texture baking` comment above a line that sets only `1UL << 63`; `Cap_SetDisplayName = ""` is still an empty config key. Limits of that check: it covers the master branch tip only (not other branches, open pull requests or the mailing list), and it searches code and config text. The CraftSim side was read from `origin/master` at `181fb0ae86`.
>
> **Why** upstream has not done each one is not recorded anywhere in the repo. Where this page says why, it is the fork's stated reason or plain inference, and it is marked as such.

## Summary

| Feature | Upstream (June 2025 snapshot) | CraftSim | Divergence |
| :--- | :--- | :--- | :--- |
| Server-Side Baking (SSB) | **Absent.** Comment in `LLClientView` names `RegionProtocols` bit 0 for server-side baking; code sets only bit 63. No `UpdateAvatarAppearance` cap. | Sets bit 0, serves the cap, composites on the server, echoes the COF version | fork-only feature |
| AIS v3 (`InventoryAPIv3`) | **Absent.** Legacy UDP and `FetchInventory` style caps (those config keys are empty by default in the snapshot). | `AISv3Module`, REST read and write, library | fork-only feature, additive |
| Display names | **Partial.** `GetDisplayNames` capability exists (`BunchOfCaps.cs`). `Cap_SetDisplayName = ""` exists as a config key but **no handler code exists**. | `SetDisplayName` cap with cooldown, live push to viewers, `[DisplayNames]` section, `osSetDisplayName`, `osSetAvatarDisplayName` | upstream read-only; fork adds the write side |
| Navmesh / pathfinding | **Absent.** Only read-back of `OBJECT_PATHFINDING_TYPE` and `OBJECT_CHARACTER_TIME` in `llGetObjectDetails`; none of `llCreateCharacter`, `llNavigateTo`, `llPursue`... exist. | `PathfindingModule` builds a DotRecast navmesh, DtCrowd steering, the SL pathfinding LSL functions, the `osPathfinding*` and `osNavMesh*` functions, see [[Module - Navmesh & Pathfinding]] | fork-only feature |
| Bakes on Mesh flag | `BakesOnMeshEnabled` advertised | same | none |

## 1. Server-Side Baking

- **Upstream**: the viewer bakes and uploads; the simulator stores and relays. The one trace of SSB is a code comment describing bit 0 right above a line that sets only bit 63.
- **CraftSim**: full path (capability, compositor, COF echo, console commands). See [[Module - Server-Side Baking (SSB)]].
- **Why the fork needed it** (stated in the repo): without bit 0 the viewer logs `Region does not support baking` and recomposes from its own COF on every login, throwing away appearances set by `osLoadAppearance` or loaded from a notecard onto an NPC.
- **Cost of the divergence**: the fork now owns a compositor. It bakes from the server's idea of the worn wearables, not from the COF, so it can disagree with the viewer. It always answers `success: true`. Both are listed as open in the SSB note.

## 2. AIS v3

- **Upstream**: no `InventoryAPIv3` anywhere (source or config) at the snapshot.
- **CraftSim**: `AISv3Module`, about 1,600 lines, added 2026-08-31, additive: the legacy path is never disabled and unsupported calls return 404 or 405 so the viewer falls back. See [[Module - AISv3 Inventory API]].
- **Why**: the repo does not say. The order of work (SSB first, AIS five weeks later) and the fact that a baking viewer fetches the COF and slams outfit links through AIS suggest AIS followed from SSB; that is inference.
- **Cost**: a second inventory path to keep consistent with the first (item upload via `hash_id`, folder versions, Hypergrid suitcase behaviour all had to be handled separately).

## 3. Display names

- **Upstream**: `GetDisplayNames` is registered when the user-management component is present (`BunchOfCaps.cs:2210`); it can only report names. `Cap_SetDisplayName = ""` is in `OpenSimDefaults.ini`, but searching all upstream C# finds no `SetDisplayName` implementation, so enabling the key would do nothing.
- **CraftSim** (commits of 2026-08-30 onward): `SetDisplayName` capability (`BunchOfCaps.cs`, `SimpleOSDMapHandler("POST", ...)`), a `[DisplayNames]` config section with a master switch (when off, `GetDisplayNames` still answers with legacy names), a **cooldown in days** (`DisplayNameCooldownDays`, 0 allows immediate changes), a live `DisplayNameUpdate` push to connected viewers, an empty string resets to the first name, `GetDisplayNames` reports unanswered ids in `bad_ids` (2026-10-13), and the LSL-side `osSetDisplayName` / `osSetAvatarDisplayName`. Tests: `BunchOfCapsDisplayNameTests.cs`.
- **Alignment with SL**: SL has a real change-limit period and name rules; the cooldown here is modelled on that idea, but which of SL's naming rules are enforced was not compared.

## 4. Navmesh and pathfinding

- **Upstream**: no navmesh and no character API. The SL pathfinding functions are simply not in `ILSL_Api`; the only pathfinding text is the read-back constants listed above. A script that calls `llNavigateTo` fails to compile on upstream.
- **CraftSim**: `OpenSim/Region/OptionalModules/World/Pathfinding/` builds a navmesh with **DotRecast** from the terrain plus static object geometry at region load, rebuilds it (debounced, background thread) when the scene or terrain changes, and drives characters with DotRecast's crowd (`DtCrowd`). LSL: `llCreateCharacter`, `llDeleteCharacter`, `llUpdateCharacter`, `llNavigateTo`, `llPursue`, `llEvade`, `llFleeFrom`, `llWanderWithin`, `llPatrolPoints`, `llGetClosestNavPoint`; OSSL: `osPathfindingHasNavMesh`, `osPathfindingRebuild`. **Off by default in the code** (`Enabled` defaults to false although the ini comment says true). Config: `[Pathfinding]` in `OpenSimDefaults.ini` (`CellSize` default 0.3 m, `CellHeight`, `AgentRadius`, `AgentHeight`, `AgentMaxClimb`, `AgentMaxSlope`). Built 2026-08-30, made incremental and toggleable 2026-09-02, with a snap-to-navmesh fallback in `CreateOrUpdateCharacter`.
- **Cost**: a full-size region at 0.3 m cell size can mean hundreds of thousands of voxels and slow rebuilds (the ini says so and tells you to raise `CellSize`). The module's class comment still says character steering is "Phase 3/4", which the code has overtaken.
- **Alignment with SL**: the function names and the use of a navmesh follow Second Life's pathfinding; whether results match SL (path shapes, character options, edge cases) was **not verified**.

## 5. Other places the fork differs (from repo notes, not re-derived)

| Area | Difference |
| :--- | :--- |
| Script engine | ZEngine (LSL to CIL, ZEXT syntax) is the default. Upstream dropped XEngine in 0.9.3.0 and keeps YEngine; see [[ZEngine]]. |
| Voice | Janus WebRTC addon and framework plumbing merged in, with `synthvoice` as backend; see [[Voice (WebRTC, Janus & SynthVoice)]]. |
| Physics | ubODE optimised native library and NPC-crowd collision work; see [[Performance & Benchmarks]]. |
| Assets | content-addressed deduplication, object canonicalization; see [[Asset Deduplication & Database]]. |
| Scripting API | `osAI*`, `osML*`, `osJev*`, `osDb*`, `osOcr`, `osPlot` and more; see [[OSSL Extensions (AI, ML, OsDb, OsQ)]]. |
| Tracking upstream | The fork still ports upstream work (for example `llGetRenderMaterial`, `llSetRenderMaterial`, `llIsLinkGLTFMaterial`, `llWorldPosToHUD`, `llMapBeacon`, and an updated `OpenMetaverse.dll`, all on 2026-10-10). Divergence is therefore a moving gap, not a fixed one. |

## 6. What this divergence costs

- **Merge burden**: every feature above touches files upstream also changes (`LLClientView.cs`, `BunchOfCaps.cs`, `LSL_Api.cs`, `OpenSimDefaults.ini`). The 2026-10 history was rewritten and force-pushed; a clone made before then no longer fast-forwards (see the note on the local checkout in [[CraftSim]]).
- **No upstream tests or users** for these paths: the fork carries its own (`AISv3ModuleTests`, `BunchOfCapsDisplayNameTests`, the `ssb test` console sweep).
- **Viewer-facing behaviour is unverified against a real Second Life server** in all four cases; the contract used is the viewer source.
