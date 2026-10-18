---
tags:
  - craftsim
  - ais
  - inventory
  - viewer
  - caps
aliases:
  - AIS
  - AISv3
  - InventoryAPIv3
---

# Module - AISv3 Inventory API

← Back to [[CraftSim]] | Related: [[Module - Server-Side Baking (SSB)]], [[Upstream OpenSim vs CraftSim - Divergence]], [[MCP Server & Live Diagnostics]], [[Known and Unresolved Problems]]

AIS ("Agent Inventory Services", version 3) is the Second Life **REST inventory API**: the viewer reads and changes the inventory over HTTP capabilities instead of the old UDP messages and `FetchInventory` caps. CraftSim added `AISv3Module` (`OpenSim/Region/ClientStack/Linden/Caps/AISv3Module.cs`, about 1,600 lines) on 2026-08-31 and extended it through 2026-10-12. Read from `origin/master` at `181fb0ae86`.

SSB depends on it: a baking viewer reads the Current Outfit Folder (COF) and writes outfit links through AIS ([[Module - Server-Side Baking (SSB)]]).

> [!INFO] What "aligned to SL" means on this page
> The module's own comments name the **viewer source** (`llaisapi.cpp`, `AISAPI::...`) as the contract it follows. No Second Life server was available to compare against, so "SL" below means *what the viewer asks for and accepts*, not an observed SL server trace.

## 1. Upstream OpenSimulator and AIS

Upstream does not implement AIS. Checked against the tag `baseline-upstream-202507` (June 2025): the string `InventoryAPIv3` occurs nowhere in the source or the configuration, and the same holds for upstream master on 2026-10-16 (HEAD `58c2118`). Upstream's inventory for a viewer is the older path: UDP messages plus the `FetchInventory2`/`FetchInventoryDescendents2` style capabilities (in the baseline `Cap_FetchInventory`, `Cap_FetchLib` and `Cap_FetchLibDescendents` are all set to empty in `OpenSimDefaults.ini`, and the serving code lives in the inventory caps modules, not in a REST API).

**The divergence.** CraftSim adds the REST layer next to the old one; it does not replace it. The old path is never disabled, and an operation AIS does not handle answers 404 or 405 so the viewer falls back to it. The motivation visible in the repo is SSB: a baking viewer fetches the COF and slams outfit links through AIS. That is an inference from the SSB notes and the commit order (SSB 2026-07-27, AIS 2026-08-31), not something the commit messages state.

## 2. Enabling it

| Setting | Meaning |
| :--- | :--- |
| `[ClientStack.LindenCaps] Cap_InventoryAPIv3 = "localhost"` (`bin/OpenSim.ini:799`) | config-time gate; any other value leaves the module off |
| `use_ais` / `dont_use_ais` (console and MCP) | runtime override, **affects new logins only**; there is no revocation of a cap already handed to a connected avatar |
| `trace_ais_on` / `trace_ais_off` | verbose request and response logging (method, path, status, truncated body); default log level is Debug because a viewer calls AIS constantly |
| `check_ais [region]` (MCP: `check_ais <region>`) | reports config enable, `Cap_InventoryAPIv3` text, runtime enable, trace |

Both gates must be true. The module is an `INonSharedRegionModule` (one per region) but the toggles are static, one global switch. Commands are registered once.

> [!WARNING] Firestorm-derived viewers
> On a non-SL grid they only try AIS when the client-side debug setting `FSUseAis3Api` is enabled (`AISAPI::isAvailable`). Server config alone is not enough.

## 3. Capabilities

For each agent the module registers `InventoryAPIv3` and, when `LibraryService.LibraryRootFolder` exists, `LibraryAPIv3`. Each is served at a per-agent path `/AISv3_<uuid>` (or `/AISv3Lib_<uuid>`) as a single segment, so the HTTP server's prefix dispatch routes `.../category/...` and `.../item/...` to the handler. The cap is listed in the seed response but is **not** added to the exact-match handler table.

## 4. What is supported

| Resource and verb | Operation |
| :--- | :--- |
| `GET category/<id\|name>/children` | folder with its sub-folders and items |
| `GET category/<id\|name>/categories` | sub-folders only |
| `GET category/<id\|name>/links` | links only |
| `GET ...?depth=N&children=id,id` | depth is clamped to **50**; `children` limits the fetch to those ids |
| `GET item/<id>` | one item |
| `POST category/<id>` | CreateInventory: items, links, categories |
| `PATCH category/<id>`, `PATCH item/<id>` | update; a changed `parent_id` is a **move** (old and new parent versions are returned); `thumbnail` and `favorite` metadata; `hash_id` for uploads |
| `DELETE category/<id>`, `DELETE item/<id>` | remove |
| `DELETE category/<id>/children` | PurgeDescendents |
| `PUT category/<id>/links` | SlamFolder: replace a whole outfit's links in one call |
| `LibraryAPIv3`: `GET` children/categories/item, `COPY category/<id>` | read-only library plus CopyLibraryCategory |
| FetchCategorySubset, FetchOrphans | `children=` filter; `IInventoryService.GetOrphanedItems` (commit `ac11fae580`) |

Other methods answer **405**, unknown ids **404**, a failed update **409** with `{error: ...}`.

**Named folders.** `category/<name>` accepts the short identifiers the viewer uses (from `AISAPI::FetchCategoryChildren(const std::string&...)`): `animatn bodypart clothing current favorite gesture inbox landmark lsltext lstndfnd my_otfts notecard object outbox snapshot sound texture trash`, plus `root`. `root` goes through `GetRootFolder`, not a "folder of type Root" lookup, because the Hypergrid suitcase service deliberately returns a differently typed root.

## 5. Behaviour worth knowing

- **Upload via `hash_id`** (commit `16818e14af`, 2026-10-12): a viewer that updates an item through AIS sends the Xfer transaction id as `hash_id` with no `asset_id` and no UDP `UpdateInventoryItem`. The module hands it to `AgentTransactionsModule.HandleItemUpdateFromTransaction`, which stores the asset and links it. If the Xfer already finished, the reply carries the new `asset_id`; if still in flight, the reply shows the old one and the link happens when the last packet arrives. Before that fix the upload was discarded.
- **Thumbnail and favorite** (commit `773ae2c308`) are stored as folder and item metadata and returned as `thumbnail: {asset_id}` and `favorite: {toggled: true}`.
- **Foreign (Hypergrid) users**: after creating an item, `PushAssetToHomeGridIfForeign` posts the asset to the user's home asset server for non-link types.
- **Versions**: folder changes bump the folder version and return it, which the viewer uses to decide what to refetch.

## 6. Tests

`OpenSim/Tests/AISv3ModuleTests.cs` (added with the module, `baccf8c7f0`) and `AISv3ModuleHGSuitcaseTests.cs` drive `HandleRequest` directly with mock request and response objects, no HTTP server. They were not re-run for this page.

## 7. Alignment with the Second Life server

| Aspect | Second Life viewer's contract | CraftSim |
| :--- | :--- | :--- |
| Capability names | `InventoryAPIv3`, `LibraryAPIv3` | same (**upstream: none**) |
| Folder depth cap | viewer pre-clamps to `AISAPI::MAX_FOLDER_DEPTH_REQUEST` (50); real AIS caps rather than recurse unbounded | clamped to 50 |
| Named folders | list in `llaisapi.cpp` | same list, plus `root` |
| COF and outfit change | `FetchCOF`, link folders, `PUT .../links` (slam) | implemented |
| Item upload | `hash_id` (Xfer transaction) | implemented 2026-10-12 |
| Unsupported operation | plain error, viewer falls back | 404 or 405, fallback intact |
| Library | read plus copy | same |
| Hypergrid inventory | n/a on SL | the module's own header says **not implemented**, yet `PushAssetToHomeGridIfForeign`, the suitcase-aware `root` and `AISv3ModuleHGSuitcaseTests` exist: the class comment is out of date or HG support is partial, not settled here |

> [!WARNING] Stale text in the repo
> The comment above `Cap_InventoryAPIv3` in `bin/OpenSim.ini` still says the capability is **read-only** and "disabled by default until validated". The code implements the write operations and the running config sets it to `localhost`. Treat the code as correct; the comment should be fixed.

## 8. MCP and console

MCP tools (`AisTools.cs`): `check_ais`, `use_ais`, `dont_use_ais`, `trace_ais_on`, `trace_ais_off`. The same five exist as console commands (commit `88d1cbd662`); before that, typing `trace_ais_on` at the console answered "Invalid command".
