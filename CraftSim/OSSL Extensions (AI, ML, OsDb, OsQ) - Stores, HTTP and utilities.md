---
tags:
  - craftsim
  - ossl
  - ai
  - ml
  - osdb
  - osq
  - oscache
  - pathfinding
---

# OSSL Extensions (AI, ML, OsDb, OsQ) - Stores, HTTP and utilities

> Part of [[OSSL Extensions (AI, ML, OsDb, OsQ)]]. Sections: OsDb, JsonStore, OsCache, OsQ, HTTP/text/OCR/charts/pathfinding.

## 3. Stores: OsDb, JsonStore, OsCache, OsQ

### OsDb — JSONL row store (`OsDbModule.cs`, `OsDbStore.cs`, `OsDbVectorIndex.cs`, `OsDbConsole.cs`)
In-memory store, notecard-backed, shared across prims by id and **refcounted** by attach/detach (`TotalRefCount`). Each row is a JSON object; a string `"id"` is auto-assigned (random UUID) when absent (`OsDbStore.cs:72`). Mutations persist the notecard via `PersistOsDbStore` (`OSSL_Api.cs:4258`); persistence lives in the module (`PersistToBackingStore`), so the `osdb` console and MCP `osdb_*` tools are equally durable.
- Lifecycle: `key osDbCreate(string nick, string notecardName)` (dedup key is `(creating prim UUID, notecardName)`; a repeat call from the same prim returns the existing id and bumps its reference count, so `os6_demo.lsl` can call it on every use; `nick` is only a label — `OsDbModule.cs:176-190`), `osDbAttach(key)`, `osDbDetach(key)`, `osDbDrop(key)`, `list osDbList()`, `int osDbFlush(key)`, `int osDbRowCount(key)`.
- CRUD: `int osDbInsert(db, jsonRow)`, **`int osDbInsertMany(db, list jsonRows)`** (loop of `Store.Insert` under one lock, then **one** `PersistOsDbStore`; returns rows inserted), `list osDbFind(db, field, op, value)` (list of row JSON strings), `list osDbFindOrderBy(db, field, op, value, sortField, int ascending, int limit)` (`field=""` = whole table, `limit<=0` = unlimited), `int osDbUpdateWhere(db, field, op, value, jsonPatch)`, `int osDbDeleteWhere(db, field, op, value)`.
- **Ops**: `=`/`==`, `!=`/`<>`, `>`, `<`, `>=`, `<=`; both sides numeric → numeric compare, else ordinal string compare (`OsDbStore.cs:~650-690`).
- Indexes: `osDbCreateIndex(db, field)` / `osDbDropIndex` / `list osDbFindByIndex(db, field, value)` (`id` is already primary-indexed and not allowed here).
- Import/export: `int osDbExport(db, notecardName, "csv"|"jsonl")`, `int osDbImport(db, notecardName, format)`.
- **Vector index** (`OsDbVectorIndex`, backed by a VP-tree rebuilt lazily when dirty): `osDbCreateVectorIndex(db, field, metric)` with metric `cosine|cos` (default), `euclidean|l2`, `dot|dotproduct|innerproduct|ip`, **`manhattan|l1`** (unknown → cosine); `osDbDropVectorIndex`; `list osDbFindVectorNearest(db, field, list vector, int topN)` → row JSON strings; **`list osDbFindVectorNearestScores(...)` → strided `[rowJson, score, rowJson, score, …]`** (NOT rows with a `_score` field, `OSSL_Api.cs:4837-4845`); `osDbFindVectorNearestByJson(db, field, vectorJson, topN)`. The vector lives in a JSON array field of the row.
- Async: `osDbFindAsync` / `osDbFindOrderByAsync` (result = rows joined by `"\n"`), `osDbInsertAsync`, `osDbUpdateWhereAsync`, `osDbDeleteWhereAsync` (result = count as string). There is **no** `osDbFindVectorNearestAsync`.
- "~275× faster: 500 rows in 0.027 s vs 7.42 s" is a claim from the old note / README, plausible from the one-persist design, **not re-measured (unverified)**.
- Console: `osdb *` commands (`OsDbConsole.cs`, 820 lines); `[DbConsole]` is a separate SQL console (see CLAUDE.md).

### JsonStore (`osJson*`)
`osJsonCreateStore`, `osJsonDestroyStore`, `osJsonStoreExists`, `osJsonGetValue`, `osJsonSetValue`, `osJsonRemoveValue`, `osJsonGetNodeType`, `osJsonGetValueType`, `osJsonArrayLength`, `osJsonList2Path` (10 functions, all present). JSONL input loads as an array via `ParseJsonOrJsonLines` (CLAUDE.md; plain `OSDParser` would truncate to line 1). Paths use `{field}` / `{a}{b}` syntax (as in `os6_demo.lsl`: `osJsonGetValue(store,"{content}")`).

### OsCache (`OsCacheModule.cs`)
`int osCacheSet(scope,key,value,double ttlSeconds)` (`ttl<=0` = never expires), `string osCacheGet(scope,key)` (empty string if missing/expired), `int osCacheHas`, `int osCacheDelete`, `int osCacheClear(scope)` (scope compare case-insensitive; empty scope clears **everything**), `osCacheGetAsync`, `osCacheSetAsync`. One cache per scene (the module is per region; **not** shared across regions). Expired entries are purged every 30 s; at `MaxEntries` it purges expired then drops the least-recently-accessed 10 %.

### OsQ queues (`OsQModule.cs`, `OsQStore.cs`) — **requires `[OsQ] Enabled = true`**
In-memory only (not persisted). Queues are keyed by **(creating prim UUID, name)** — `osQueueCreate` dedups per prim, so two prims using the same name get different queues unless one passes the id to the other.
- `key osQueueCreate(name)`, `osQueueDelete(q)`, `osQueuePurge(q)`, `list osQueueList()`, `list osQueueStats(q)`.
- `osQueuePublish(q, topic, message)`.
- `key osQueueSubscribe(q, topicFilter)`: filter `""` or `"*"` = all; exact string; or **a trailing `*` prefix match** (`"analytics.*"` matches topics starting `analytics.`). No mid-string wildcards (`OsQStore.cs:~295-315`).
- `list osQueueConsume(q, sub [, double timeout])` → **`[msgId, body, topic]`** (3 elements; empty list when nothing). `timeout>0` **blocks the calling script worker in a 50 ms `Thread.Sleep` loop** until a new message arrives or the deadline passes (`OsQStore.cs:161-190`) — do not use big timeouts on a shared ZEngine worker. `osQueuePeek(q, sub)` same shape.
- `osQueueAck(q, sub, msgId)`. `osQueueSetMaxConcurrency(q, sub, n)` / `GetMaxConcurrency` / `GetInFlightCount` / `GetCursor`: per-subscription in-flight limit (`0` = unlimited); `Consume` returns nothing while in-flight ≥ limit.
- Delivery: per-subscription cursor; a message is removed once all matching subscriptions acked.

---

## 4. HTTP, text, OCR, charts, pathfinding, utilities

- **HTTP** (Moderate): `string osHTTPGet(url [, list headers])`, `osHTTPPost(url, body [, headers])`, `osHTTPRequest(method, url, body, headers)`, `list osHTTPRequestExt(...)` (status, body, content type per the old note — exact element order **unverified**), `osHTTPGetAsync/PostAsync`, `osFetchClean(url [, headers])` / `osFetchCleanAsync` (HTML → plain text, `CleanHtml`). Errors come back as `"ERROR: …"` strings.
- **OCR**: `string osOcr(string source, string prompt [, string engine])`, `osOcrAsync`. `source` = http(s) URL (25 MB cap `MaxOcrFetchBytes`) or the name/UUID of an item in the calling prim's inventory (`OSSL_Api.cs:10141-10146`); `engine` `"llm"` (default, OpenRouter vision) or `"tesseract"` (needs `[OSSL] TesseractDataPath` pointing at tessdata, else an explicit error). PDFs: text layer first (PdfPig, max **200 pages**, `MaxPdfPages`); no text layer → embedded page images OCR'd, max **10 images** (`MaxPdfOcrImages`).
- **Charts**: `string osPlot(chartType, list series, string extraParams)` / `osPlotFace(…, int face)`; types `line|bar|scatter|pie`. **`series` must be numbers** (flat list = one series, x = index), or nested lists of series, each either numbers or `[x,y]` pairs (`PlotSeriesToWireFormat`, `OSSL_Api.cs:1100-1160`); strings are not labels. Labels go in `extraParams`: `width,height,bgcolor,lossless,title,xlabel,ylabel,legend:true|false,labels:A|B|C,colors:RRGGBB|…`. Returns the **texture updater UUID** or `ERROR: …`. Renderer: `ChartRenderModule` (`OpenSim/Region/CoreModules/Scripting/Plot/`, `System.Drawing`; needs `libgdiplus` on Linux).
- **Pathfinding** (High): `osNavMesh(enable)`, `osIsNavMesh()`, `osNavMeshRadius(float)` (additive; enables pathfinding), `osClearNavMesh()`, `osPathfindingHasNavMesh()`, `osPathfindingRebuild()` (all verified present in `IOSSL_Api.cs:156-164`). Tunables in `[Pathfinding]`: `CellSize 0.3`, `CellHeight 0.2`, `AgentRadius 0.4`, `AgentHeight 1.8`, `AgentMaxClimb 0.4`, `AgentMaxSlope 45` (`OpenSimDefaults.ini:2065-2085`).
- **Lex/parse/RAG text**: `list osLex(string input, list rules)`; `list osYacc(string input, list lexRules, string grammarSpec, string startSymbol)` and `osYacc(list tokens, string grammarSpec, string startSymbol)`; `list osChunk(string text, int chunkSize [, int overlap [, string delimiter]])` (default overlap 0, delimiter `"\n"`); `osCleanText`, `osEscapeJson` / `osEscapeJSON` (both exist).
- **Logging**: `osLog(level,msg)`, `osLogTrace/Debug/Info/Warn/Error(msg)`, `osLogSetLevel(level)`; levels `TRACE, DEBUG, INFO, WARN, ERROR, NONE` (`OSSL_Api.cs:5470`). Threat level High.
- **Misc**: `osLoadAppearance`, `osNpcPlayAnimationAdvanced`, `osSetDisplayName`, `osSetAvatarDisplayName`, `osResetRegion`, `osStringSlice`, `osValue2Grayscale`, `osValue2sRGB`, `osListRandomize`, `osRandomColor/Float/Integer` — present in the interface; semantics **not individually verified**. Upstream LSL ports (`llGetRenderMaterial`, `llSetRenderMaterial`, `llIsLinkGLTFMaterial`, `llWorldPosToHUD`, `llMapBeacon`): `LSL_Api*.cs` — **unverified**.

---

---
Back to [[OSSL Extensions (AI, ML, OsDb, OsQ)]].
