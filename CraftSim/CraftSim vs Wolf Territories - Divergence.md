---
tags:
  - craftsim
  - wolf
  - divergence
  - comparison
  - upstream
aliases:
  - craftsim vs wolfworld
  - Wolf Territories
  - WolfStorm
---

# CraftSim vs Wolf Territories (WolfWorld) - Divergence

← Back to [[CraftSim]] | Related: [[Upstream OpenSim vs CraftSim - Divergence]], [[Voice (WebRTC, Janus & SynthVoice)]], [[CraftStorm]], [[mcp-wolfvoice]]

Both are OpenSimulator forks that depart from upstream in a big way. This page puts them side by side and says what each adds that upstream does not.

> [!WARNING] How this was checked, and how far to trust it
> **CraftSim** is read from its own repo (see [[Upstream OpenSim vs CraftSim - Divergence]]).
> **Wolf Territories Grid** (run by IntelligentWolf Ltd; viewers **WolfStorm** in the browser and **WolfViewer** on the desktop) has **no public source** that I found. Everything about it comes from its public web pages, read on 2026-11-03: `wolfstorm.app`, `wolf-grid.com` (home, What's New, `/lsl`, `/region-api`, `/wolfvoice`, `/wolfnpc`, `/wolf-chat`, `/experiences`). The fetch summarised the pages, so details may be lossy. `wiki.wolfstormviewer.org` did not resolve and is not covered. Nothing on the Wolf side was run or tested, and "Wolf does not" below means "the pages do not say".
>
> There is **no baseline tag** for Wolf, so unlike the CraftSim page this is not a diff against `baseline-upstream-202507`.

## 1. What each one is

| | CraftSim | Wolf Territories Grid |
| :--- | :--- | :--- |
| Nature | Source fork, one developer, used for research and a small grid | Public commercial grid (home page shows 3,690 members and "82,844 square miles"), online since 2020 |
| Simulator build | `VersionInfo` 1.1.0, `Flavour.Dev`; upstream snapshot June 2025 | "OpenSimWolf 0.9.3.9333" (an example on the Region API page), BulletSim physics |
| Source | Open (this repo) | Not found. The LSL page mentions a C# file `Wolf_Api.cs`, which is not published |
| Viewers | Native C++ ([[CraftStorm Native C++ Viewer (Firestorm Fork)]]) and a web client ([[CraftStorm]]) | WolfStorm (browser, nothing to download), WolfViewer (Windows, Mac, Linux). Firestorm also works with the grid address |
| Focus | Script engine, data, AI inside scripts, server operations | Player-facing game systems: vehicles, flight, sailing, roleplay, NPCs, weather, a credit economy |

## 2. Where the two overlap

| Area | CraftSim | Wolf |
| :--- | :--- | :--- |
| Positional voice | Janus WebRTC addon with `synthvoice`; [[mcp-wolfvoice]] is the author's own Rust backend ([[Voice (WebRTC, Janus & SynthVoice)]]) | **WolfVoice**, open source (Apache 2.0), per-listener mix, fades out past about 60 m, built on Robert Adams's `os-webrtc-janus`. Same idea, same addon |
| Browser client | `web/` bridge + browser renderer ([[CraftStorm Web - Architecture, Rendering & Batching]]) | WolfStorm in a browser |
| AI in the world | `osAI*`, `osJev*`, `osML*` called from scripts; the MCP server drives the sim | WolfNPC and "Create with AI", billed in Wolf Credits |
| Physics | ubODE tuned native library, BulletSim selectable | BulletSim named; ubODE is mentioned on the LSL page |
| NPCs | Upstream `osNpc*` plus crowd collision and navmesh pathfinding ([[Module - Navmesh & Pathfinding]]) | WolfNPC: scripted characters (greeters, packs, a bartender) |
| Simulator health | `show stats`, MCP `script_engine_status`, [[MCP Server & Live Diagnostics]] | Read-only **Region Monitoring API** over HTTPS |

## 3. What Wolf adds that upstream (and CraftSim) do not

From the public pages. The grid's own count is **890 functions = 482 `ll*`, 261 `os*`, 134 `wolf*`, 9 `mod*`, 4 `ls*`**.

### 3.1 `wolf*` script API (134 functions, "available only on this grid")
- **Roleplay game engine** (the bulk): `wolfGameCreate/Set/Get/Register`, players, stats (`wolfGameDefineStat`, `wolfGameAddStat`, `wolfGameSetStatMax`), damage, heal, kill, respawn, currency (`wolfGameGive/Take/Pay`), items, pets, combat (abilities, gear, ammo, buffs, targets, cooldowns), and a player-screen layer (HUD stats, dialogs, notifications, web panels, markers, tags). The server owns game state, not the scripts.
- **Wolf Adventure** (`wolfAdventure*`): triggers, flags, chests, battles, rewards, companions. An *Adventure Studio* is listed as experimental.
- **Vehicles**: coupling points between vehicles (`wolfCoupler*`, `wolfAttach`, `wolfDetach`), `wolfDriveInput` (wheel, pedals, gear), `wolfDashboard`.
- **Weather**: `wolfSetRegionWeather`, `wolfGetRegionWeather`, `wolfSetParcelWeather` (rain, snow, fog, aurora); `wolfEnvironmentGet/Set/Remove` per region or parcel.
- **Effects and destruction**: `wolfEffect`, `wolfShakeCamera`, `wolfRollCamera`, `wolfCollapse` (a build falls into rubble), `wolfSink` with `wolfRestoreBoat` / `wolfRestoreBuilding`, `wolfBreath` (dragon fire and ice).
- **Characters**: `wolfNPC(command, value)` and `wolfNPCInfo`; reactions come back as link messages on number -7600100.
- **Plugins**: `wolfCall` (sync) and `wolfCallAsync` (result by `dataserver`) call server-side plugin functions by name.
- **Misc**: `wolfGetLastError`, `wolfGetAgentCountry`, `wolfGetAgentIPHash`, `wolfVisitorObject`.

### 3.2 Server and grid behaviour
- **Draw distance up to 4096 m** on every region (6 Oct 2026).
- **Own asset store served first** (7 Oct 2026): uploads and textures come from Wolf's store before any external storage, because a slow external provider caused upload failures.
- **Region types and age checks**: General, Moderate, Adult, Adult Content; only Adult Content needs a one-off age check (UK Online Safety Act statement, 30 Oct 2026).
- **Credits economy**: AI use is charged to the owner in Wolf Credits at the provider's price plus 10 %, with an estimate shown first, a daily limit per character, and a cost window. AI only runs when a real person is near (4 m direct, 20 m by name).
- **Group chat as "servers"**: Wolf Chat with text channels, voice rooms, pins, and Discord linking.
- **Experiences**: a *curated directory* of places with an admin review step (three listings per account). This is not the SL/OpenSim "Experience" permission system; the pages do not say how the two relate.
- **Extras**: Wolf DJ / Talk Show mixer for region owners, Lag Detector (names runaway scripts and heavy physics), flight and sailing modes with autopilots, real-place astronomy and weather.
- **Fixes shipped as grid updates**: voice reconnect loop (about half of connections dropped every minute), PBR tint and texture scale resetting after teleport. Regions pick up simulator changes when they restart.

### 3.3 Region Monitoring API (`/api/v1/regions/{id}/status`, `weather`, `weather/forecast`, `weather/text`)
Bearer-key HTTPS API, up to 20 keys per owner, 60 requests per minute, keys refused in the URL. Fields: `status`, `instance_id` (changes on restart), `uptime_seconds`, `fps`, `frame_time_ms`, `agent_count`, `physics_frame_time_ms`, `active_scripts`, `script_events_per_second`, `prims`, `prim_allowance`. Process start time comes from the Linux process, not the simulator, and the simulator's own admin, console and REST interfaces are **not** exposed. Script time per frame is always `null` because "the script engine does not measure it".

## 4. What CraftSim adds that Wolf's pages do not mention

| CraftSim | Detail |
| :--- | :--- |
| **ZEngine** script engine | LSL to CIL compiler with ZEXT syntax sugar; Wolf's pages do not name a script engine ([[ZEngine]]) |
| Per-script timing | Stopwatch-based script execution time (Wolf's API reports `null`) ([[Performance & Benchmarks]]) |
| **OsDb / OsQ / OsCache / JsonStore** | Row store with WHERE/ORDER BY, vector search, queues, TTL cache ([[Module - OsDb & Vector Search]], [[Module - OsQ, OsCache & JsonStore]]) |
| `osAI*` / `osJev*` / `osML*` / `osOcr` / `osPlot` | AI, typed decisions, embeddings, OCR and charts as script functions ([[OSSL Extensions (AI, ML, OsDb, OsQ)]]) |
| **Content-addressed asset dedup** | One blob per unique hash, backfill and sweep ([[Asset Deduplication & Database]]) |
| **Server-side baking, AISv3, navmesh pathfinding** | SL-viewer features absent from upstream ([[Module - Server-Side Baking (SSB)]], [[Module - AISv3 Inventory API]], [[Module - Navmesh & Pathfinding]]) |
| **Display names**, **pooled Guest User login** | [[Avatar, NPC & Region Modules]], [[Module - Guest User Pool]] |
| **Embedded MCP server** (78 tools) | Console, DB, scripts, breakpoints, NPCs, with JWT login ([[MCP Server & Live Diagnostics]]). Wolf's Region API is read-only and narrow by comparison |
| **Runtime feature switches** | [[Runtime Feature Switches]] |
| Script debugger | Breakpoints, globals and locals over MCP |

## 5. Design differences worth noting

1. **Where the "game" lives.** Wolf builds a game layer into the simulator (stats, money, items, pets, combat, HUD) that scripts steer through `wolfGame*`. CraftSim keeps the simulator neutral and gives scripts general tools (a database, queues, AI calls); a game would be written in ZEXT on top.
2. **Cost model.** Wolf bills AI per character through a credit system and skips the model when nobody is near. CraftSim's `osAI*` calls use the grid's own provider key, gated by a threat level per function; the repo has no per-owner billing.
3. **Operations.** Wolf gives tenants a minimal read-only API. CraftSim gives the operator a full admin surface over MCP, reached only through an ssh tunnel and JWT, and now also from the viewer's admin console (see [[CraftStorm Native C++ Viewer (Firestorm Fork)]], section 7).
4. **Openness.** CraftSim is source-available. Wolf is a service with documentation only. WolfVoice is the one open-source piece on the Wolf side, and it plugs into the same `os-webrtc-janus` addon CraftSim uses.
5. **API style.** Wolf adds a separate `wolf*` family with `wolfGetLastError` for failures. CraftSim extends the `os*` family, so script names look like OpenSim's but do not exist upstream either.
6. **Merge burden is similar.** Both carry changes in files upstream also edits (CraftSim: `LLClientView.cs`, `BunchOfCaps.cs`, `LSL_Api.cs`). Wolf's build number 0.9.3.9333 suggests it also follows upstream 0.9.3.x, but that is inference.

## 6. Things to verify before relying on this page
- Whether Wolf's `wolf*` functions live in the simulator (`Wolf_Api.cs`) or are proxied to an external service. The pages only say "C# source" and "plugins".
- Whether WolfNPC sits on upstream `osNpc*`. The page does not say.
- Which script engine Wolf runs (YEngine, XEngine or custom). Not stated.
- What "OpenSimWolf 0.9.3.9333" is exactly. A vendor rebuild of upstream 0.9.3.x is the likeliest reading.
- Whether WolfVoice is used the same way in both. CraftSim's voice note was itself rebuilt on 2026-10-17 and says the voice server is another repo.
