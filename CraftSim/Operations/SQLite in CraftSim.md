---
tags:
  - craftsim
  - sqlite
  - database
  - operations
aliases:
  - SQLite
  - System.Data.SQLite
---

# SQLite in CraftSim

← Back to [[CraftSim]] | Related: [[Asset Deduplication & Database]], [[Build, Configuration & Self-Hosting]], [[Performance & Benchmarks]], [[Upstream OpenSim vs CraftSim - Divergence]]

SQLite is OpenSimulator's file-based database: no server process, one `.db` file per service. In CraftSim it is the **zero-setup default** (`BUILDING.md`: "SQLite is the default and requires no separate database server") and the store the benchmarks and most tests use, while the running grid in `CLAUDE.md` is configured for **MySQL**. Read from `origin/master` at `181fb0ae86` (checked out in WSL, `~/w/craftsim`).

## 1. Which database does what

| Setup | Database | Where it is set |
| :--- | :--- | :--- |
| Fresh clone, standalone | SQLite | `config-include/storage/SQLiteStandalone.ini`, included from `StandaloneCommon.ini` |
| The configured grid in `CLAUDE.md` | **MySQL** (`opensim` database on localhost) | `bin/OpenSim.ini` `[DatabaseService]`, `[AssetService]` and the other service sections; `StandaloneCommon.ini` has the SQLite include **commented out** |
| Robust (grid mode) | SQLite-backed per `README.md` (`bin/Robust.ini`); the `bin/` directory of the WSL checkout holds `Robust.db` | `bin/Robust.ini` |
| Unit tests | SQLite, throwaway file (`AssetDedupStatsTests`, `AssetDiagnosticsTests`) | test code |
| Benchmarks | SQLite, fresh DBs each run (`tasks/stress_combined.py` deletes them) | harness |

`StandaloneCommon.ini` says: `;Include-Storage = "config-include/storage/SQLiteStandalone.ini";` (commented out, "using MySQL below"). Uncomment it and remove the MySQL `ConnectionString` lines in `OpenSim.ini` to go back to SQLite.

### One file per service (`SQLiteStandalone.ini`)

| Section | Connection string |
| :--- | :--- |
| `[DatabaseService]` (prims, regions, estates) | `URI=file:OpenSim.db;UseUTF16Encoding=True`, provider `OpenSim.Data.SQLite.dll` |
| `[AssetService]`, `[HGAssetService]` | `URI=file:Asset.db` (the two must point at the same file in Hypergrid) |
| `[InventoryService]` | `URI=file:inventory.db;UseUTF16Encoding=True` (the UTF-16 flag is for a legacy inventory store) |
| `[AvatarService]` | `URI=file:avatars.db` |
| `[AuthenticationService]` | `URI=file:auth.db` |
| `[UserAccountService]` | `URI=file:userprofiles.db` |
| `[GridUserService]` | `URI=file:griduser.db` |
| `[FriendsService]` | `URI=file:friends.db` |

Files are created relative to the working directory, which is `bin/` when started with `bin/opensim.sh`. `bin/*.db` is git-ignored.

## 2. The provider: Mono.Data.Sqlite replaced by System.Data.SQLite

- **Upstream** (tag `baseline-upstream-202507`): every SQLite data class still uses `Mono.Data.Sqlite`, the obsolete Mono provider (last updated about 2013, removed from Mono itself).
- **CraftSim** (commit `7d3917e75c`, 2026-09-02): ported to `System.Data.SQLite` 2.0.2 with native `libe_sqlite3` **3.50.4.5**. It is a mechanical rename (`Sqlite*` to `SQLite*`) across all providers, the UserStatistics module, and the reflection-based provider resolver in `SqlConsoleModule.cs` (which backs the `db` / `db_*` console commands and MCP tools; it would otherwise try to load the removed assembly).
- **Vendored in `bin/`**: `System.Data.SQLite.dll` plus native libraries (`lib64/libe_sqlite3.so`, `libe_sqlite3-arm64.so`, `e_sqlite3.dll`, two macOS builds). They are force-tracked despite the blanket `bin/*.dll` ignore, like `OpenMetaverse.dll`. The project references the DLL by `HintPath` (`bin/System.Data.SQLite.dll`) in `prebuild.xml`, not as a NuGet package.
- **Two follow-up fixes** the commit carries: `UserProfiles.migrations` was missing a `;` after a `CREATE TABLE`, which silently broke that migration on affected SQLite versions (fixed and bumped `:VERSION 3` to `4` so existing stores re-run it); and `SQLiteUserProfilesData.GetUserAppData` never set the insert command's text, appended the INSERT onto the consumed SELECT string and bound `:Id` for a `:UserId` placeholder, so it was rewritten.
- **Verification recorded in the commit**: clean solution build; the full `OpenSim.Tests` run before and after showed the **same 37 failures** (an OSSL threat-level test-config issue, unrelated), so no new regression; the SQLite dedup tests exercise real reads, writes and migrations.

## 3. Classes and schema

`OpenSim/Data/SQLite/` holds one class per store: `SQLiteAssetData`, `SQLiteFSAssetData`, `SQLiteAuthenticationData`, `SQLiteAvatarData`, `SQLiteAgentPreferencesData`, `SQLiteEstateData`, `SQLiteExperienceData`, `SQLiteFriendsData`, `SQLiteGridUserData`, `SQLiteHGTravelData`, `SQLiteMuteListData`, `SQLitePresenceData`, `SQLiteRegionData`, `SQLiteSimulationData`, `SQLiteUserAccountData`, `SQLiteUserProfilesData`, `SQLiteXInventoryData`, plus `SQLiteGenericTableHandler` and `SQLiteUtils`. Schema is versioned by `Resources/*.migrations` files (17 of them, for example `AssetStore.migrations`, `XInventoryStore.migrations`, `ExperienceStore.migrations`) that run automatically at start.

## 4. What CraftSim added on top

- **Asset deduplication** is implemented in `SQLiteAssetData` as well as MySQL and PostgreSQL (SQLite schema migration v7): table `assetdata(hash, data)` holds each blob once, keyed by SHA-256; `assets.hash` points at it; `assetvariants` holds the per-object varying fields. Writes use `insert or ignore`. Operations: `GetDedupStats`, `InspectAsset`, `SweepOrphanedAssetData` (`assets sweep`), `BackfillAssetData` (`assets backfill`). Full explanation in [[Asset Deduplication & Database]].
- **SQLite region and presence stores** for standalone and grid mode (commit `7364ff261d`, 2026-07-03).
- **`db` / `db_*` console commands and MCP tools** work for SQLite-backed instances through the provider resolver (see [[MCP Server & Live Diagnostics]]).
- **`ExperienceStore`** (Experience Tools MVP, 2026-09-01) has a SQLite store and `ExperienceDataTests`.

## 5. WAL mode

SQLite's default rollback journal blocks readers while a writer holds the file; WAL lets readers and one writer run together, which matters with a busy scene heartbeat. CraftSim has **no WAL setting in the connection strings**. The only tool is `bin/enable_wal.py` (commit `b759a21ca8`, 2026-06-28), run by hand: for each `bin/*.db` it executes `PRAGMA journal_mode=WAL` and `PRAGMA synchronous=NORMAL`.

> [!WARNING] What the script does and does not persist
> `journal_mode=WAL` is stored **in the database file**, so it survives restarts. `synchronous` is a **per-connection** setting: setting it from Python on a connection that is then closed changes nothing for the simulator, which opens its own connections with SQLite's default (`FULL`). So the script gives WAL, not the "maximum write performance" its comment promises. This is SQLite behaviour, not measured here; to get `NORMAL` the provider would have to set it on open. The effect of WAL on simulator throughput was **not measured**.

New databases created after the script ran start in rollback-journal mode again until the script is re-run.

## 6. Operating notes and traps

- **`stress_combined.py` deletes** `OpenSim.db`, `Asset.db`, `auth.db`, `avatars.db`, `friends.db`, `griduser.db`, `userprofiles.db` (and `-wal`, `-shm`) from the `bin/` next to it, then lets the sim recreate them with auto-answered estate prompts. Never run it from a `bin/` that holds data you want; use a scratch copy.
- **Copying a live SQLite DB** while the simulator runs can give an inconsistent file, especially in WAL mode where recent writes sit in `-wal`. Copy all of `.db`, `-wal`, `-shm` together after a clean shutdown, or use SQLite's backup API.
- **Hypergrid**: `[AssetService]` and `[HGAssetService]` must point to the same file, otherwise HG assets land in a second database.
- **Size**: SQLite is one file per service, so the asset DB grows as one file; dedup reduces it only for content stored after dedup was enabled unless `assets backfill` is run.
- **Concurrency**: one writer at a time per file. Fine for one standalone sim; for many regions or heavy asset traffic CraftSim's own configured grid uses MySQL, and PostgreSQL has an atomic, identical-rewrite-skipping asset store (commit `613318cbfc`).
- **Not compared**: no SQLite vs MySQL throughput measurement exists in the repo, so the choice between them is operational (setup effort, concurrency), not a measured performance one.
