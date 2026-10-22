---
tags:
  - craftsim
  - database
  - assets
  - deduplication
  - mysql
  - postgresql
  - sqlite
---

# Asset Deduplication & Database

← Back to [[CraftSim]] | Related: [[Build, Configuration & Self-Hosting]], [[MCP Server & Live Diagnostics]], [[ZEngine]], [[Added Functions Summary]]

CraftSim adds **content-addressed asset storage** to the SQL asset database layer (MySQL, PostgreSQL, SQLite), a canonicalizer that lets identical in-world objects share one blob, a secondary-asset-server fallback, read-only dedup diagnostics (console, MCP, grid HTTP) and cheaper/atomic writes on MySQL and PostgreSQL.

> Verified against code (commit `9fea6a0332`, 2026-10-16). Sections 6-8 (code dedup, overview, history) added and checked against `47fb914f1c`, 2026-10-17.

**Two kinds of dedup live here**: *asset dedup* (sections 1-5: identical bytes or identical objects are stored once in the SQL asset database) and *code dedup* (section 6: identical script source is compiled once and shared by every running copy). Section 7 puts them side by side.

Key files:

| Area | File |
| :--- | :--- |
| MySQL store | `OpenSim/Data/MySQL/MySQLAssetData.cs` (918 lines) |
| PostgreSQL store | `OpenSim/Data/PGSQL/PGSQLAssetData.cs` (810) |
| SQLite store | `OpenSim/Data/SQLite/SQLiteAssetData.cs` (771) |
| Schema | `OpenSim/Data/{MySQL,PGSQL,SQLite}/Resources/AssetStore.migrations` |
| Canonicalizer | `OpenSim/Region/Framework/Scenes/Serialization/ObjectAssetCanonicalizer.cs` (607) |
| Service + console | `OpenSim/Services/AssetService/AssetService.cs` (`assets *` commands, lines 78-104) |
| Fallback | `OpenSim/Services/AssetService/AssetFallbackHandler.cs`, `SimpleRemoteAssetService.cs` |
| Grid diagnostics | `OpenSim/Server/Handlers/Asset/AssetDiagnosticsHandler.cs`, `AssetServerConnector.cs:96` |
| MCP | `OpenSim/ApplicationPlugins/McpServer/Tools/ConsoleTools.cs:163-215`, `AdminTools.cs:60-76` |
| Tests | `OpenSim/Tests/AssetDedupStatsTests.cs` (6), `AssetDiagnosticsTests.cs` (12), `MySqlAssetFastPathTests.cs` (3) |

---

## Parts

This note was split because it had grown too long. Read the parts in order or jump to the one you need.

- [[Asset Deduplication & Database - Storage, canonicalization and patching]]: content-addressed storage, object canonicalization, in-place `#include` patching.
- [[Asset Deduplication & Database - Fallback, DB tuning and code cache]]: secondary asset server, MySQL/PostgreSQL write path, compiled-script cache.

## 7. The layers side by side

| Layer | What is shared | Key | Where it lives | Reclaimed by |
| :--- | :--- | :--- | :--- | :--- |
| Asset blob | identical asset bytes (scripts, textures, notecards, ...) | SHA-256 of the bytes | `assetdata` table (SQLite/MySQL/PostgreSQL) | `assets sweep` |
| Object template | identical builds placed or owned differently | SHA-256 of the canonical XML (identity fields removed) | `assetdata` + per-asset `assetvariants` row | `Delete` removes the variant at once; `assets sweep` the blob |
| Identical rewrite | writing the same asset again | id + hash + name + description + type + flags + creator all equal | skipped before any transaction (MySQL, PostgreSQL) | - |
| Compiled script | identical script source | SHA-256 of the resolved source | `m_compileCache` in memory | refcount reaching zero |
| Secondary server | assets missing locally | asset UUID | fetched and stored through the normal dedup path | - |

**Operator view**

| Task | Console | MCP tool |
| :--- | :--- | :--- |
| How much is saved? | `assets dedup stats [<asset-type>]` | `asset_dedup_stats` |
| Why is this asset big or shared? | `assets inspect <asset-id>` | `inspect_asset` |
| Migrate old inline rows into the blob table | `assets backfill` | `assets_backfill` |
| Reclaim orphaned blobs | `assets sweep` | `assets_sweep` (`confirm=true`) |
| Consistent MySQL dump before a sweep | `bin/backup-db.sh` | `db_backup` |
| Grid / Hypergrid | `GET /assetdiag/dedup-stats` and `/assetdiag/inspect/<uuid>` on Robust | - |

## 8. History of the dedup work

| Date | Change |
| :--- | :--- |
| 2026-07-04/05 | Content-addressed storage for SQLite, MySQL, PostgreSQL; Object-asset canonicalization (`assetvariants`); `assets sweep`; contained-inventory objects canonicalize; P6 `assets backfill`; P7 KeyframeMotion / LinksetData / StartStr as pass-through content. A code review the same week caught two real bugs: a stale `assetvariants` row not cleared on re-store, and a blob write that was not atomic with its `assets` row. |
| 2026-07 (P2) | ZEngine compile cache gets refcounted eviction |
| 2026-08-26 `eb5dcc4d34` | Fix: every part of a linkset shared one UUID / GroupPosition placeholder, so `SceneObjectGroup` threw a duplicate key while loading and the whole linkset silently failed to load. Placeholders are now unique per part |
| 2026-08-31 / 09-01 | `adaa0da41b` read-only `asset_dedup_stats`; `a85f0e51ab` dedup stats on PostgreSQL, SQLite and Hypergrid regions |
| 2026-10-01 | `613318cbfc` PostgreSQL asset write made atomic, identical rewrites skipped; `3644a28cac` secondary-server fallback |
| 2026-10-10 | `112521feb2` asset inspection and stats dispatch through `IAssetDataPlugin` and every connector; `6e2d4ff17d` grid-mode diagnostics over HTTP, MySQL identical-rewrite skip |
| 2026-10-11 `7b92f85ffc` | MCP tools `assets_sweep`, `assets_backfill`, `db_backup`, `zengine_run`, `create_object`, `delete_object` |
| 2026-10-12 `16818e14af` | AISv3: an inventory item PATCH that carries `hash_id` now links the uploaded asset |

## Corrections vs previous doc

- **`assetvariants` does not hold KeyframeMotion, LinksetData or StartStr.** The table stores owner/last-owner/rezzer ids, creation date, root rotation, per-part positions and per-part UUIDs only. Those three payloads are passed through as hashed content (with a documented dedup miss for moving keyframe objects). "Introduced in optimization pass P7" was not verifiable in the code.
- `[AssetService] EnableRemoteDiagnostics` **defaults to true**, it does not need to be switched on.
- The `ON CONFLICT` atomic upsert is **PostgreSQL only**; MySQL uses `replace INTO` (which resets create/access time) plus `INSERT IGNORE` for blobs; SQLite uses update-or-insert under a lock with no transaction and no identical-write skip. The old text implied all backends behaved alike and that create/access time are always preserved.
- The `IsAlreadyStored` skip is not attempted for canonicalized Object assets, and only exists on MySQL and PostgreSQL.
- The fallback timeout is 6000 ms per HTTP call (metadata and data are separate calls), and the handler is only built when `EnableAssetFallback = true`.
- Added: schema/migration versions, SQL for stats/inspect/sweep/backfill, `assets_sweep` requires `confirm=true`, HG connector reports local side only, missing blobs read back as empty arrays, `Delete` removes variants immediately, test files, and the PG/SQLite forms of the in-place patch SQL.

## Verified against code (commit 9fea6a0332, 2026-10-16)
Read: the three `*AssetData.cs` backends (store/get/backfill/stats/inspect), `AssetStore.migrations`, `ObjectAssetCanonicalizer.cs` (docs + variant model), `AssetService.cs`, `AssetFallbackHandler.cs`, `SimpleRemoteAssetService.cs`, `AssetDiagnosticsHandler.cs`, `AssetServerConnector.cs`, MCP `ConsoleTools.cs`/`AdminTools.cs`, ini examples. Not verified: PostgreSQL `GetDedupStats`/`InspectAsset` SQL line by line, the ZEngine include-cache claim, and runtime behaviour (no tests were executed).
