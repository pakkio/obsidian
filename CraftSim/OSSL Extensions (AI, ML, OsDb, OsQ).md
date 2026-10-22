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

# OSSL Extensions (AI, ML, OsDb, OsQ)

← Back to [[CraftSim]] | Related: [[ZEngine]], [[ZEXT Language Extensions]], [[MCP Server & Live Diagnostics]], [[Build, Configuration & Self-Hosting]]

> [!NOTE] Verified against code (commit `9fea6a0332`, 2026-10-16)
> Checked against `OpenSim/Region/ScriptEngine/Shared/Api/Interface/IOSSL_Api.cs`, `.../Implementation/OSSL_Api.cs` (~11 600 lines), `OpenSim/Region/OptionalModules/Scripting/{OsDb,OsQ,OsCache,JsonStore}`, `OpenSim/Region/OptionalModules/World/Pathfinding`, `OpenSim/Framework/{DecisionClient,ImageGenClient}.cs`, and `bin/OpenSim.ini`, `bin/OpenSimDefaults.ini`, `bin/config-include/osslEnable.ini`. Unverified items are marked.

> [!TIP] Dedicated Module Deep-Dives & Non-Trivial Code Examples
> - **[[Module - OsDb & Vector Search]]** — `osDb*` SQL/JSONL row stores, indexes, `osDbInsertMany`, and k-NN vector RAG examples.
> - **[[Module - osAI & osJev]]** — `osAI*` OpenRouter LLMs, `osAIImage` (`ImageDataRenderModule`), and `osJev*` typed decisions (several decision models: Jev, Microsoft, GPT-6 Luna, Perplexity pplx-decider, SubQ; benchmark in [[Decision Models Benchmark]]).
> - **[[Module - osML & Embeddings]]** — `osML*` (`pca`, `cluster`, `categorize`, `regression`, `xgboost`), `osMLclean`/`split`/`evaluate`, and `word2vec`/`GloVe`/`ONNX`/`OpenRouter` embeddings.
> - **[[Module - OsQ, OsCache & JsonStore]]** — `osQueue*` pub/sub with backpressure, `osCache*` scoped TTL cache, and `osJson*` JSONL tree stores.
> - **[[Module - osOcr, osPlot, osHTTP, osLex & osYacc]]** — `osOcr` (Vision/Tesseract/PdfPig), `osPlotFace` (`ChartRenderModule`), `osHTTP*`/`osFetchClean*`, `osLex`/`osYacc`, and `osLog*`.
> - **[[Module - Pathfinding, Experience, Materials & Avatar Utilities]]** — DotRecast NavMesh (`osPathfinding*`, `llCreateCharacter`), `[Experience]` KVP, GLTF `llSetRenderMaterial`, and `llWorldPosToHUD`.
> - **[[End-to-End Multi-Module LSL Architectures]]** — the 6-module NPC (`os6_demo.lsl`) and a Web-to-Vector-to-Chart RAG blueprint.
> - **[[OSSL Extension Examples - Complete Coverage]]** — a non-trivial LSL example for every extension `os*` function not shown elsewhere (osDb, osCache, osQueue, osJson, osAI/osJev, osML, osHTTP, osLex, osOcr, osPlot, NavMesh).

---

## 0. How the API surface is built

- Declarations: `IOSSL_Api.cs` (1188 lines). A crude regex over the interface counts **~471 `os*` declarations (406 distinct names, overloads included)**; the "383" in CLAUDE.md and the previous note is stale. Implementations: `OSSL_Api.cs` (region sections: `osDb` ~4258-4930, `osCache` ~4920-5000, `osQueue` ~5040-5220, `osLog` ~5400-5530, AI ~5550-6100, `osJev` ~6340-6580, `osML` ~6586-8760, embeddings ~8760-9540, HTTP ~9720-9810, OCR ~9857-10230, `osLex/osYacc` ~11400-11560).
- Both engines bind the same API: ZEngine through `ZApiRegistry` (reflection over `OSSL_Api`), YEngine through `TokenDeclInline.AddInterfaceMethods` (claim from CLAUDE.md, YEngine side **unverified** by me).
- **Permission model**: every function begins with `CheckThreatLevel(level, name)`. Levels seen: `osAI*`, `osJev*`, `osML*`, `osDb*`, `osCache*`, `osQueue*`, `osLog*`, `osOcr`, `osNavMesh` = **High**; `osHTTP*`/`osFetchClean*` = **Moderate**; `osLex` = Low; `osPlot`/`osJson*` = VeryLow; `osChunk` = None. `bin/config-include/osslDefaultEnable.ini` sets `OSFunctionThreatLevel = VeryLow`, so everything above VeryLow needs an explicit `Allow_<fn> = true` in `bin/config-include/osslEnable.ini` (389 `Allow_os*` lines at this commit; includes `osQueue*`, `osCache*`, `osDb*`, `osJson*`, `osOcr`, `osPlot`, `osJev*`, `osAI*`). A function missing there fails with an OSSL permission error — forgetting `Allow_*` is the usual reason a new `os*` call "does nothing".
- **Async family**: `*Async` variants return a `key` and deliver the result as a `dataserver(key, string)` event (usable with ZEXT `->`, see [[ZEXT Language Extensions]]). All go through `StartGatedRequest(fn, work, gate, max)` with the **process-wide 8-slot `m_aiRequestGate`** (`OSSL_Api.cs:6253`, `MaxConcurrentAIRequests = 8`) — this includes `osDb*Async`, `osCache*Async`, `osHTTP*Async`, `osFetchCleanAsync`, `osML*Async`. Past the cap new requests are rejected rather than queued. `osMLLoadEmbeddingsAsync` uses the separate 2-slot `m_embedLoadGate` (`:6264`).

### Which modules must be switched on (important)
| Module | Switch | Shipped default |
| :--- | :--- | :--- |
| OsDb | `[OsDb] Enabled` (module default false; module absent → disabled) | **on** in `bin/OpenSim.ini:1187` |
| JsonStore | `[JsonStore] Enabled` (code default false) | **on** in `bin/OpenSim.ini:1178`; `MaxStringSpace = 65536` (UTF-8 bytes/store, 0 = unlimited), `MaxStoreCount = 1024` |
| OsCache | `[OsCache] Enabled` (code default **true**), `MaxEntries` (default 10 000) | no section needed; enabled |
| **OsQ** | `[OsQ] Enabled` (module default false; section missing → `return`) | **no `[OsQ]` section exists in any shipped ini at this commit**, so `osQueueCreate` returns an empty key (`OSSL_Api.cs:5047-5049`) and every `osQueue*` call is a silent no-op until you add `[OsQ]` / `Enabled = true`. (README.md:1225 only lists the `osslEnable.ini` rows.) |
| Pathfinding | `[Pathfinding] Enabled` — **code default false** (`PathfindingModule.cs:~120`); the `;;Enabled = true` "default: true" comment in `bin/OpenSimDefaults.ini:2067` is stale | off; `osNavMesh(TRUE)` / `osNavMeshRadius` / console enable it at runtime |

---

## 1. AI (`osAI*`) and typed decisions (`osJev*`)

### OpenRouter chat/completion/image
Key: `osAISetOpenRouterAPIKey(key)` or env `OPENROUTER_API_KEY` (`DecisionClient.ResolveApiKey`). Base URL `[OSSL] AIBaseURL` (default `https://openrouter.ai/api/v1`, `OSSL_Api.cs:5559`). No key → functions return `"ERROR: …"` strings before any HTTP request; they do not throw.

| Signature | Notes |
| :--- | :--- |
| `string osAIChat(string model, list messages [, int maxTokens [, list options]])` | `messages` is a **strided** list `["system","…","user","…"]`; an image is added after the content: `["user","Describe","image_url","https://…"]` (multiple `image_url` allowed, `OSSL_Api.cs:5677-5728`) |
| `string osAIChatExt(model, messages, maxTokens, options)` | full JSON response |
| `string osAICompletion(model, prompt [, maxTokens])` | |
| `list osAIListModels()` | queries `/models` |
| `string osAIImage(string prompt, string model, int face, string extraParams)` | `model=""` → `[OSSL] ImageModel` (default `recraft/recraft-v4.1-flash`, `ImageGenClient.cs:56`); `face -1` = all sides. Returns the **texture updater UUID (not the texture asset UUID)** or `"ERROR: …"` (`OSSL_Api.cs:5598-5610`). Image models need `/chat/completions` with `modalities:["image"]` or `["image","text"]`; dedicated models that 404 are re-sent to `POST /images` and remembered (`ImageGenClient.cs:42-65,152-224`). Rendering: `ImageDataRenderModule` (content type `"imagedata"`; **System.Drawing** for drawing, **SkiaSharp** only to decode the returned image, `ImageDataRenderModule.cs:269`). Caption grammar in `extraParams`: `width,height,lossless,text,textpos,textsize,textcolor,textbg,settext,settextcolor,settextalpha`. Decoded image cap 20 MB (CLAUDE.md, **unverified**) |
| `key osAIChatAsync / osAIChatExtAsync / osAICompletionAsync / osAIImageAsync` | same args |

`options` list keys (`OSSL_Api.cs:5843-5890`): `temperature`, `top_p`, `frequency_penalty`, `presence_penalty`, `seed`, `stop`, `reasoning_effort`, e.g. `["temperature",0.7,"seed",42]`.

### Typed decisions (`osJev*`)
Backed by `OpenSim.Framework.DecisionClient`, **not** TypeSafe any more: default model **`perplexity/pplx-decider-v1.2`** (`DecisionClient.DefaultModel`; it was `pplx-decider-v1.1-27b`, retired by OpenRouter, and `perplexity/sonar` before 2026-10-16), base `https://openrouter.ai/api/v1`, config `[OSSL] DecisionBaseURL` / `DecisionModel`, same OpenRouter key as `osAI*`. Model comparison: [[Decision Models Benchmark]]. The reply is validated (a choice not offered is an error; numbers clamped to 0..1).
- `string osJevAsk(string state, string questionsJson)`
- `float osJevNoul(string state, string instructions)` → probability 0..1, **`-1.0` on any error / missing key** (`OSSL_Api.cs:~6407`)
- `string osJevChoice(state, instructions, list options)`, `osJevChoiceExt(state, instructions, list optionsWithDescriptions)`
- `float osJevScore(state, instructions, list levels)` → 0.0 (lowest level) … 1.0 (highest)
- **Decision models, Box and images (merged, live since 2026-10-16/17)**: `osJevAskExt(state, questionsJson, model, assetsJson)` / `osJevAskExtAsync` pick the model per call; ids of real decision models (Jev, `microsoft/microsoft-decision-1`, `openai/gpt-6-luna-decisions`, `perplexity/pplx-decider-*`, ...) are sent to OpenRouter's `/api/alpha/decisions`, other ids still use chat. `osJevBox(state, target, assetsJson)` (Jev 1.14) returns `[top, bottom, left, right, confidence, present]`; `osJevModels()` lists known decision models. `[OSSL]` keys: `DecisionProtocol`, `DecisionsURL`, `DecisionBoxModel`. **Images and Box go straight to TypeSafe** (`TypeSafeAPIKey`, `TypeSafeURL`) because OpenRouter's decisions endpoint never shows the image to the model. **`subq-snap-preview`** (SubQ, text only) goes to `api.subq.ai/v1/systemone` with `SubqAPIKey` / `SUBQ_API_KEY` / `SubqURL`. `pplx-decider-v1.2` can answer `HTTP 429` (retry after 1 s); the client does not retry yet. Details: [[Module - osAI & osJev]] section 6.
- `*Async` for all five; `osJevSetAPIKey` is a deprecated no-op. Note: `osJevChoiceExtAsync` etc. all exist in the interface.

---

## 2. Machine learning and embeddings (`osML*`)

### Models (`m_mlModels`, static region-wide `ConcurrentDictionary`, `OSSL_Api.cs:6586`)
`string osMLtrain(string name, string modelType, list data [, int k [, int clean [, int epochs]]])` — returns `OK…`/`ERROR: …`. **Valid `modelType`: `pca`, `cluster`, `autocatalogue`, `categorize`, `regression`, `xgboost`** (`:8097-8100`).
- `data` is a list of rows (nested lists of numbers; a flat list = one row). For `categorize`/`regression`/`xgboost` the **last column is the label/target**, string feature columns are one-hot encoded (also at predict time).
- `clean` bit-mask: `1` drop non-finite rows, `2` drop duplicate rows, `4` z-score numeric columns.
- `epochs` (0 = default): `xgboost` boosting rounds (default 24, falls back to `k` if >0), `cluster`/`autocatalogue` max Lloyd iterations (default 100), ignored for `pca`/`categorize`/`regression`.
- `cluster` = K-means (`k<=0` auto via elbow heuristic for `autocatalogue`); the seed is derived with FNV-1a over the model name (`:6667`). `categorize` = k-NN over prototypes (majority vote), `regression` = closed-form ridge, `xgboost` = shallow-tree gradient boosting.
- `list osMLpredict(name, row)`, `list osMLevaluate(name, testData)` (categorize result starts `["categorize", n, accuracy, macroPrecision, …]`; regression/xgboost have their own evaluator), `list osMLsplit(data, trainFrac [, evalFrac [, seed]])`, `list osMLclean(data [, mode])` (default mode **3** = drop non-finite + duplicates; add 4 for z-score), `list osMLlist()`, `int osMLdrop(name)`.
- Async: `osMLtrainAsync`, `osMLpredictAsync`, `osMLevaluateAsync`, `osMLcleanAsync` (no async for `osMLsplit`). Training/evaluation use `ThreatLevel.High`.

### Embeddings
`string osMLLoadEmbeddings(name, method, source [, vocabSource])`; **methods**: `word2vec` (binary or text, auto-detected), `glove` (text, no header), `onnx` (generic `input_ids` int64 → float output; contract checked by `ValidateOnnxContract`, `OSSL_Api.cs:9170`), `openrouter` (`source` = model id like `openai/text-embedding-3-small`; nothing is downloaded, calls `<AIBaseURL>/embeddings`). Returns `"OK: …"` or `"ERROR: …"`.
- `list osMLEmbed(name, text)` → list of floats, or a single `ERROR…` string; `osMLEmbedAsync` returns the vector as a **comma-separated string** (`:9521-9534`). `osMLEmbedList()` → `[name, method, dim, …]`, `osMLEmbedDrop(name)` → 1/0.
- Limits (`[OSSL]` section, read at `OSSL_Api.cs:257-260`): `EmbedMaxFetchBytes` (2 GB default, stream-to-disk via `FetchToDiskFromUrl`), `EmbedMaxMemoryBytes` (2 GB default; file size is the estimate; checked before parsing, `0` disables), `EmbedCacheDirectory` (`osMLEmbedCache`), `EmbedKeepCacheFiles` (false). Loads gated by the 2-slot `m_embedLoadGate`.
- A model that "runs" can still produce meaningless output if its real contract doesn't match the assumed ONNX convention — not detectable at runtime.

---

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

## Corrections vs previous doc

- **"383 total `IOSSL_Api` methods" / "over 100 new"** — stale; now ~471 declarations (406 names). "Over 100 new" cannot be checked without diffing against upstream (**unverified**).
- **OsQ is not enabled by default**: no `[OsQ]` section exists in `bin/OpenSim.ini`, `bin/OpenSimDefaults.ini` or `config-include`; without it all `osQueue*` calls silently return empty/no-ops.
- `osQueueConsume` returns **three** items `[msgId, body, topic]`; wildcard filter is a **trailing-`*` prefix** match only; queues are scoped to the creating prim; `timeout` busy-waits with 50 ms sleeps.
- `osDbFindVectorNearestScores` returns a **strided `[rowJson, score, …]` list**, not rows with `_score`. Vector metrics also include `manhattan`/`l1`. Index is a **VP-tree**.
- `osMLtrain` model types: the old text said "pca, cluster, autocatalogue + supervised"; the actual set is `pca, cluster, autocatalogue, categorize, regression, xgboost`. `osMLclean` default mode is 3.
- `osAIImage` returns the **texture updater UUID**, not the texture asset UUID; drawing is `System.Drawing`, SkiaSharp is only used to decode.
- `osPlot` series cannot contain string labels (`["c0", 0.8, …]` is wrong); use `labels:` in `extraParams`.
- `osMLEmbed` returns a list; only the **async** form delivers a CSV string. OpenRouter embeddings use `<AIBaseURL>/embeddings`, not a hard-coded `/api/v1/embeddings`.
- Pathfinding: defaults **off** in code (a stale comment in `OpenSimDefaults.ini` says true); `osIsNavMesh` and `osClearNavMesh` do exist.
- JsonStore: module code default is *disabled*; the shipped ini enables it. `OsCache` is enabled by default with `MaxEntries = 10000` (undocumented before).
- `osJevNoul` returns **-1.0** on error/missing key; `osJevScore` scale 0.0–1.0.
- `osDbFindVectorNearestAsync` appears in the ZEXT note's example but does not exist.
