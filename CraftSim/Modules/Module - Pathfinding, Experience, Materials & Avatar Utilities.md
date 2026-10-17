---
tags:
  - craftsim
  - pathfinding
  - dotrecast
  - experience
  - materials
  - gltf
  - ossl
  - lsl
  - module
aliases:
  - DotRecast Pathfinding
  - Experience Module
  - Render Materials
---

# Module - Pathfinding, Experience, Materials & Avatar Utilities

← Back to [[CraftSim]] | Related: [[Avatar, NPC & Region Modules]], [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[MCP Server & Live Diagnostics]]

Verified against code (commit `9fea6a0332`, 2026-10-16).

Covers the DotRecast pathfinding module, the Experience Tools / key-value store module, GLTF render-material and HUD LSL functions, and a few avatar/colour/random OSSL helpers. All paths are relative to `C:\Users\pakkio\w\craftsim` unless stated.

---

## 1. DotRecast Pathfinding

### Files (`OpenSim/Region/OptionalModules/World/Pathfinding/`)
| File | Lines | Role |
| :--- | :--- | :--- |
| `PathfindingModule.cs` | 688 | `PathfindingModule : INonSharedRegionModule, IPathfindingModule` - enable state, navmesh build/debounce, console commands, delegation to the character manager |
| `PathfindingNavMesh.cs` | 178 | Recast build settings (`Settings.Default()` at L49) and navmesh creation |
| `PathfindingGeometryBuilder.cs` | 285 | Terrain + prim geometry to Recast verts/faces |
| `PathfindingCharacterManager.cs` | 696 | `DtCrowd` steering of script-created characters, 100 ms tick, `path_update` events |
| `PathfindingCoords.cs` | 58 | OpenSim to Recast coordinate conversion |

NuGet: `DotRecast.Core/Recast/Detour/Detour.Crowd 2026.1.3` (`Directory.Build.props:28-31`).

LSL surface: `OpenSim/Region/ScriptEngine/Shared/Api/Implementation/LSL_Api_Pathfinding.cs` is a thin marshalling layer: `llCreateCharacter` (L98), `llUpdateCharacter` (L107), `llDeleteCharacter` (L122), `llNavigateTo` (L128), `llPatrolPoints` (L140), `llWanderWithin` (L156), `llPursue` (L165), `llEvade` (L180), `llFleeFrom` (L195), `llExecCharacterCmd` (L204), `llGetClosestNavPoint` (L213). Each character call checks `RequireCharacterSupport` and, if pathfinding is off, errors with `pathfinding is not enabled on this region`; `llGetClosestNavPoint` returns an empty list instead. `llGetObjectDetails` supports `OBJECT_PATHFINDING_TYPE` and `OBJECT_CHARACTER_TIME` (`LSL_Api_Part4.cs:1921,1934`). Constants live in `LSL_Constants.cs` (`PU_*` at ~L797-807).

### Enable state and config (important)
- **Pathfinding is OFF by default in code**: `Enabled = config != null && config.GetBoolean("Enabled", false)` (`PathfindingModule.cs:122`). The comment in `bin/OpenSimDefaults.ini:2065-2068` says "default: true" (`;;Enabled = true`) - that comment is wrong. Turn on with `[Pathfinding] Enabled = true`, console `pathfinding enable`, or `osNavMesh(TRUE)` (the code comment names a non-existent `osPathfindingEnable`).
- Build keys read at `PathfindingModule.cs:132-137`; defaults from `PathfindingNavMesh.Settings.Default()`: `CellSize = 0.3`, `CellHeight = 0.2`, `AgentRadius = 0.4`, `AgentHeight = 1.8`, `AgentMaxClimb = 0.4`, `AgentMaxSlope = 45`. Fixed (not configurable): `RegionMinSize 8`, `RegionMergeSize 20`, `EdgeMaxLen 12`, `EdgeMaxError 1.3`, `VertsPerPoly 6`, `DetailSampleDist 6`, `DetailSampleMaxError 1`.
- Auto rebuild: scene prim updates and terrain taint call `MarkDirty()`, which (re)starts a 2000 ms debounce timer (`RebuildDebounceMs`, L59); only the last edit in a burst rebuilds. `SuspendAutoRebuild` / `ResumeAutoRebuild` nest and replay one dirty edit on the outermost resume (used for bulk loads).
- `SetEnabled(false)` drops the navmesh, cancels the debounce timer and clears the character manager mesh; `SetEnabled(true)` kicks off an async full rebuild.
- Character manager constants: tick `100 ms` (`TickMs`), defaults radius 0.4, height 1.8, max accel 8, max speed 3.5 (`PathfindingCharacterManager.cs:57-65`). Movement is applied by setting prim position and `ScheduleTerseUpdate()`; no physics engine is involved.

### OSSL functions (`IOSSL_Api.cs:153-164`, impl `OSSL_Api.cs:686-748`)
| Function | Threat | Behaviour |
| :--- | :--- | :--- |
| `integer osPathfindingRebuild()` | High | Full rebuild; **rate limited to once per 30 s per script instance** (returns 0 when throttled); returns 0 if pathfinding disabled or nothing walkable; else 1 |
| `integer osPathfindingHasNavMesh()` | None | 1 if a navmesh is currently built |
| `integer osNavMesh(integer enable)` | High | Turn pathfinding on/off for the region; returns the **previous** enabled state |
| `integer osIsNavMesh()` | None | 1 if pathfinding is enabled (even if no mesh has been built yet) |
| `integer osNavMeshRadius(float radius)` | High | Additively folds terrain/prims within `radius` m of the calling prim into an accumulated mesh; `radius <= 0` returns 0; **enables pathfinding as a side effect** |
| `void osClearNavMesh()` | High | Discards the mesh and the accumulation; does not disable pathfinding |

The previous doc merged `osNavMesh` / `osIsNavMesh` into one row; they are separate functions (`IOSSL_Api.cs:158,160`).

### Console commands (`PathfindingModule.cs:515-560`, category "Regions")
`pathfinding rebuild`, `pathfinding test-character <object-uuid> <x> <y> <z>`, `pathfinding enable`, `pathfinding disable`, `pathfinding status`, `pathfinding rebuild-radius <x> <y> <z> <radius>`, `pathfinding clear`.

### MCP tools
`pathfinding_rebuild(regionName)` and `pathfinding_status(regionName)` in `OpenSim/ApplicationPlugins/McpServer/Tools/PathfindingTools.cs:45,58`. Note `pathfinding_rebuild` returns `ERROR: navmesh rebuild failed` when pathfinding is disabled (RebuildNavMesh returns false when `!Enabled`).

### `path_update` events - known gap
`FirePathUpdate` (`PathfindingCharacterManager.cs:539`) triggers `EventManager.TriggerPathUpdateEvent`. The only codes the manager actually fires are `PU_GOAL_REACHED` (L609, 617, 627), `PU_FAILURE_NO_VALID_DESTINATION` (L620), `PU_FAILURE_TARGET_GONE` (L639, 655) and `PU_SLOWDOWN_DISTANCE_REACHED` (L678). `PU_FAILURE_UNREACHABLE` is declared (L84) but no call fires it, and `PU_FAILURE_NO_NAVMESH` (0x09) exists only as a constant (`LSL_Constants.cs:807`, ZEngine builtin `ZCILCodeGen.cs:1114`) - no firing site found in `OpenSim/` (grep). Scripts relying on those two failure codes will not be notified.

### Tests
`OpenSim/Tests/LSL_ApiPathfindingTests.cs`, `PathfindingCharacterManagerTest.cs`, `PathfindingCoordsTest.cs`, `PathfindingDotRecastSpikeTest.cs`, `PathfindingNavMeshBuildTest.cs`; MCP coverage in `McpCraftsimIntegrationTests.cs:155`. The repo demo `tasks/pathfinding_demo.lsl` is a compile-coverage script (picked up by `ZLslCorpusCompileTest`), not a gameplay demo.

### Example (adapted from `tasks/pathfinding_demo.lsl`, with the enable step it lacks)
```lsl
vector gA = <128.0, 100.0, 21.0>;
vector gB = <128.0, 156.0, 21.0>;

default
{
    state_entry()
    {
        if (!osIsNavMesh())          osNavMesh(TRUE);        // off by default in code
        if (!osPathfindingHasNavMesh()) osPathfindingRebuild(); // returns 0 if throttled/disabled

        llCreateCharacter([
            CHARACTER_RADIUS, 0.4, CHARACTER_LENGTH, 1.8, CHARACTER_ORIENTATION, VERTICAL,
            CHARACTER_TYPE, CHARACTER_TYPE_A,
            CHARACTER_AVOIDANCE_MODE, AVOID_CHARACTERS | AVOID_DYNAMIC_OBSTACLES,
            CHARACTER_MAX_ACCEL, 8.0, CHARACTER_MAX_SPEED, 4.0, CHARACTER_STAY_WITHIN_PARCEL, FALSE ]);
        llUpdateCharacter([CHARACTER_DESIRED_SPEED, 3.0]);
        llPatrolPoints([gA, gB], [PATROL_PAUSE_AT_WAYPOINTS, TRUE]);
    }

    touch_start(integer n)
    {
        llPursue(llDetectedKey(0), [PURSUIT_OFFSET, <0,0,0>, REQUIRE_LINE_OF_SIGHT, FALSE, PURSUIT_GOAL_TOLERANCE, 1.5]);
    }

    path_update(integer type, list reserved)
    {
        if (type == PU_GOAL_REACHED) llOwnerSay("goal reached");
        else if (type == PU_FAILURE_NO_VALID_DESTINATION || type == PU_FAILURE_TARGET_GONE)
            llOwnerSay("pathfinding failure " + (string)type);
    }
}
```

---

## 2. Experience Tools and key-value store

### Files
`OpenSim/Region/OptionalModules/Experience/ExperienceModule.cs` (335 lines, `partial class ExperienceModule : ISharedRegionModule, IExperienceModule`, one instance per simulator process), `ExperienceConsole.cs` (263 lines), LSL side `OpenSim/Region/ScriptEngine/Shared/Api/Implementation/LSL_Api_Experience.cs`. Storage: `OpenSim/Data/IExperienceData.cs` with `MySQL/MySQLExperienceData.cs`, `PGSQL/PGSQLExperienceData.cs`, `SQLite/SQLiteExperienceData.cs`; tests `OpenSim/Tests/ExperienceDataTests.cs`.

### Configuration (`bin/OpenSimDefaults.ini:2189-2202`, read at `ExperienceModule.cs:94-113`)
| Key | Default |
| :--- | :--- |
| `[Experience] Enabled` | `False` |
| `StorageProvider`, `ConnectionString` | blank; fall back to `[DatabaseService]`. If neither is set the module logs an error and disables itself |
| `MaxKeyValueQuotaBytes` | `134217728` (128 MiB per experience) |
| `MaxKeyBytes` | `1011` |
| `MaxValueBytes` | `4095` |

Keys/values over the byte limits fail the op (`ExperienceModule.cs:302`).

### Console commands (`ExperienceConsole.cs:53-103`)
`experience create <owner-uuid> <name> [description]`, `experience list [owner-uuid]`, `experience show <uuid>`, `experience delete <uuid>`, `experience state <uuid> <valid|suspended|disabled>`, `experience grant <agent-uuid> <experience-uuid>`, `experience revoke <agent-uuid> <experience-uuid>`, `experience associate <prim-uuid|prim-name> <script-name> <experience-uuid>`, `experience disassociate <prim-uuid|prim-name> <script-name>`.

### LSL API
`llRequestExperiencePermissions(key agent, string name)`, `llAgentInExperience(key)`, `llGetExperienceDetails(key)`, `llGetExperienceErrorMessage(integer)`, `llCreateKeyValue`, `llReadKeyValue`, `llUpdateKeyValue(k, v, checkOriginal, originalValue)`, `llDeleteKeyValue`, `llKeysKeyValue(first, count)`, `llKeyCountKeyValue`, `llDataSizeKeyValue`; events `experience_permissions` / `experience_permissions_denied`.
- The experience a script runs in is `m_item.ExperienceID` (set by `experience associate`); without it permission requests post `XP_ERROR_NO_EXPERIENCE`; with the module absent `XP_ERROR_EXPERIENCES_DISABLED`.
- `llGetExperienceDetails` returns a 6-element list: `[name, owner key, experience key, state int, state message ("valid"/"suspended"/"disabled"/"unknown"), group key]` (`LSL_Api_Experience.cs:239-268`). Passing `NULL_KEY` uses the script's own experience.
- KVP calls are asynchronous: they return a dataserver request key. The reply string is `"1,<value>"` on success or `"0,<xp_error_code>"` on failure (`FormatKvpSuccess` / `FormatKvpError`, ~L309-313). Because the value is appended after a comma, split with a limit (or take everything after the first comma) if values may contain commas.

### Example (matches the structure of `tasks/experience_demo.lsl`)
```lsl
recordVisit(key agent)
{
    string k = "visits_" + (string)agent;
    llReadKeyValue(k) -> string readData;                 // ZEngine async-bind sugar
    integer ok = (integer)llGetSubString(readData, 0, 0); // "1,<value>" or "0,<code>"
    if (!ok)
    {
        llCreateKeyValue(k, "1") -> string createRes;
        llRegionSayTo(agent, 0, "First visit!");
    }
    else
    {
        string oldVal = llGetSubString(readData, 2, -1);
        string newVal = (string)((integer)oldVal + 1);
        llUpdateKeyValue(k, newVal, TRUE, oldVal) -> string updRes; // checked update
        llRegionSayTo(agent, 0, "Visit " + newVal);
    }
}
```

---

## 3. GLTF render materials, HUD projection, colour and random helpers

### Upstream-derived LSL (`LSL_Api_Part4.cs`)
- `llSetRenderMaterial(string material, integer face)` (L6577): `material` is a **task-inventory item name (type Material) or an asset UUID**; `NULL_KEY` (`UUID.ZeroString`) clears the face (or all sides with `ALL_SIDES`); empty string errors `material "" not found`. Cleans legacy material overrides (`CleanMaterialOverrides`), sets `Changed.MATERIAL` and schedules `PrimUpdateFlags.MaterialOvr | FullUpdate`. Returns silently if no materials module or `face >= number of sides`.
- `string llGetRenderMaterial(integer face)` (L6476), `integer llIsLinkGLTFMaterial(integer link, integer face)` (L6530; returns 0 when the part has no RenderMaterials entries).
- `vector llWorldPosToHUD(vector worldPos)` (L6694): returns `ZERO_VECTOR` unless the object is a HUD attachment (attach points `HUDCenter2` to `HUDBottomRight`) and the owner granted `PERMISSION_TRACK_CAMERA`; projects using the owner's camera position/axes, FOV and viewer aspect ratio. (The error message text on missing permission says `llGetCameraPos` - copied upstream wording.)
- `llMapBeacon(string simname, vector pos, list options)` (L864): requires a touching agent via detect params, or an attachment (uses the owner); `options` must have length 3 (else error).

### OSSL helpers (`OSSL_Api.cs`, `IOSSL_Api.cs`)
| Function | Threat | Notes |
| :--- | :--- | :--- |
| `osLoadAppearance(string notecard)` | (see [[Avatar, NPC & Region Modules]]) | L3455-3500: restores missing body parts, deletes and re-rezzes attachments, substitutes the default avatar texture for unreachable bakes (`ValidateBakedTextures`, L3593), persists via `AvatarService.SetAppearance` |
| `osNpcPlayAnimationAdvanced(key npc, string anim, int priority, int duration)` | High (L4031) | priority clamped to 1-10, duration to 1-60 s; looks up the animation in the prim's inventory first, then `DefaultAvatarAnimations`; requires `module.CheckPermissions` |
| `osSetDisplayName(string)` | VeryLow (L1471) | |
| `osSetAvatarDisplayName(key, string)` | High (L1477) | |
| `osValue2sRGB(float v [, min, max])` | VeryLow | **Not a general sRGB map**: normalises to 0..1 then returns a green-yellow-red ramp (`<t,1,0>` up to 0.5, then `<1,1-t,0>`); `min == max` gives v=0 |
| `osValue2Grayscale(float v [, min, max])` | VeryLow | `<v,v,v>` after clamping/normalising |
| `osRandomFloat()`, `osRandomFloat(min, max)`, `osRandomInteger(min, max)`, `osRandomColor()`, `osListRandomize(list, stride)` | VeryLow | L13935-13975; `osRandomFloat()` also has a no-argument form |

---

## Verified against code (commit 9fea6a0332, 2026-10-16)
Everything above was checked against the source files cited inline. Not verified: runtime behaviour of DtCrowd steering (only code read, plus test names), the ZEngine `->` sugar in the examples (the syntax itself is used in `tasks/async_sugar_sample.lsl`), exact PostgreSQL schema.

## Corrections vs previous doc
- Pathfinding is disabled by default in code; the previous example skipped enabling it, so `osPathfindingRebuild()` would have returned 0. The `bin/OpenSimDefaults.ini` comment claiming "default: true" is itself stale.
- `osNavMesh` and `osIsNavMesh` are two separate functions (previous table merged them); `osNavMeshRadius` enables pathfinding; `osPathfindingRebuild` is throttled to once per 30 s.
- `PU_FAILURE_UNREACHABLE` and `PU_FAILURE_NO_NAVMESH` are never fired, so the old example's `osNavMeshRadius(15.0)` recovery branch would not trigger.
- `tasks/pathfinding_demo.lsl` does not contain the old patrol/pursue toggle logic; it is a compile-coverage script.
- Experience: `Enabled` default is `False` in `bin/OpenSimDefaults.ini` (not `bin/OpenSim.ini`); KVP replies are `"1,<value>"` / `"0,<code>"`; `llGetExperienceDetails` index 4 is the state message string, index 3 the numeric state.
- `osValue2sRGB` is a green-yellow-red ramp, not an sRGB conversion; `osNpcPlayAnimationAdvanced` and the display-name functions have specific threat levels (High / VeryLow / High).
- `llSetRenderMaterial` accepts inventory material names as well as UUIDs.
