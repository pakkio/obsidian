---
tags:
  - craftsim
  - ssb
  - npc
  - warp3d
  - experience
  - pathfinding
  - addons
---

# Avatar, NPC & Region Modules

← Back to [[CraftSim]] | Related: [[ZEngine]], [[Voice (WebRTC, Janus & SynthVoice)]], [[Performance & Benchmarks]], [[Module - Pathfinding, Experience, Materials & Avatar Utilities]], [[Module - osOcr, osPlot, osHTTP, osLex & osYacc]]

Verified against code (commit `9fea6a0332`, 2026-10-16).

CraftSim adds several region and avatar subsystems beyond upstream OpenSimulator: Server-Side Baking (SSB), AISv3 inventory, DisplayNames, NPC attachment cloning, Warp3D map-tile changes, the chart/image dynamic-texture renderers and two optional addon modules. Experience and Pathfinding are documented in detail in [[Module - Pathfinding, Experience, Materials & Avatar Utilities]]; only a summary is kept here. Paths are relative to `C:\Users\pakkio\w\craftsim`.

---

## 1. Server-Side Baking (SSB), AISv3 and Display Names

### Server-Side Baking
Files: `OpenSim/Region/ClientStack/Linden/Caps/UpdateAvatarAppearanceModule.cs`, `Caps/ServerBaking/AvatarBakeService.cs` (static `TryBake(Scene, ScenePresence, Action<string>)`, L73), `ServerBaker.cs`, `TgaResourceLoader.cs`.
- Config: `[ClientStack.LindenCaps] EnableServerSideBakingShim`. The code default is `true` (`UpdateAvatarAppearanceModule.cs:68`; also `LLClientView.cs:990`), but `bin/OpenSimDefaults.ini:847` ships `false` and `bin/OpenSim.ini:790` sets `true`, so a stock CraftSim install has it on.
- Per agent, `RegisterCaps` registers the `UpdateAvatarAppearance` simple handler (L162-). The POST must carry `cof_version` (L188); `ProcessAppearanceUpdate(scene, agentID, cofVersion, report)` (L211) bakes via `AvatarBakeService.TryBake` and **only if the bake succeeded** echoes the COF version with `IServerSideBakingClient.SendAppearanceWithCOF(agentID, visualParams, bakes, hoverZ, cofVersion)` (L240-244) - only clients implementing that interface (real UDP `LLClientView`) get the echo; others are reported as "not sent".
- Console (category `ssb`, registered once per process, L86-143): `ssb bake <uuid | first last>`; `ssb app <uuid|first last> [cof_version]` (full `ProcessAppearanceUpdate` including the COF-echo gate); `ssb http <uuid|first last> [cof_version]` (registers a throwaway real loopback HTTP handler); `ssb cof <uuid|first last> [cof_version]` (only the `SendAppearanceWithCOF` packet); `ssb test [uuid|first last]` (sweep of `app` + `http`). `cof_version` defaults to 1.
- Tests: `OpenSim/Region/ClientStack/Linden/Caps/Tests`.

### AISv3 (`AISv3Module.cs`, `INonSharedRegionModule`)
- Capability `InventoryAPIv3` is registered per agent at a unique path `/AISv3_<guid>` (`RegisterCaps`, L274-). Gate: `[ClientStack.LindenCaps] Cap_InventoryAPIv3 = "localhost"` (`bin/OpenSim.ini:799`, read at `AISv3Module.cs:131`) **and** a static runtime flag `RuntimeEnabled` toggled by the MCP/console `use_ais` / `dont_use_ais` (L97-210); trace via `trace_ais_on/off`.
- Scope per the class doc comment (L46-56): read - FetchCategoryChildren/Categories/Links/COF, FetchItem, FetchCategorySubset, FetchOrphans; write - CreateInventory, UpdateItem/UpdateCategory (including moves via `parent_id`), RemoveItem/RemoveCategory, PurgeDescendents, SlamFolder; library - read-only plus `CopyLibraryCategory`. Hypergrid inventories are not implemented; unsupported operations return 404/405 so viewers fall back to the legacy path. `MaxFolderDepth = 50` clamps `depth=*`. (The `bin/OpenSim.ini` comment at L792 still says "read-only"; the module documents write operations.)
- **PATCH /item with `hash_id`** (commit `16818e14af`, "AISv3: link the uploaded asset when an item PATCH carries hash_id"): `HandleUpdateItem` (L1084) reads `body["hash_id"]`; when non-zero and the scene has an `AgentTransactionsModule` and the agent presence exists, it calls `HandleItemUpdateFromTransaction(presence.ControllingClient, hashId, item)` (L1148-1152) so the Xfer-uploaded asset is stored and linked to the item. Without a `hash_id` it performs the usual metadata update.
- **Hypergrid library copy**: `HandleCopyLibraryCategory` (L571) calls `PushAssetToHomeGridIfForeign(ownerID, assetID, assetType)` (L675-688) for each copied non-link item; for a foreign HG user (`IsForeignUser(...)` with a non-empty home asset server) it calls `HGAssetMapper.Post(assetID, ownerID, userAssetServer)`. (The previous doc named `HGAssetService.PostAsset` and wrong line numbers.)

### Appearance fixes (OSSL side)
- `osLoadAppearance` (`OSSL_Api.cs:~3455-3500`): unpacks the LLSD notecard, `RestoreMissingBodyParts` (L3528) and `RestoreHairFromCurrentOutfit`, deletes current attachments then `RezAttachments`, `ValidateBakedTextures` (L3593) substitutes `AppearanceManager.DEFAULT_AVATAR_TEXTURE` for baked textures that cannot be reached locally, and persists via `AvatarService.SetAppearance`. A grep for `RequestRebake` in `OSSL_Api.cs` finds no occurrence, consistent with the claim that the rebake request was removed (the exact `AvatarFactory` path is not re-read here, so treat the "no rebake" detail as unverified).
- `NPCModule.ValidateBakedTextures` (`NPCModule.cs:185`) performs the same substitution for NPCs.

### Display Names
Files: `OpenSim/Region/ClientStack/Linden/Caps/BunchOfCaps/BunchOfCaps.cs` (caps), `OpenSim/Region/CoreModules/Framework/UserManagement/UserManagementModule.cs` (rules).
- Config `[DisplayNames]` (`bin/OpenSimDefaults.ini:639-651`, read at `UserManagementModule.cs:1422-`): `Enabled = true`, `MinLength = 1`, `MaxLength = 31`, `CooldownDays = 0`. **`BannedNames` does not exist** in the code (the previous doc claimed it); validation in `TrySetDisplayName` (L~437-500) is: feature enabled, local grid account only, empty or equal to first name / full legacy name means reset, trimmed length within bounds, no control characters, no-op if unchanged, and cooldown by `DisplayNameSetAt` when `CooldownDays > 0`.
- `GetDisplayNames` cap (`BunchOfCaps.cs:2220`, registered L288): every requested id must end up in `agents` or `bad_ids` (commit `a53ee20b82`, "GetDisplayNames: report unanswered ids in bad_ids"); the `answered` HashSet (L~2253) tracks resolved ids, placeholder user rows with empty `FirstName` or `FirstName == "Unkown"` (sic, L2276) are treated as unknown, and the remainder is emitted as `bad_ids` (L2305-2308) so the viewer cache does not stay at "(Loading...)".
- `SetDisplayName` cap (`BunchOfCaps.cs:2341`, registered L293 only when `DisplayNamesEnabled`): accepts `display_name` as an array (takes element `[1]` when present, else `[0]`) or a string, calls `TrySetDisplayName`, replies LLSD with `status`, `agents`, `agent`, `display_name_next_update`, etc., then broadcasts `DisplayNameUpdate` via `IEventQueue` to every root presence in the region (L2425-2433). Failure replies come from `WriteSetDisplayNameFailure` (L2329).
- Script/MCP access: `osSetDisplayName(string)` (VeryLow), `osSetAvatarDisplayName(key, string)` (High), MCP tools `get_display_name` / `set_display_name` (`JevTools.cs:144-145`).

---

## 2. NPC enhancements

Files: `OpenSim/Region/OptionalModules/World/NPC/NPCModule.cs`, `OpenSim/Region/CoreModules/Framework/InventoryAccess/InventoryAccessModule.cs`.
- **Attachment cloning**: `NPCModule` calls `CloneAppearanceAttachmentsForNPC` (~L150) and `IInventoryAccessModule.CloneInventoryItemForNPC(itemID, owner, targetAssetID)` (`NPCModule.cs:265`; interface `IInventoryAccessModule.cs:152,157`; impl `InventoryAccessModule.cs:1475-1600`). The clone is a temporary in-memory item (`Folder = UUID.Zero`) kept in `m_npcItemCache` under a lock with insertion times; once the cache exceeds 100 entries `CleanupNPCItemCache` drops entries older than 1 hour.
- **Correction to the old "gotcha"**: when the source item row is not found in the owner's inventory, the code now builds a *synthetic* "NPC Attachment" item from `targetAssetID` if one is supplied (NPCModule passes `attach.AssetID`), and only fails with a WARN (`... no targetAssetID was provided`) and `UUID.Zero` when none is given. The previous statement that cloning "silently skips" whenever the inventory row is missing is only true for callers that pass no asset id.
- **`osNpcPlayAnimationAdvanced(key npc, string anim, int priority, int duration)`** (`OSSL_Api.cs:4029`, High): priority clamped 1-10, duration clamped 1-60 s, inventory animation first then built-in names.
- **Limits**: `[NPC] MaxNumberNPCsPerScene` - code default `40` (`NPCModule.cs:54`), `bin/OpenSimDefaults.ini:2049` documents 40 (commented), `bin/OpenSim.ini:1459` sets **150**. Regions: `MaxAgents = 250` in both regions of `bin/Regions/Regions.ini` (L11, L24). Other documented NPC keys in `OpenSimDefaults.ini:2042-2060`: `AllowNotOwned`, `AllowSenseAsAvatar`, `AllowCloneOtherAvatars`, `NoNPCGroup` (all default true).

---

## 3. Experience module and DotRecast pathfinding (summary)

Full detail, options and examples are in [[Module - Pathfinding, Experience, Materials & Avatar Utilities]].
- **Experience** (`OpenSim/Region/OptionalModules/Experience/`): `[Experience] Enabled = False` by default; quotas `MaxKeyValueQuotaBytes = 134217728`, `MaxKeyBytes = 1011`, `MaxValueBytes = 4095`; backends MySQL / PGSQL / SQLite under `OpenSim/Data`; console commands `experience create|list|show|delete|state|grant|revoke|associate|disassociate`.
- **Pathfinding** (`OpenSim/Region/OptionalModules/World/Pathfinding/`): disabled by default in code (`[Pathfinding] Enabled` default `false`; the ini comment saying true is stale); console `pathfinding enable|disable|status|rebuild|rebuild-radius <x> <y> <z> <radius>|clear|test-character <object-uuid> <x> <y> <z>`; scripts use `osNavMesh`, `osIsNavMesh`, `osPathfindingRebuild`, `osPathfindingHasNavMesh`, `osNavMeshRadius`, `osClearNavMesh`; MCP tools `pathfinding_status` and `pathfinding_rebuild`. (There is no `osPathfindingEnable` function and no `osNavMesh*`-prefixed family beyond those listed.)

---

## 4. Warp3D map tiles and dynamic-texture renderers

### Warp3D (`OpenSim/Region/CoreModules/World/Warp3DMap/Warp3DImageModule.cs`)
1. **`MapTilesDirectory`** (read L139-150): empty by default (`bin/OpenSimDefaults.ini:483` shows the commented example `;MapTilesDirectory = "./localmaptiles"`); when set the directory is created and tiles are saved/loaded as `MAP-<region-uuid>.png` (L229-231). On directory creation failure the setting is cleared to `null`.
2. **Per-prim fault isolation**: `CreateAllPrims` (L448) wraps each `CreatePrim` in try/catch and logs `[Warp3D] failed to render prim <name> at <pos>: <msg>`. (The previous doc also claimed this for `CreateWater`; `CreateWater` (L330) has **no** try/catch.)
3. **`TexturePrimSize` cutoff** (L81, L514): a prim is skipped when the diagonal of its scale `sqrt(x^2+y^2+z^2)` is **smaller** than `TexturePrimSize`. The code default is `48` and `bin/OpenSimDefaults.ini:505` ships `TexturePrimSize = 48` (the old doc said 8). Upstream read the key but never used it; the code comment marks the use as a Craft-World change.
4. **Managed `ComputeAverageColor`** (L~871-894): `Marshal.ReadByte` on `bitmapData.Scan0` instead of an `unsafe` block.

### Dynamic textures
- `ImageDataRenderModule` (`OpenSim/Region/CoreModules/Scripting/ImageData/ImageDataRenderModule.cs`, content type `"imagedata"`, used by `osAIImage`, `OSSL_Api.cs:5610-5631`): draws with System.Drawing (GDI+) and uses **SkiaSharp** only to decode source images GDI+ cannot read (WebP etc.), re-encoding them as PNG first (`ToGdiReadablePng`, L~265-272). It does not use SkiaSharp for all decoding as the old doc implied.
- `ChartRenderModule` (`.../Scripting/Plot/ChartRenderModule.cs`, content type `"chart"`): line/bar/scatter/pie for `osPlot` / `osPlotFace`; details in [[Module - osOcr, osPlot, osHTTP, osLex & osYacc]].

---

## 5. Addon modules (`addon-modules/`)

`prebuild.xml:3136` includes `addon-modules/*/prebuild*.xml`. Both modules are inert unless configured.
- **`UsersLog`** (`addon-modules/UsersLog/src/UsersLog.cs`): `CheckConfig` (L145-178) requires `[UsersLog] enabled = true` (default false), `DataBase` and `ConnectionString`; `DataBase` values are `web` (JSON POST, `JSONData.cs`), `mysql` (`MySQLData.cs`) or `sqlite` (`SQLiteData.cs`). Country lookup uses MaxMind (`bin/MaxMind.Db.dll`, `bin/MaxMind.GeoIP2.dll`, `bin/Mono.Data.Sqlite.dll` present) with `<homeDir>/data/GeoLite2-Country.mmdb`; if the file is missing `AgentCountry` logs an **error** ("GeoLite2 database not found ... Country lookup disabled") and continues. It ignores NPCs on root-agent events.
- **`OpenSimSearch`** (`addon-modules/OpenSimSearch/Modules/SearchModule/OpenSearch.cs`, 725 lines): reads `[Search] SearchURL` (default empty) (L47-60). Not listed in `bin/OpenSim.ini`/`OpenSimDefaults.ini` by `[Search]` section at the time of checking (grep found no `[Search]` there) (unverified whether it is documented elsewhere).

---

## Corrections vs previous doc
- Pathfinding default: disabled in code; Warp3D `TexturePrimSize` default is 48 not 8; `BannedNames` for display names does not exist; `CreateWater` has no per-prim guard.
- AISv3 line references were wrong (`PushAssetToHomeGridIfForeign` is at L675, not 814-862); it calls `HGAssetMapper.Post`, not `HGAssetService.PostAsset`; the module is not just `FetchCOF` + PATCH but a broad AIS read/write API gated by `Cap_InventoryAPIv3` and the `use_ais`/`dont_use_ais` runtime flag.
- `EnableServerSideBakingShim`: code default true, `OpenSimDefaults.ini` false, `OpenSim.ini` true.
- NPC cloning gotcha is partly obsolete (synthetic item when `targetAssetID` supplied); NPC limit is 150 only because `OpenSim.ini` overrides the code default of 40.
- `ImageDataRenderModule` uses SkiaSharp only as a decode fallback; `UsersLog` missing database logs an error (not a "one-time notice") and supports `web`/`mysql`/`sqlite`.
- `osLoadAppearance`'s no-`RequestRebake` behaviour and the `ValidateBakedTextures` / `RestoreMissingBodyParts` helpers live in `OSSL_Api.cs` (L3528, L3593), not in an appearance module.
