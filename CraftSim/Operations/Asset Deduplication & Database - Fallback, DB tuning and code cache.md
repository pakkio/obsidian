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

# Asset Deduplication & Database - Fallback, DB tuning and code cache

> Part of [[Asset Deduplication & Database]]. Sections: secondary asset server, MySQL/PostgreSQL write path, compiled-script cache.

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

---
Back to [[Asset Deduplication & Database]].
