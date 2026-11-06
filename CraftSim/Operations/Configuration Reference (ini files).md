---
tags:
  - craftsim
  - operations
  - config
  - ini
aliases:
  - ini reference
  - OpenSimDefaults
  - Configuration Reference
---

# Configuration Reference (ini files)

← Back to [[CraftSim]] | Related: [[Build, Configuration & Self-Hosting]], [[Runtime Feature Switches]], [[Module - Guest User Pool]], [[MCP Server & Live Diagnostics]], [[Voice (WebRTC, Janus & SynthVoice)]]

> Read on 2026-11-03 from craftsim `bin/OpenSimDefaults.ini`, `bin/OpenSim.ini`, `bin/config-include/*.ini`, `bin/Robust*.ini.example` and the craftsim `README.md` configuration tables. Defaults come from the ini file and, where the two differ, the code is named. Nothing was run. This page lists what CraftSim **adds or changes**; the stock OpenSim keys are in upstream's own `OpenSimDefaults.ini`.

## 1. Which file wins

| File | Role | Tracked in git? |
| :--- | :--- | :--- |
| `bin/OpenSimDefaults.ini` | Every default. The header of `OpenSim.ini` says "DO NOT MODIFY OpenSimDefaults.ini, as your changes would be lost" on upgrade. | yes |
| `bin/OpenSim.ini` | The region simulator's overrides. A key here replaces the same key in the defaults. | `bin/*.ini` is gitignored, so a fresh clone has only the `.example` templates (see `SELF_HOSTING.md`) |
| `bin/config-include/*.ini` | Pulled in by `[Architecture] Include-Architecture` (`Standalone.ini`, `StandaloneHypergrid.ini`, `Grid.ini`, `GridHypergrid.ini`) and by `Include-osslDefaultEnable`. Each carries its own `[AssetService]`, `[LoginService]` and so on. | the shipped ones, yes; `*Common.ini` are the editable copies |
| `bin/Robust.HG.ini` (from `Robust.HG.ini.example`; `Robust.ini` from `Robust.ini.example` for a grid without Hypergrid) | The grid services process: login, assets, inventory, user accounts, the Hypergrid gatekeeper. Reads **its own** `[Features]` and `[LoginService]`. **Required** here: the tracked `OpenSim.ini` selects `GridHypergrid.ini` since 2026-11-06 | example only (the live file is gitignored) |
| `bin/Secrets.ini` (from `Secrets.ini.example`) | Referenced as `${Secrets|Name}` from any ini. Holds `McpToken`, `JanusApiSecret`, DB password. See [[Build, Configuration & Self-Hosting]] section 3 | no |

Rules of thumb:
- Put the key in the file of the **process that reads it**. Guest login and asset storage run in Robust on a split grid, so their keys go in `Robust.ini`, not `OpenSim.ini`. In a standalone setup they come from `config-include/Standalone*.ini`.
- Most of the fork's sections are in `OpenSimDefaults.ini` but **disabled or commented**; `OpenSim.ini` turns them on.
- The runtime switches in `[Features]` can be flipped without a restart; everything else in this page is read at startup.

## 2. Sections CraftSim adds to `OpenSimDefaults.ini`

| Section | Key | Default (ini) | Notes |
| :--- | :--- | :--- | :--- |
| `[Features]` | `AssetDedup`, `DisplayNames`, `ServerSideBaking`, `GuestLogin` | commented, effective on | Runtime switches; see [[Runtime Feature Switches]]. A key set here wins over the legacy key |
| `[DisplayNames]` | `Enabled` | true | Same as `[Features] DisplayNames` (which wins if both set) |
| | `MinLength` / `MaxLength` | 1 / 31 | Trimmed length bounds |
| | `CooldownDays` | 0 | Days between changes; 0 = none |
| `[Statistics]` | `NumberOfFrames` | 10 | |
| | `EnablePreciseTiming` | true | Stopwatch timing |
| | `EnableMetricsCollection` | true | Frame breakdowns, object counts, login tracking |
| | `EnableMetricsLogging` | true | |
| `[Pathfinding]` | `Enabled` | **off** (the ini comment says true; wrong, see section 5) | `PathfindingModule.cs:122` reads `GetBoolean("Enabled", false)`, so set it explicitly. See [[Module - Navmesh & Pathfinding]] |
| | `CellSize` / `CellHeight` | 0.3 / 0.2 | Navmesh voxel size; raise `CellSize` on large regions |
| | `AgentRadius` / `AgentHeight` | 0.4 / 1.8 | |
| | `AgentMaxClimb` / `AgentMaxSlope` | 0.4 / 45 | |
| `[Mcp]` | `Enabled` | false | `OpenSim.ini` sets it true |
| | `Port` | 80 | Path fixed at `/mcp-craftsim`. The production host uses 9200/tcp (section 5) |
| | `ListenAddress` | empty = all interfaces | `127.0.0.1` keeps it off the network |
| | `Token` | `${Secrets|McpToken}` in `OpenSim.ini` | Empty with keys present means JWT only |
| | `AuthorizedKeysFile` | `mcp_authorized_keys` | `ssh-ed25519` and `ssh-rsa` (2048+) lines |
| | `JwtMaxLifetimeMinutes` | 1440 | |
| | Details | | [[MCP Server & Live Diagnostics]] |
| `[JsonStore]` | `Enabled` | false | `OpenSim.ini` turns it on |
| | `MaxStoreCount` | 1024 | 0 = unlimited |
| | `MaxStringSpace` | 65536 in `OpenSim.ini` | |
| `[OsDb]` | `Enabled` | false | `OpenSim.ini` turns it on. See [[Module - OsDb & Vector Search]] |
| `[Experience]` | `Enabled` | false | |
| | `StorageProvider`, `ConnectionString` | empty | Empty falls back to `[DatabaseService]` |
| | `MaxKeyValueQuotaBytes` | 134217728 | |
| | `MaxKeyBytes` / `MaxValueBytes` | 1011 / 4095 | Match SL's documented limits |
| `[ClientStack.LindenCaps]` | `EnableServerSideBakingShim` | false | Legacy key for `ssb`; see [[Module - Server-Side Baking (SSB)]] |

## 3. Keys in `OpenSim.ini` that matter for the fork

| Section | Key | Value | Notes |
| :--- | :--- | :--- | :--- |
| `[Startup]` | `DefaultScriptEngine` | `"ZEngine"` | |
| `[ZEngine]` | `Enabled` | true | |
| | `NumWorkers` | 4 | Script worker threads |
| | `MinTimerInterval` | 0.01 | |
| | `MaxEventSliceSeconds` | 30 | Abort a runaway event |
| | `MaxWorkerQuantumMilliseconds` | 5 | Fairness yield; 0 disables |
| | `EnableInclude` | true | `#include` preprocessor |
| | `IncludeTrustedCreators` | one creator UUID | Gate for `#include <uuid>` |
| | `ZEXT` | true | Allow the non-standard language extensions. See [[ZEngine]] |
| `[OSSL]` | `Include-osslDefaultEnable` | `config-include/osslDefaultEnable.ini` | Without it the OSSL functions are disabled |
| | `Allow_osAI*`, `Allow_osJev*` | true | These are `ThreatLevel.High`, above the blanket level, so each must be enabled. See [[OSSL Extensions (AI, ML, OsDb, OsQ)]] |
| `[DbConsole]` | `Enabled` | true | Read-only `db` console commands and `db_*` MCP tools |
| | `AllowWrite` | false | Lets `db exec` change data; no undo |
| | `MaxRows` / `MaxCellChars` | 100 / 120 | |
| `[WebRtcVoice]` | `Enabled` | true | os-webrtc-janus addon with the synthvoice backend. See [[Voice (WebRTC, Janus & SynthVoice)]] |
| | `SpatialVoiceService`, `NonSpatialVoiceService` | `WebRtcVoice.dll:WebRtcVoiceServiceConnector` | |
| | `WebRtcVoiceServerURI` | an https URL | HTTPS required: the SDP carries the DTLS fingerprint |
| | `StunServers` | `stun:stun.l.google.com:19302` | Sent to the viewer as `stun-servers` |
| | `MessageDetails` | false | Verbose debug |
| `[WebRTCVoice]` (note the spelling) | `enabled` | false | The internal module; the two sections register the same caps, so only one may be on |
| | `Backend` | `"wolfvoice"` or `"janus"` | |
| | `WolfVoiceServerURI`, `JanusUrl`, `JanusApiSecret` | | Keep the URIs in step with `[WebRtcVoice]` |
| `[AssetService]` | `ConnectionString` | DB connection | Dedup keys: see [[Asset Deduplication & Database]] |

## 4. Guest user pool (`[LoginService]`)

Read by `LLLoginService` in whichever process hosts login: **Robust** on a split grid, the region process in standalone mode.

| Key | Default | Where it is documented | Meaning |
| :--- | :--- | :--- | :--- |
| `GuestPoolCleanupIntervalHours` | 24 | commented in `Robust.ini.example`, `Robust.HG.ini.example`, `config-include/StandaloneHypergrid.ini` | Hours between sweeps; first run 5 min after start. **0 disables** the sweep |
| `GuestPoolKeepWarm` | 1 | same | Pool accounts the sweep never removes; negative is treated as 0 |
| `[Features] GuestLogin` | true (effective) | `OpenSimDefaults.ini` and runtime `features` command | Refuse the "Guest User" login. Set it in **Robust's** `[Features]` and use Robust's console |

How the pool works: [[Module - Guest User Pool]].

Gaps worth fixing:
- The two pool keys are now commented in `Robust.ini.example` and `Robust.HG.ini.example` (added 2026-11-03) and in `config-include/StandaloneHypergrid.ini`. They are still **not** in `Standalone.ini`, `StandaloneCommon.ini.example` or `OpenSimDefaults.ini`.
- The sweep only covers scope `UUID.Zero`.
- The login needs a real `Guest User` template account with a known password; no ini key creates it.

## 5. Inconsistencies, and how each is settled

| Question | Settled answer | Evidence |
| :--- | :--- | :--- |
| `[Pathfinding] Enabled`: the ini says `default: true` | **Off by default.** The ini comment is wrong; the code wins. Turn it on with `[Pathfinding] Enabled = true`, the `pathfinding enable` console command, or `osPathfindingEnable(TRUE)` from a script | `PathfindingModule.cs:122` `GetBoolean("Enabled", false)` and its comment "Defaults OFF"; same conclusion already in [[Module - Navmesh & Pathfinding]]. The wrong comment is still in `bin/OpenSimDefaults.ini` (a craftsim repo fix, not a vault one) |
| `[Mcp] Port`: 80 or 9200? | **80 in the tracked config and in code; 9200/tcp on the production host.** `.mcp.json` points clients at `127.0.0.1:9200` through the ssh tunnel because that is the host's override. Do not read 9200 as a default | `McpCraftsimPlugin.cs:93` `GetInt("Port", 80)`; `bin/OpenSim.ini` `[Mcp] Port = 80`; `CLAUDE.md:120` "MCP listens on 9200/tcp" (host-specific, not verifiable from the repo). Region UDP 9200 is a different protocol and port space |
| `[WebRtcVoice]` vs `[WebRTCVoice]`: one section or two? | **Two sections, two components.** `[WebRtcVoice]` is read by the `os-webrtc-janus` addon (5 files under `OpenSim/Addons/os-webrtc-janus/`); `[WebRTCVoice]` is read only by the internal `WebRTCVoiceModule.cs`. Keep the internal one `enabled = false`, because both register the same capabilities | `Configs["WebRtcVoice"]` and `Configs["WebRTCVoice"]` in the files named; the `OpenSim.ini` comment on the same point. The only thing not checked is whether Nini treats section names case-insensitively, which does not change the setup above |
| README config table vs ini | **The ini wins.** The README table is derived from it | Rule, not a code fact |
| Italian comments in `OpenSim.ini` | Not a conflict: the voice keys are commented in Italian, the rest in English. Meaning is the same | |
