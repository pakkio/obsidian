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

## Parts

This note was split because it had grown too long. Read the parts in order or jump to the one you need.

- [[OSSL Extensions (AI, ML, OsDb, OsQ) - API surface, AI and ML]]: how the API surface is built, `osAI*`/`osJev*`, `osML*`.
- [[OSSL Extensions (AI, ML, OsDb, OsQ) - Stores, HTTP and utilities]]: OsDb, JsonStore, OsCache, OsQ, HTTP/text/OCR/charts/pathfinding.

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
