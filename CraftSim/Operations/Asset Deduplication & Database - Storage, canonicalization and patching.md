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

# Asset Deduplication & Database - Storage, canonicalization and patching

> Part of [[Asset Deduplication & Database]]. Sections: content-addressed storage, object canonicalization, in-place `#include` patching.

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

---
Back to [[Asset Deduplication & Database]].
