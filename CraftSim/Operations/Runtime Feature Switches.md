---
tags:
  - craftsim
  - operations
  - config
aliases:
  - Features
---

# Runtime Feature Switches

← Back to [[CraftSim]] | Related: [[MCP Server & Live Diagnostics]], [[Asset Deduplication & Database]], [[Module - Server-Side Baking (SSB)]]

> From craftsim commit `cc3a401f8f` (2026-11-03). Not re-run in this pass; written from the commit message, `bin/OpenSimDefaults.ini` and `FeatureTools.cs`.

Four server features can be switched while the server runs. `OpenSim.Framework.FeatureFlags` holds the registry; the value is read at each use, so no restart is needed.

| Name | `[Features]` key | Legacy key | Off means |
| :--- | :--- | :--- | :--- |
| `asset-dedup` | `AssetDedup` | none | New writes are stored inline (hash NULL) on SQLite, MySQL, PGSQL. Deduplicated rows stay readable; `assets backfill` converts them back, `assets sweep` frees unreferenced blobs |
| `display-names` | `DisplayNames` | `[DisplayNames] Enabled` | `SetDisplayName` refused with a reason; `GetDisplayNames` echoes legacy names. The capability is always offered, so enabling reaches agents already in world |
| `ssb` | `ServerSideBaking` | `[ClientStack.LindenCaps] EnableServerSideBakingShim` | `UpdateAvatarAppearance` and the region-handshake bit are not offered; a request after switch-off gets 503 |
| `guest-login` | `GuestLogin` | none | The pooled "Guest User" login is refused with a message; guests already in world stay |

## Control surfaces
- Console (simulator and Robust): `features list`, `features enable <name>`, `features disable <name>` (`FeatureFlagsConsole.cs`).
- MCP: `feature_list`, `feature_set(name, enabled)` (`McpServer/Tools/FeatureTools.cs`). Same implementation as the console.
- Shell: `scripts/mcp-call.sh feature_set:'{"name":"guest-login","enabled":false}'` (see [[MCP Server & Live Diagnostics]]).

## Rules
- Startup value: `[Features]` key, else the legacy key, else on. A key set in `[Features]` wins over the legacy one.
- A runtime change is **not** written back to the ini; a restart returns to the startup value.
- With a separate Robust, asset storage and login read **Robust's** `[Features]` and are changed from Robust's console; the region console does not reach them.
