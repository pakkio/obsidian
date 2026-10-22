---
tags:
  - craftsim
  - ossl
  - mcp
  - console
  - zext
  - summary
  - reference
aliases:
  - Added functions
  - What CraftSim adds
---

# Added Functions Summary

← Back to [[CraftSim]] | Related: [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[ZEXT Language Extensions]], [[MCP Server & Live Diagnostics]], [[Known and Unresolved Problems]], [[Decision Models Benchmark]]

One page, one line each: everything CraftSim adds on top of upstream OpenSim, grouped by where you use it. Names and signatures come from the repo README tables (`Extensions Reference`), the MCP tool list the live server exposes (76 tools), and the module notes linked in each section. Descriptions marked *(from name)* are read from the signature or neighbouring functions, not from a doc line; open the linked module note for the exact contract.

Counts: the interface declares **~471 `os*` methods (406 names)**; this page lists the extension families, not upstream's.

## 1. Script language (ZEXT, ZEngine)

ZEngine is the CIL script engine (default). ZEXT adds syntax on top of LSL; with `ZEXT = false` each construct below is a compile error. `zengine desugar <file>` shows the plain LSL it lowers to. Details: [[ZEXT Language Extensions]], [[ZEngine]].

| Construct                                                                | What it does                                                                            | Lowered to                                                                    |
| :----------------------------------------------------------------------- | :-------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------- |
| `boolean`, `true` / `false`                                              | boolean type and lowercase literals                                                     | `integer`, `TRUE` = 1                                                         |
| `decimal`                                                                | decimal type                                                                            | see ZEXT note                                                                 |
| `c ? a : b`                                                              | ternary                                                                                 | temporary + `if` / `else`                                                     |
| `a ?? b`                                                                 | null-coalescing (blank string, null key, invalid JSON sentinel)                         | temporary + empty check                                                       |
| `x \|> f(y)`                                                             | pipeline: left side becomes the first argument                                          | `f(x, y)`                                                                     |
| `s[i]`, `list[i]`, `s[a:b]`                                              | indexing and inclusive range slice                                                      | `llGetSubString` / `llList2<Type>`                                            |
| `["k" => v]`                                                             | map literal                                                                             | strided list `["k", v]`                                                       |
| `f"hi {x}"`, `r"C:\temp"`, `"""..."""`, adjacent literals                | f-strings, raw and triple-quoted strings, concatenation                                 | escaped `+` chains                                                            |
| `for`, `while`, `do`, `break`, `continue`, `switch` / `case` / `default` | structured control flow (fall-through kept)                                             | labels and `jump`                                                             |
| `x op= y` for bitwise and shift ops                                      | compound assignment                                                                     | expanded                                                                      |
| `-> var : timeout` async bind, `await { }`, named timers                 | call an `*Async` function and resume with the result                                    | `__zstate` dispatcher with synthesised `dataserver`, `http_response`, `timer` |
| state parameters and arguments                                           | `state name(args)`                                                                      | see ZEXT note                                                                 |
| `#include`, `#define`, `#ifdef`                                          | preprocessor; `#include` needs `[ZEngine] EnableInclude = true` and is permission-gated | text splice before compile                                                    |

Engine features that are not syntax: typed globals and arrays, direct-typed API fast path, compiled-script cache shared by identical scripts ([[Asset Deduplication & Database]] §6), script state persistence, `llSleep` as a cooperative yield, live line tracking and dynamic breakpoints (`set_breakpoint`, below).

## 2. `os*` functions by family

`*Async` variants return a key at once and deliver the result through `dataserver` (bind with `->` in ZEXT). `osAI*` and `osJev*` are threat level High (they spend money): enable each in `osslEnable.ini`.

### 2.1 AI, OpenRouter (`osAI*`) - [[Module - osAI & osJev]]

| Function | What it does |
| :--- | :--- |
| `osAIChat`, `osAIChatAsync` | chat completion; images by adding `"image_url", "<url>"` after a message |
| `osAIChatExt`, `osAIChatExtAsync` | same, returns the full JSON (content, reasoning, model, usage) |
| `osAICompletion`, `osAICompletionAsync` | plain text completion |
| `osAIImage`, `osAIImageAsync` | text-to-image; draws the result on a prim face and returns the texture UUID |
| `osAIListModels` | model ids from the provider's `/models` |
| `osAISetOpenRouterAPIKey` | sets the global OpenRouter key |

### 2.2 Typed decisions (`osJev*`) - [[Module - osAI & osJev]], [[Decision Models Benchmark]]

| Function | What it does |
| :--- | :--- |
| `osJevAsk`, `...Async` | several typed questions in one request; answer JSON |
| `osJevNoul`, `...Async` | yes/no as a probability 0..1 (`-1.0` on error) |
| `osJevChoice`, `osJevChoiceExt`, `...Async` | pick one of the offered ids (Ext: with a description per option) |
| `osJevScore`, `...Async` | position on an ordered scale, 0.0 to 1.0 |
| `osJevAskExt`, `...Async` | like `osJevAsk` with a per-call model and optional images |
| `osJevBox`, `...Async` | bounding box of a described object in an image (Jev 1.14; goes to TypeSafe) |
| `osJevModels` | decision model ids known to this build |
| `osJevSetAPIKey` | deprecated no-op |

### 2.3 Machine learning and embeddings (`osML*`) - [[Module - osML & Embeddings]]

| Function | What it does |
| :--- | :--- |
| `osMLtrain`, `...Async` | train a model (categorize, regression, PCA, cluster, XGBoost) from CSV-style rows |
| `osMLpredict`, `...Async` | predict with a trained model |
| `osMLevaluate`, `...Async` | score a model on held-out labelled rows (accuracy, precision, recall, F1, confusion matrix) |
| `osMLsplit` | split rows into train and evaluation sets |
| `osMLclean`, `...Async` | drop non-finite rows, de-duplicate, optional z-score filtering |
| `osMLlist`, `osMLdrop` | list or remove trained models |
| `osMLLoadEmbeddings`, `...Async` | load a text-embedding model (`word2vec`, `glove`, `onnx`, `openrouter`) |
| `osMLEmbed`, `...Async` | embed text with a loaded model (async delivers a CSV string) |
| `osMLEmbedList`, `osMLEmbedDrop` | list or unload embedding models |

### 2.4 Notecard row stores (`osDb*`) - [[Module - OsDb & Vector Search]]

| Function | What it does |
| :--- | :--- |
| `osDbCreate`, `osDbAttach`, `osDbDetach`, `osDbDrop`, `osDbList` | create a store backed by a notecard in this prim; share it with other prims by id; list the region's stores |
| `osDbInsert`, `osDbInsertMany`, `...Async` | add one row, or many with a single write-back |
| `osDbFind`, `osDbFindOrderBy`, `osDbFindByIndex`, `...Async` | WHERE queries; sorted + limited; O(1) lookup through an index |
| `osDbUpdateWhere`, `osDbDeleteWhere`, `...Async` | merge a patch into, or delete, every matching row |
| `osDbCreateIndex`, `osDbDropIndex` | equality secondary index on a field |
| `osDbCreateVectorIndex`, `osDbDropVectorIndex`, `osDbFindVectorNearest`, `...ByJson`, `...Scores` | vector index and k-nearest-neighbour search (RAG) |
| `osDbRowCount`, `osDbFlush` | row count; force write-back to the notecard |
| `osDbExport`, `osDbImport` | write rows to, or append rows from, a notecard in the calling prim (`csv` or `jsonl`) |

### 2.5 JSON, cache and queues - [[Module - OsQ, OsCache & JsonStore]]

| Function | What it does |
| :--- | :--- |
| `osJsonCreateStore`, `osJsonDestroyStore`, `osJsonStoreExists` | in-memory JSON tree stores |
| `osJsonGetValue`, `osJsonSetValue`, `osJsonRemoveValue` | read, write, delete by path (`"a.b.0"`) |
| `osJsonGetNodeType`, `osJsonGetValueType`, `osJsonArrayLength`, `osJsonList2Path` | inspect a node; build a path from a list |
| `osCacheSet`, `osCacheGet`, `osCacheHas`, `osCacheDelete`, `osCacheClear`, `...Async` | scoped key-value cache with a TTL |
| `osQueueCreate`, `osQueueDelete`, `osQueueList`, `osQueueStats`, `osQueuePurge` | message queues: lifecycle and statistics |
| `osQueuePublish`, `osQueueSubscribe`, `osQueueUnsubscribe` | topic pub/sub |
| `osQueueConsume`, `osQueuePeek`, `osQueueAck` | pull the next message, look at it, acknowledge it |
| `osQueueSetMaxConcurrency`, `osQueueGetMaxConcurrency`, `osQueueGetInFlightCount`, `osQueueGetCursor` | per-subscription backpressure and position |

### 2.6 Web, text and documents - [[Module - osOcr, osPlot, osHTTP, osLex & osYacc]]

| Function | What it does |
| :--- | :--- |
| `osHTTPGet`, `osHTTPPost`, `osHTTPRequest`, `osHTTPRequestExt` (+ `Get`/`Post` `Async`) | HTTP client with headers |
| `osFetchClean`, `osFetchCleanAsync` | GET a page and return plain text without tags, scripts and styles |
| `osOcr`, `osOcrAsync` | text from an image or PDF through an LLM vision model (default) or local Tesseract; scanned PDFs via their embedded images |
| `osPlot`, `osPlotFace` | render line, bar, scatter or pie charts to a dynamic texture (all faces or one) |
| `osLex`, `osYacc` | regex tokenizer and Earley parser from inside a script |
| `osChunk` | split long text into chunks for RAG indexing |
| `osCleanText`, `osEscapeJSON`, `osEscapeJson` | text clean-up; JSON string escaping *(from name)* |
| `osStringSlice` | substring helper *(from name)* |
| `osLog`, `osLogTrace/Debug/Info/Warn/Error`, `osLogSetLevel` | per-prim logging with a minimum level |

### 2.7 World, avatars, navigation, small utilities

| Function | What it does |
| :--- | :--- |
| `osNavMesh`, `osIsNavMesh`, `osNavMeshRadius`, `osClearNavMesh`, `osPathfindingRebuild`, `osPathfindingHasNavMesh` | turn pathfinding on or off, fold nearby geometry into the mesh, clear it, rebuild (rate limited to once per 30 s), ask whether one exists |
| `osLoadAppearance` | load an LLSD appearance notecard onto the script owner |
| `osNpcPlayAnimationAdvanced` | NPC animation with priority (1-10) and duration (1-60 s) |
| `osSetDisplayName`, `osSetAvatarDisplayName` | set own or another avatar's display name |
| `osResetRegion` | factory-reset the region |
| `osRandomFloat`, `osRandomInteger`, `osRandomColor`, `osListRandomize` | random values; shuffle a strided list |
| `osValue2Grayscale`, `osValue2sRGB` | map a number to a grey or a green-yellow-red colour |

### 2.8 Ported from upstream (2026-10-10)

`llGetRenderMaterial`, `llSetRenderMaterial`, `llIsLinkGLTFMaterial`, `llWorldPosToHUD`, `llMapBeacon`.

## 3. Console commands (new)

| Command | What it does |
| :--- | :--- |
| `assets dedup stats [type]`, `assets inspect <id>`, `assets backfill`, `assets sweep` | asset deduplication: report, inspect, migrate legacy rows, reclaim orphan blobs |
| `check_ais`, `use_ais`, `dont_use_ais`, `trace_ais_on`, `trace_ais_off` | AISv3 inventory capability: state, runtime enable / disable for new logins, request tracing |
| `db tables / describe / count / query / exec / info` | SQL console on the grid database (read-only `query`, mutating `exec`) |
| `osdb list / info / count / rows / find / insert / update / delete / attach / detach / drop / export / import` | work on osDb stores from the console |
| `zengine run <file> [--touch] [--say ch msg] [--copies n] [--wait ms]`, `zengine vars`, `zengine include show`, `zengine desugar` | compile and run an LSL file against a throwaway prim; global variables and memory of running scripts; resolved `#include` chain; show lowered LSL |
| `reset script`, `reset object scripts id/name` | reset one script or every script in an object |
| `set script source`, `set notecard content`, `show notecard content`, `set object desc` | edit object inventory and description from the console |
| `pathfinding enable / disable / status / rebuild / rebuild-radius / clear / test-character`, `region reset` | pathfinding control per region; factory reset |
| `experience create / list / show / delete / state / associate / disassociate / grant / revoke` | experiences without the viewer |
| `ssb app / bake / cof / http / test` | exercise the server-side-baking shim without a viewer |
| `jev choose <query>`, `jev status` | pick the best MCP tool with a decision model; key, model and catalog status |
| `log browse`, `log history` | recent log entries by level; recent console commands |
| `show script vars` | same as `zengine vars` |
| `dialog <prim>`, `touch <prim> [bot]` | pCampBot: touch a prim, or send a short chat dialogue |

## 4. MCP tools (76)

Server: [[MCP Server & Live Diagnostics]]. Anything not listed here can be done through `run_console_command`.

| Group | Tools | What they do |
| :--- | :--- | :--- |
| Console and server | `run_console_command`, `list_console_commands`, `console_tail`, `server_version`, `show_server_info`, `show_uptime`, `show_threads`, `show_estates`, `force_gc`, `login_status`, `login_enable`, `login_disable`, `get_craftsim_skill` | run any console command, read the log tail, version / uptime / threads, GC, open or close logins, fetch the CraftSim skill |
| Regions | `list_regions`, `region_status`, `region_broadcast`, `region_restart` | region inventory, health, broadcast a message, restart |
| Avatars, users, NPCs | `list_avatars`, `teleport_avatar`, `list_users`, `get_display_name`, `set_display_name`, `inventory_stats`, `create_npc`, `delete_npc`, `npc_list_with_state`, `npc_move_to` | who is where; move people; display names; inventory statistics *(from name)*; create, remove and drive NPCs |
| Objects | `list_objects`, `get_object_details`, `create_object`, `delete_object`, `get_notecard_content`, `set_notecard_content`, `update_inventory_item_asset` | find and inspect objects, create or delete them, read and write notecards, replace an inventory item's asset *(from name)* |
| Scripts | `script_engine_status`, `reset_script`, `reset_object_scripts`, `zengine_run`, `get_script_globals`, `get_script_locals`, `current_script_line`, `is_script_paused`, `set_breakpoint`, `remove_breakpoint`, `resume_script` | engine state; reset; compile and run an LSL file; live debugging (variables, current line, breakpoints) |
| Database | `db_tables`, `db_describe`, `db_count`, `db_query`, `db_exec`, `db_info`, `db_backup` | SQL against the grid database; consistent MySQL dump |
| osDb | `osdb_list`, `osdb_info`, `osdb_count`, `osdb_rows`, `osdb_find`, `osdb_insert`, `osdb_update`, `osdb_delete`, `osdb_attach`, `osdb_detach`, `osdb_export`, `osdb_import` | the `osdb` console commands as tools |
| Assets | `asset_dedup_stats`, `inspect_asset`, `assets_backfill`, `assets_sweep` | dedup report, inspection, legacy migration, orphan sweep (`confirm=true`) |
| Inventory API | `check_ais`, `use_ais`, `dont_use_ais`, `trace_ais_on`, `trace_ais_off` | AISv3 control |
| Pathfinding | `pathfinding_status`, `pathfinding_rebuild` | navmesh state and rebuild |
| Decisions | `choose_best_tool` | pick the best tool for a query with a decision model (keyword fallback without a key) |

## 5. Modules, addons and services

| Component | What it adds | Enabled by |
| :--- | :--- | :--- |
| ZEngine | CIL script engine with ZEXT | `DefaultScriptEngine = "ZEngine"` |
| Image renderer, chart renderer | `imagedata` texture behind `osAIImage`; `chart` texture behind `osPlot` | loaded by default |
| JsonStore, OsDb, OsCache, OsQ | the `osJson*`, `osDb*`, `osCache*`, `osQueue*` families | `[JsonStore] Enabled`, `[OsDb] Enabled`, `osslEnable.ini` |
| Pathfinding | navmesh and DtCrowd-driven NPC movement | `[Pathfinding] Enabled` (off by default) |
| Experience | experiences and their console | `[Experience]` |
| Database console | `db` commands and `db_*` tools | `[DbConsole]` |
| AISv3 | `InventoryAPIv3` / `LibraryAPIv3` capability; thumbnails and favourites metadata | `Cap_InventoryAPIv3` |
| Server-side baking shim | `UpdateAvatarAppearance` capability | `EnableServerSideBakingShim` |
| Asset deduplication, fallback, diagnostics | content-addressed assets, secondary servers, `/assetdiag` on Robust | [[Asset Deduplication & Database]] |
| MCP server | the 76 tools above, EdDSA-JWT or static-token login, SSH tunnel bridge | `[Mcp] Enabled` |
| WebRTC voice | `os-webrtc-janus` addon and the internal module (wolfvoice backend) | `[WebRtcVoice]` |
| UsersLog, OpenSimSearch | login logging with country lookup; search | `[UsersLog]`, `[Search]` |
| Secrets | `${Secrets\|Name}` in any ini, from `bin/Secrets.ini` | any config that uses it |
| Native libraries | optimised ubODE and BulletSim | `bin/lib64/` |
| Operations scripts | `preflight.sh`, `backup-db.sh`, `restore-db.sh` | `bin/` |

Problems and unverified claims for everything above: [[Known and Unresolved Problems]].
