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

## 1. Content-addressed storage (`assetdata`)

### Schema (migration versions)
- **MySQL** `AssetStore.migrations`: v10 base `assets`; **v11** adds `assetdata(hash char(64) PK, data longblob)`, makes `assets.data` nullable, adds `assets.hash char(64)` + index `assets_hash`; **v12** adds `assetvariants`; **v13** adds `assetvariants.part_uuids`.
- **SQLite**: same layering as v7 (assetdata + `hash`), v8 (assetvariants), v9 (`part_uuids`). **PostgreSQL** has the same tables (`assetdata` at migration line 114, `assetvariants` at 134, `part_uuids` at 159).

### Write path (`StoreAsset`)
1. Truncate name/description to `AssetBase.MAX_ASSET_NAME` / `MAX_ASSET_DESC` (warn log).
2. If `asset.Type == AssetType.Object`, try `ObjectAssetCanonicalizer.TryCanonicalize` (section 2); on success hash the canonical bytes, otherwise the raw bytes.
3. `hash = Util.SHA256Hash(dataToHash)` (lowercase hex, 64 chars).
4. Write the blob once, then the `assets` row with `data = NULL` and `hash` set:
   - MySQL (`MySQLAssetData.cs:202`): one **transaction** with `INSERT IGNORE INTO assetdata`, `REPLACE INTO assetvariants` (or `DELETE FROM assetvariants WHERE id=` when not canonicalized, to clear stale overlays), then `replace INTO assets(...)`. `replace INTO` is delete+insert, so **`create_time`/`access_time` are reset to now** on every non-skipped rewrite.
   - PostgreSQL (`PGSQLAssetData.cs:197`): same transaction shape with `INSERT ... ON CONFLICT (hash) DO NOTHING` and `INSERT INTO assets ... ON CONFLICT (id) DO UPDATE SET name, description, "assetType", local, temporary, creatorid, data, hash` (create_time / access_time / asset_flags deliberately **not** in the update list).
   - SQLite (`SQLiteAssetData.cs:184`): `insert or ignore into assetdata`, `insert or replace into assetvariants`, then `UPDATE` or `INSERT` of `assets` depending on `AssetsExist`, all inside `lock(this)` but **without** an explicit transaction and without the `IsAlreadyStored` fast path.
5. The blob insert, variant write and `assets` write are in **one transaction on MySQL and PostgreSQL** so a concurrent `assets sweep` cannot delete a just-written, not-yet-referenced blob (comment at `MySQLAssetData.cs:247`). SQLite relies on the process-wide lock instead.

### Read path (`GetAsset`)
`SELECT ... data, hash FROM assets`; if `hash` is NULL/empty the inline `data` column is used (**legacy rows keep working**); else `SELECT data FROM assetdata WHERE hash=?` (a missing blob yields an **empty byte array**, not an error). For Object-type assets the `assetvariants` row, if any, is applied with `ObjectAssetCanonicalizer.Reconstruct` (`MySQLAssetData.cs:109-195`, same shape in the other two).

`Delete(id)` removes the `assets` row and the `assetvariants` row immediately; the shared `assetdata` blob is left for `assets sweep`.

### Console commands (registered by `AssetService`, only when a `MainConsole` exists)
| Command | MCP tool | Implementation |
| :--- | :--- | :--- |
| `assets dedup stats [<asset-type>]` | `asset_dedup_stats(regionName)` | `GetDedupStats(int? assetType)` per backend: total / deduped (`hash IS NOT NULL`) / legacy (`hash IS NULL`) rows, `COUNT(DISTINCT hash)` unique blobs, actual bytes (sum of each referenced blob once) vs without-dedup bytes. Argument is a numeric `AssetType` (10 = LSLText, 6 = Object). The MCP tool also prints a scripts-only (type 10) breakdown. PostgreSQL uses `octet_length`, MySQL `LENGTH`. |
| `assets inspect <asset-id>` | `inspect_asset(assetId, regionName?)` | `InspectAsset` returns size, SHA-256, `IsDeduplicated`, `SharedCopies` (`COUNT(*) WHERE hash = a.hash`), `BytesSaved = (copies-1) * blobLen`, and provenance `DB2_FALLBACK` when `AssetFlags.FromFallback (8)` is set, else `DB1_LOCAL`. If not found locally and fallback is enabled it triggers a fallback fetch first (`AssetService.cs:430-445`). |
| `assets backfill` | `assets_backfill` | `BackfillAssetData`: batches of 200 legacy rows (`WHERE hash IS NULL`), canonicalizes Objects, writes blob + variant + `UPDATE assets SET hash=?, data=NULL` per row in its own transaction; skips rows with neither hash nor data. Returns rows migrated. |
| `assets sweep` | `assets_sweep(confirm=true required)` | `DELETE FROM assetdata WHERE hash NOT IN (SELECT hash FROM assets WHERE hash IS NOT NULL)`. Providers with no blob store (FSAsset) are a no-op. |

`FSAsset`/`XAsset` stores have their own migrations and are not deduplicated by this mechanism.

### Grid and Hypergrid diagnostics
- Robust registers `AssetDiagnosticsHandler` (`GET /assetdiag/dedup-stats[/<assetType>]`, `GET /assetdiag/inspect/<uuid>`; XML responses, 404 on unknown asset, 400 on bad input) in `AssetServerConnector.cs:96`.
- **`EnableRemoteDiagnostics` defaults to `true`** (`serverConfig.GetBoolean("EnableRemoteDiagnostics", true)`); set it to `false` to turn the endpoints off. It uses the same service auth as the other asset handlers. (`Robust.ini.example:294` shows it commented out as `;EnableRemoteDiagnostics = true`.)
- Consumers: `AssetServicesConnector.GetDedupStats/InspectAsset` (`:300`, `:311`), `RegionAssetConnectorModule` (local connector only, `:588-595`), `HGAssetService` (`:231-238`, local side only; foreign assets proxied by HG are not counted).

---

## 2. Object canonicalization (`assetvariants`)

OpenSim bakes owner, creation time, rotation and world position into every serialized `SceneObjectGroup` XML, so identical builds never hash equal. `ObjectAssetCanonicalizer.TryCanonicalize(byte[] data, out byte[] canonical, out ObjectAssetVariant variant)` (`:~262`) removes those fields so the remaining "template" can be shared:

- Applies only to `AssetType.Object` XML whose root is `SceneObjectGroup` with a `SceneObjectPart`; any parse failure or missing identity field returns false and the original bytes are stored as-is (debug log). OOM is rethrown.
- **Stored per asset in `assetvariants`** (`id` = the asset's own UUID, **not** a hash): `owner_id`, `last_owner_id`, `rezzer_id`, `creation_date`, `rot_x/y/z/w` (root rotation only), `positions` (`"x,y,z;x,y,z;..."`, one per part in document order) and `part_uuids` (`"uuid;uuid;..."`, original per-part UUID, restored on `Reconstruct`).
- Owner / last owner / rezzer / creation date are collapsed to a **single shared value** (the root's). A part whose value differs (mixed-owner linkset) keeps its real value in the template, so it just dedupes less.
- Per-part UUID/FolderID in the **template** use a placeholder that must be unique per part (`PlaceholderPartIdString`) because `SceneObjectGroup.SetRootPart/AddPart` key by UUID and a shared placeholder would make the whole linkset fail to load. `LocalId` and `ParentID` are never restored (regenerated on rez).
- **Contained inventory**: `TaskInventoryItem.ItemID` gets a deterministic index-based placeholder (unique per item, since `TaskInventoryDictionary` is keyed by ItemID); `OldItemID`, `ParentPartID`, `ParentID` are placeholders and are not restored. `AssetID`, creator, group, perms, names are real content, so different contents don't dedupe.
- **`KeyframeMotion`, `<lnkstdt>` (LinksetData) and `<StartStr>` are passed through unchanged as ordinary hashed content**; they are **not** moved into `assetvariants` (the table has no columns for them). Consequence: two otherwise identical objects with active keyframe motion at different positions do not dedupe, because the serialized motion JSON embeds absolute positions (documented trade-off at `ObjectAssetCanonicalizer.cs:155-175`).

---

## 3. Patching an `#include <uuid>` script asset in place

Reads resolve content through `assets.hash -> assetdata.data`, so `UPDATE assets SET data = ...` alone has no effect (the stale `hash` still wins). To change a shared LSL library but keep its UUID:

```sql
-- MySQL (PostgreSQL: INSERT ... ON CONFLICT (hash) DO NOTHING; SQLite: INSERT OR IGNORE)
INSERT IGNORE INTO assetdata (hash, data) VALUES ('<lowercase-hex-sha256-of-new-bytes>', <new-bytes-blob>);
UPDATE assets SET hash = '<lowercase-hex-sha256-of-new-bytes>' WHERE id = '<uuid-to-preserve>';
SELECT a.id, a.name, a.hash, LENGTH(d.data) AS live_bytes
  FROM assets a LEFT JOIN assetdata d ON d.hash = a.hash WHERE a.id = '<uuid-to-preserve>';
```
- The hash must be the SHA-256 of the **exact stored bytes** (UTF-8, no BOM, LF, per the old guidance). Scripts are not Object assets, so no `assetvariants` row is involved.
- The old blob becomes orphaned if no other asset shares it; run `assets sweep` later to reclaim it. Running `assets inspect <uuid>` confirms the new hash and size.
- Scripts that `#include` the UUID must be reset/recompiled; [[ZEngine]] resolves includes at compile time and caches compiled CIL keyed by the resolved source hash (verified 2026-10-17: see section 6). The example UUID `6673bb92-08fd-4a0c-b7f8-b71519d6cde7` (brain_lib.lsl) was not checked against any database.

---

## 4. Secondary asset server fallback (`AssetFallbackHandler`)

- Config in the asset service section (`Robust.ini.example:296-303`, `StandaloneCommon.ini.example:85-92`): `EnableAssetFallback = false` (default), `FallbackAssetServers = ""` (comma-separated base URLs **without** `/assets`), `StoreRetrievedFallbackAssets = true` (default).
- `AssetService` creates the handler only if `EnableAssetFallback` is true (`AssetService.cs:60-64`), and calls `TryGetFallbackAsset(id)` when `m_Database.GetAsset` returns null (`:165-169`).
- Each URL becomes a `SimpleRemoteAssetService` that does `GET {url}/assets/{id}/metadata` then `GET {url}/assets/{id}/data` with `Timeout = 6000` ms each (`SimpleRemoteAssetService.cs:39-65`). Servers are tried in order; the first hit wins; exceptions are logged and the next server is tried.
- A hit gets `AssetFlags.FromFallback` set and, if `StoreRetrievedFallbackAssets`, is stored via the normal dedup `StoreAsset` path; `assets inspect` then reports `DB2_FALLBACK`.
- The handler logs at Info level (including the "DEBUG" lines) for every call, which is noisy on a busy region.

---

## 5. MySQL / PostgreSQL write-path optimizations

1. **`IsAlreadyStored` fast path** (MySQL `:351`, PGSQL `:375`): one indexed `SELECT 1 FROM assets a WHERE id, hash, name, description, assetType, local, temporary, asset_flags, CreatorID all equal AND NOT EXISTS (assetvariants)`. A match returns `true` without opening a write transaction. Skipped for canonicalized Objects (`variant != null`) because comparing every `assetvariants` column was judged not worth it. A failed check is logged and the normal write proceeds. A skipped write leaves `create_time` / `access_time` untouched (on MySQL the normal path would reset them). Tests: `MySqlAssetFastPathTests`. SQLite has no such path.
2. **PostgreSQL atomic upsert**: the previous `UPDATE; INSERT ... WHERE NOT EXISTS` pair could fail with an `assets_pkey` violation when several viewers triggered concurrent fallback fetches for one missing asset; `ON CONFLICT (id) DO UPDATE` settles it in the database (comment `PGSQLAssetData.cs:204-211`). MySQL uses `replace INTO` for the same purpose.

---

## 6. Code dedup: the ZEngine compiled-script cache

Many prims run the same script (vendors, timers, doors, a library copied into hundreds of objects). ZEngine compiles each distinct source **once per region** and every instance shares the result. This is separate from the asset blobs above: it deduplicates *compiled code in memory*, not stored bytes.

| Piece | Where | What it does |
| :--- | :--- | :--- |
| `m_compileCache` | `ZEngine.cs:113` | `ConcurrentDictionary<string, CompileCacheEntry>`, one per region's script engine |
| Key | `ZEngineInstance.ComputeSourceHash` (`:2948`) | SHA-256 (hex) of the **source text handed to the compiler**, i.e. after `#include` / `#define` have been resolved |
| `CompileWithCache` | `ZEngineInstance.cs:2825` | `GetOrAdd(key, ...)` then `Interlocked.Increment(entry.Refcount)`; binds the instance to the entry (`BoundCacheEntry`) |
| `Lazy<ZCompiledScript>` | same, `ExecutionAndPublication` | when many identical scripts rez at once (OAR load) exactly one compiles, the rest wait for its result |
| Shared object | `ZCompiledScript` | immutable after codegen; all mutable state (typed globals, locals, call depth) lives in each instance's own `ZScriptContext`, so sharing is safe |
| `CompileCacheEntry` | `ZEngineInstance.cs:3017` | `Key`, `Lazy`, `Refcount` |
| Eviction | `ReleaseCompileCacheEntry` (`ZEngine.cs:1758`) | `Interlocked.Decrement`; at 0 the entry is removed with a compare-and-remove (`ICollection<KeyValuePair>.Remove`), so a concurrent re-rez of the same source that bumped the count back to 1 never loses its live entry |

**Properties worth knowing**

- **Failures are cached too.** `CompileSourceUncached` is pure and returns a `ZCompiledScript` even when it carries compile errors, so a broken script rezzed 1000 times is parsed and code-generated once.
- **Eviction never hurts a running script.** Removing the dictionary entry does not free the JIT-compiled delegate: each live instance holds the entry directly (`BoundCacheEntry`), so the compiled code lives as long as any instance uses it.
- **Release paths**: script removal (`ZEngine.cs:1740`), a successful recompile on reset or edit (`TryRecompile` returns the old entry, released at `:1823`, `:2152`, `:2186`), and the `zengine run` test harness (`:1135`). Before the P2 fix (2026-07) the cache never evicted and grew for the life of the region.
- **A reset cannot break a working script.** `TryRecompile` swaps the new compiled script in only on success; if a shared include was broken in the meantime, the instance stays on its previous compiled script and the errors are shouted in-world.
- **Dynamic breakpoints change the key**: `TryRecompile` hashes the source plus a `// __zdynbreak:<lines>` suffix, so a script with breakpoints compiles separately from the same script without.
- **Includes are resolved before hashing** (`ZIncludeResolver.cs` header, "CACHE INTERACTION"): two scripts that `#include` the same UUID with the same current content share one entry; editing the included asset changes the resolved text, hence the hash, hence a recompile - stale compiled forms are never served across an asset version bump. `#include` is off by default (`[ZEngine] EnableInclude`) and permission-gated (your own scripts, or a full-perm library).
- **In memory only.** The cache is not written to disk. `ScriptEngines/ScriptCache/<itemID>.state.json` (`m_cacheDir`, `ZEngine.cs:314`) holds *script state* for persistence across restarts, not compiled code; a restart recompiles each distinct source once.
- **No counters exposed.** I found no console command or MCP tool that reports cache size, hits or refcounts; `zengine vars` and `script_engine_status` report per-script state, not the cache.
- **Related**: `ZApiRegistry` builds its API-name lookup tables once (`FrozenDictionary`), and its per-script work is "deduped across identical scripts by the compile cache" (`ZApiRegistry.cs:66`).

Validation recorded in the README (P2): 1000 shared instances plus a concurrent-rez stress test against the real `ZEngineInstance` / `ZApiRegistry` classes.

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
