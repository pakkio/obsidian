---
tags:
  - craftsim
  - ossl
  - osdb
  - vector-search
  - rag
  - module
aliases:
  - OsDb
  - osDb*
---

# Module - OsDb & Vector Search

← Back to [[CraftSim]] | Related: [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[Module - osAI & osJev]], [[Module - osML & Embeddings]], [[ZEXT Language Extensions]], [[MCP Server & Live Diagnostics]]

**OsDb** gives LSL scripts a thread-safe, **in-memory JSONL row store** persisted to a notecard in a prim's inventory. It offers SQL-like `field op value` filtering, `ORDER BY` / `LIMIT`, equality secondary indexes, and vector similarity indexes (cosine / euclidean / dot / manhattan) for in-world RAG and k-NN.

Source: `OpenSim/Region/OptionalModules/Scripting/OsDb/` (`OsDbModule.cs` 780 lines, `OsDbStore.cs` 685, `OsDbVectorIndex.cs` 657, `OsDbConsole.cs` 820). Interface: `OpenSim/Region/Framework/Interfaces/IOsDbModule.cs`. OSSL glue: `OpenSim/Region/ScriptEngine/Shared/Api/Implementation/OSSL_Api.cs:4247-4915` (region "OsDb Functions"); signatures in `.../Interface/IOSSL_Api.cs:549-576`.

> Verified against code (commit `9fea6a0332`, 2026-10-16).

---

## 1. Architecture

```
LSL script -> OSSL_Api.osDb*  (CheckThreatLevel(High); lock(per-store lock))
           -> IOsDbModule (OsDbModule, one INonSharedRegionModule per region)
                -> RegistryEntry { Lock, OwnerHostId, NotecardName, Nick, Attachments{hostId->count}, Store }
                     -> OsDbStore { List<OSDMap> rows; id->rowIdx dict; field->value->{ids} secondary idx; field->OsDbVectorIndex }
           -> PersistToBackingStore -> notecard on the OWNER prim (jsonl text)
```

- **Row model**: each row is an `OSDMap` (a JSON object). `AppendRow` (`OsDbStore.cs:70`) assigns a random UUID `"id"` when the row has none, an empty string id, or a non-string id. `id` is the primary key with an O(1) dictionary (`m_idIndex`). Ids are **not** de-duplicated: inserting the same id twice creates two rows and the id index points to the later one (documented on `ImportJsonl`, `OsDbStore.cs:~490`).
- **Registry** (`OsDbModule.cs:56-81`): `m_registry` (storeId -> entry) and `m_keyToId` keyed by `(hostId, notecardName)`. `osDbCreate` is idempotent per (prim, notecard): the same id is returned and another reference is added (`CreateStore`, `OsDbModule.cs:176`).
- **Refcounting**: `Attachments` maps prim UUID -> count. When the total reaches 0, `Store` is set to `null` (`DetachStore`, `OsDbModule.cs:267`), i.e. **rows, secondary indexes and vector indexes are discarded from memory**; the registry entry survives so `osDbAttach` can reload from the notecard. Indexes are therefore **not** persisted and **not** rebuilt on reload (call `osDbCreateIndex` / `osDbCreateVectorIndex` again).
- **Locking**: every osDb* call except `osDbCreate` takes `module.GetLockObject(storeId)` for its whole body (`OSSL_Api.cs:4270`), so "mutate then persist" is atomic across prims. `osDbRowCount` takes **no** lock.
- **Persistence** (`OsDbConsole.cs:194` `PersistToBackingStore`): exports all rows as JSONL and rewrites the notecard on the **owner prim** (the prim that first called `osDbCreate`), even if the caller is another prim. Silently no-ops if that prim is gone. Every mutating call (`Insert`, `InsertMany`, `UpdateWhere`, `DeleteWhere`, `Import`, `Flush`, `Detach`, `Drop`) persists; there is no write-behind.
- **Errors**: every OSSL wrapper swallows exceptions (`catch { }`) and returns `0` / empty list / `NULL_KEY`. A script cannot distinguish "unknown store" from "no match" except via the integer return of mutators.

## 2. Configuration

| Key | Where | Default | Notes |
| :--- | :--- | :--- | :--- |
| `[OsDb] Enabled` | `bin/OpenSim.ini:1187`, `bin/OpenSimDefaults.ini:2178` | `False` in Defaults, `True` in the shipped `OpenSim.ini` | `OsDbModule.Initialise` (`OsDbModule.cs:90`); module is silent if the section is missing. No other keys exist. |
| `Allow_osDb* = true` | `bin/config-include/osslEnable.ini:57-84` | one line per function (28 entries) | Threat level of every osDb* function is `High` (`CheckThreatLevel(ThreatLevel.High, ...)`). |

There is no server-side data directory: a store is backed *only* by a notecard in a prim inventory.

## 3. Complete `osDb*` API (28 functions in `IOSSL_Api.cs:549-576`)

| Function | Signature | Behaviour (verified) |
| :--- | :--- | :--- |
| `osDbCreate` | `key osDbCreate(string nick, string notecardName)` (`OSSL_Api.cs:4289`) | Loads the notecard from the calling prim (empty if missing), creates/reattaches store, registers `nick`. Fails (`NULL_KEY`) if `notecardName` is blank or the module is off. |
| `osDbAttach` | `key osDbAttach(key db)` (`:4314`) | Adds a ref for the calling prim. If refcount was 0, reloads the notecard **from the owner prim**. `NULL_KEY` if id unknown/dropped. |
| `osDbDetach` | `void osDbDetach(key db)` (`:4383`) | Persists, then removes one ref. Unloads rows at total refcount 0. |
| `osDbDrop` | `void osDbDrop(key db)` (`:4408`) | Persists, then removes the registry entry regardless of refcount; notecard left intact. |
| `osDbList` | `list osDbList()` (`:4355`) | Flat **stride 4**: `[id, nick, refcount, attachedHostsCsv, ...]`. Includes unloaded (refcount 0) stores. |
| `osDbInsert` | `int osDbInsert(key db, string jsonRow)` (`:4428`) | 1 on success, 0 if JSON invalid or not an object. Persists the whole table every call. |
| `osDbInsertMany` | `int osDbInsertMany(key db, list jsonRows)` (`:4458`) | Returns count inserted; malformed rows are skipped; **one** persist for the batch (only if >0 inserted). |
| `osDbFind` | `list osDbFind(key db, string field, string op, string value)` (`:4485`) | Rows as JSON strings. `field=="id"` with `=`/`==` is an O(1) hit; everything else is a **full scan** (it does *not* use secondary indexes). |
| `osDbFindOrderBy` | `list osDbFindOrderBy(key db, string field, string op, string value, string sortField, int ascending, int limit)` (`:4516`) | Empty `field` = no WHERE (all rows). `limit <= 0` = unlimited. Stable sort; numeric compare if both sides parse as double, else ordinal string compare (`OsDbStore.cs:336-370`). |
| `osDbUpdateWhere` | `int osDbUpdateWhere(key db, string field, string op, string value, string jsonPatch)` (`:4542`) | Merges patch keys into every matching row; returns rows changed (0 if patch invalid). The `id` key in the patch is ignored. Re-indexes rows (secondary + vector). Empty `field` matches nothing. |
| `osDbDeleteWhere` | `int osDbDeleteWhere(key db, string field, string op, string value)` (`:4565`) | Returns rows removed; rebuilds the id index (O(N)). |
| `osDbRowCount` | `int osDbRowCount(key db)` (`:4588`) | No lock. |
| `osDbFlush` | `int osDbFlush(key db)` (`:4605`) | Forces a notecard rewrite; returns 1. |
| `osDbExport` | `int osDbExport(key db, string notecardName, string format)` (`:4632`) | `"csv"` or `"jsonl"`. Writes a **separate** snapshot notecard on the *calling* prim, overwriting a notecard of the same name. CSV header = ordered union of all keys. |
| `osDbImport` | `int osDbImport(key db, string notecardName, string format)` (`:4663`) | Reads a notecard on the *calling* prim and **appends** (never clears). CSV values all become strings; first row is the header. Returns rows appended. |
| `osDbCreateIndex` / `osDbDropIndex` | `int (key db, string field)` (`:4695`, `:4713`) | Equality index; `field=="id"` or empty returns 0. Idempotent. Not persisted. |
| `osDbFindByIndex` | `list osDbFindByIndex(key db, string field, string value)` (`:4734`) | Returns row **ids** (not rows) from the index; empty if no index. Values are compared by their string form (`AsString`). |
| `osDbCreateVectorIndex` | `int osDbCreateVectorIndex(key db, string field, string metric)` (`:4760`) | `field=="id"`/empty returns 0. Indexes every row whose `field` parses as a numeric array. |
| `osDbDropVectorIndex` | `int (key db, string field)` (`:4778`) | |
| `osDbFindVectorNearest` | `list (key db, string field, list vector, int topN)` (`:4796`) | Top-N row JSON strings, best first. |
| `osDbFindVectorNearestScores` | `list (key db, string field, list vector, int topN)` (`:4823`) | **Stride 2**: `[rowJson, score, rowJson, score, ...]` (score is `LSL_Float`). |
| `osDbFindVectorNearestByJson` | `list (key db, string field, string vectorJson, int topN)` (`:4853`) | Query vector as JSON array text. |
| `osDbFindAsync` | `key osDbFindAsync(key db, field, op, value)` (`:4880`) | See section 5. |
| `osDbFindOrderByAsync`, `osDbInsertAsync`, `osDbUpdateWhereAsync`, `osDbDeleteWhereAsync` | (`:4890-4915`) | Same args as sync versions. |

**WHERE operators** (`RowMatches`, `OsDbStore.cs:~640`): `=`, `==`, `!=`, `<>`, `>`, `<`, `>=`, `<=`. If both the field value and the literal parse with `double.TryParse` the comparison is numeric, otherwise `string.CompareOrdinal`. A row lacking the field never matches. Unknown operator never matches. There is no AND/OR, LIKE or nested-field access.

## 4. Vector search (`OsDbVectorIndex.cs`)

- **Metrics** (`enum VectorMetric`: Cosine, Euclidean, DotProduct, Manhattan). `ParseMetric` (`:430`) accepts `cosine|cos`, `euclidean|l2`, `dot|dotproduct|innerproduct|ip`, `manhattan|l1`; blank **or any unknown string silently falls back to cosine**.
- **Score semantics** (`ComputeScore`): cosine = similarity in [-1,1] (higher better; zero-norm vector scores -1); dot = raw dot product (higher better); euclidean / manhattan = **distance** (lower better). Results are sorted best-first by a `BoundedPriorityQueue` of size topN.
- **Vector parsing** (`VectorParser`): row field must be a JSON array of numbers (or numeric strings); stored as `float[]`. Entries with a different dimension than the query are skipped.
- **VP-tree**: built lazily (`RebuildVPTreeIfNeeded`) only when >= 32 entries and the metric is cosine/euclidean/manhattan, but the query path only *uses* it for Euclidean and Manhattan (`SearchNearest`, `:605`); cosine and dot always do a linear scan. Any insert/update/remove marks the tree dirty (rebuilt on the next query).
- **No index = transient scan**: `FindVectorNearestWithScores` (`OsDbStore.cs:180`) builds a temporary cosine index on every call when `osDbCreateVectorIndex` was never called, so queries still work but are O(N) with allocation.
- **Maintenance**: `Insert` / `UpdateWhere` / `DeleteWhere` keep vector indexes in sync (`AddToIndexes` / `RemoveFromIndexes`).
- Embedding generation is *not* part of OsDb; see [[Module - osML & Embeddings]] and [[Module - osAI & osJev]] (`osAIChat`).

## 5. Async variants

`*Async` calls (`OSSL_Api.cs:4880-4915`) go through `StartGatedRequest(..., m_aiRequestGate, MaxConcurrentAIRequests)` (`:6253-6290`):

- They share the **process-wide 8-slot gate used by osAI/osML async calls** (`MaxConcurrentAIRequests = 8`). When saturated the call returns a key string `"ERROR: too many in-flight requests (max 8), try again shortly"` instead of a request key (no queueing).
- Work runs on `Task.Run`; the result is posted to the calling prim as a **`dataserver(key req, string data)`** event. In ZEXT, `osDbFindAsync(...) -> string rows;` and `await { ... }` desugar to this ([[ZEXT Language Extensions]]).
- `osDbFindAsync` / `osDbFindOrderByAsync` deliver the matching rows **joined with `"\n"`** (empty string when none); mutators deliver the integer result as text. Exceptions deliver `"ERROR: <message>"`.
- The sync call inside still takes the per-store lock, so async calls on one store serialize.

## 6. Console and MCP surface

Console (`OsDbConsole.cs:80-190`, group "OsDb"): `osdb list | info <id|nick> | rows <id|nick> [max] [--sort f] [--desc] | find <id|nick> f op v [max] | count | insert (json or @file) | update f op v patch | delete f op v | export <csv|jsonl> [file] | import <csv|jsonl> file | index create|drop <field> | attach | detach | drop`. Console attach uses the pseudo-host `c0450db0-0000-0000-0000-000000000001`. Commands are registered once and resolve stores across all region module instances.

MCP tools (`OpenSim/ApplicationPlugins/McpServer/Tools/DbTools.cs:130-376`): `osdb_list`, `osdb_info`, `osdb_rows`, `osdb_find`, `osdb_count`, `osdb_insert`, `osdb_update`, `osdb_delete`, `osdb_export`, `osdb_import`, `osdb_attach`, `osdb_detach` (mutators call the same `PersistToBackingStore`). See [[MCP Server & Live Diagnostics]].

## 7. Performance

- `osDbInsert` re-serializes and rewrites the entire notecard each call => O(N^2) for N sequential inserts. `osDbInsertMany` persists once. README (`README.md:86-90`) reports 500 rows: 7.42 s (14.8 ms/row) -> 0.027 s (54 us/row), ~275x. (Figure copied from the repo README, not re-measured: unverified.)
- Secondary indexes only accelerate `osDbFindByIndex`; `osDbFind` on an indexed non-id field still scans (the comment at the top of `OsDbStore.cs` says otherwise, but `Find` at `:309` never consults `m_indexes`).

## 8. Real examples (from the repo)

### A. Upsert + parallel reads (`tasks/questgiver.lsl`)
One row per avatar keyed by avatar UUID; `osDbUpdateWhere` returning 0 triggers `osDbInsert` (lines 64-72), index created at line 120, and `osDbFindAsync` + `osDbFindOrderByAsync` issued inside one `await { } : timeout` (line ~134).

```lsl
// tasks/questgiver.lsl (excerpt)
if (osDbUpdateWhere(g_db, "id", "==", (string)av, patch) == 0)
    osDbInsert(g_db, llList2Json(JSON_OBJECT, ["id", (string)av, "name", llKey2Name(av), "stage", stage]));
await {
    osDbFindAsync(g_db, "id", "==", (string)av)               -> string mineRows;
    osDbFindOrderByAsync(g_db, "", "", "", "stage", FALSE, 3) -> string topRows;
} : 10.0 else { return; }
```

### B. Vector RAG (`tasks/mragent_rag.lsl`, `tasks/karpathywiki_rag.lsl`, `tasks/test_osdb_vector.lsl`)
The shipped RAG demos do **not** call a real embedding model: embeddings are hand-written 3-d vectors chosen by keyword (`mragent_rag.lsl:24-26`, `karpathywiki_rag.lsl:64-70`). Pattern:

```lsl
g_dbStore = osDbCreate("mragent_store", "mragent_data");
osDbInsert(g_dbStore, """{"type":"content","id":"c1","text":"...","embedding":[0.90,0.10,0.05]}""");
osDbCreateVectorIndex(g_dbStore, "embedding", "cosine");
list topMatches = osDbFindVectorNearestScores(g_dbStore, "embedding", [0.9, 0.1, 0.0], 2); // [row, score, row, score]
```
`tasks/test_osdb_vector.lsl` is the minimal smoke test of all three nearest-* functions (cosine, query `[1.0, 0.0]`, topN 1) plus `osDbDropVectorIndex`.

### C. Bulk insert, index migration, reattach (`tasks/stress/osdb_demo.lsl`)
Exercises `osDbInsertMany` (500 rows), `osDbCreateIndex(db,"dept")`, `osDbFindByIndex`, moving `e1` from `sales` to `ops` through `osDbUpdateWhere` (index buckets follow), a bulk `osDbUpdateWhere` bonus, `osDbDeleteWhere(... "age","<","22")`, CSV/JSONL export plus additive import into a second DB, `osDbDropIndex`, and `osDbList()`/`osDbAttach` discovery (walk the list with stride 4).

## 9. Limits and known gaps

- Not durable across deletion of the owner prim (it holds the only copy); no transactions; no schema; whole-table rewrite per write (very large tables will be slow; notecard size limits unverified).
- Indexes and vector indexes are lost when refcount hits 0 and are not auto-rebuilt.
- `tasks/karpathywiki_rag.lsl:48,75` calls `osDbCreate("karpathywiki", "")` (blank notecard name is rejected by `CreateStore`) and a 4-argument `osDbCreateVectorIndex(db, "embedding", 3, "cosine")` that does not match the 3-argument API; that demo is stale.
- `.ini` comments still mention `osDbOpen` (`bin/OpenSim.ini:1189`, `OpenSimDefaults.ini:2169`); that function no longer exists.
- Tests: only `OpenSim/Tests/OsDbConsoleTests.cs` (insert/find/persist, update/delete, nick resolution, rows sort/cap, attach reload, list, MCP round trip, unknown store). No unit tests for `OsDbStore` or the vector index; coverage comes from LSL scripts in `tasks/`.

## Corrections vs previous doc

- Metric list was incomplete: a fourth metric **manhattan** (`l1`) exists; unknown metric names silently become cosine.
- Example B in the old note was invented: it used `osMLEmbed("glove_model", ...)`, `osChunk` + `osDbInsertMany` and an `osAIChatAsync` answer step. The real demos use hand-made 3-d vectors and per-row `osDbInsert`, and one is itself stale (section 9).
- `osDbFindVectorNearestScores` returns **stride-2** `[row, score]` pairs; with no vector index it falls back to a transient cosine scan.
- Secondary index was described as giving O(1) queries generally; only `osDbFindByIndex` uses it (returns ids), `osDbFind` does not.
- `osDbDrop` / `osDbFlush` / `osDbExport` semantics clarified: export goes to the *calling* prim, persistence to the *owner* prim; import appends. Async calls share the 8-slot AI gate and return newline-joined rows through `dataserver`.
- Missing facts added: threat level High, per-store lock, indexes lost on unload, `id` auto-assignment, duplicate-id behaviour, console and MCP surface, `[OsDb] Enabled` default `False` in `OpenSimDefaults.ini`.
- Examples A and C are now excerpts tied to the real scripts rather than rewritten code.

## Verified against code (commit 9fea6a0332, 2026-10-16)
Read: `OsDbModule.cs`, `OsDbStore.cs`, `OsDbVectorIndex.cs`, `OsDbConsole.cs` (persistence + command table), OSSL implementations, `osslEnable.ini`, ini files, demo scripts. Not re-measured: the 275x benchmark.
