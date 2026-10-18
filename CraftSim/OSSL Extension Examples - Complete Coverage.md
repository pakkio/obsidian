---
tags:
  - craftsim
  - ossl
  - lsl
  - examples
  - osdb
  - osjev
---

# OSSL Extension Examples - Complete Coverage

← Back to [[CraftSim]] | Index: [[OSSL Extensions (AI, ML, OsDb, OsQ)]] | See also: [[End-to-End Multi-Module LSL Architectures]], [[Added Functions Summary]]

Runnable LSL examples for the extension `os*` functions. **Every script on this page was run on the live simulator on 2026-10-17** with the MCP tool `zengine_run` (console: `zengine run <file> --wait <ms>`), and the output shown is what it printed. The files are in the repo under `tasks/jev/`.

> [!NOTE] What this page covers
> Sections 1-2: **osDb** (2 scripts) and **osJev** (8 scripts: one per function family, per model, SubQ, async bind, Box with an image). The other families have examples in their module pages ([[Module - OsQ, OsCache & JsonStore]], [[Module - osML & Embeddings]], [[Module - osOcr, osPlot, osHTTP, osLex & osYacc]], [[Module - Pathfinding, Experience, Materials & Avatar Utilities]], [[Module - osAI & osJev]]) that were **not re-run here**; see section 3.

> [!WARNING] This page replaces a truncated one
> The previous version ended in the middle of a statement at exactly 4096 bytes, so it covered only osDb. It also said the scripts were "not compiled or run", and the osDb scripts called `osDbCreate("players")` with one argument; the real signature takes **two** (`nick`, `notecardName`), so they would not have compiled. Both osDb scripts below are corrected and were executed.

Functions above VeryLow need `Allow_<fn> = true` in `osslEnable.ini`. `osJev*` also need a key: OpenRouter for most models (`[OSSL] OpenRouterApiKey` / `OPENROUTER_API_KEY`), `SubqApiKey` for SubQ, `TypeSafeApiKey` for `osJevBox`. Without a key a call returns an `ERROR:` string and `osJevNoul` / `osJevScore` return `-1.0`.

---

## Parts

This note was split because it had grown too long. Read the parts in order or jump to the one you need.

- [[OSSL Extension Examples - Complete Coverage - osDb examples]]: lifecycle, queries, import/export, backup and restore.
- [[OSSL Extension Examples - Complete Coverage - osJev examples]]: typed decisions: Noul, Choice, Score, Ask, models, SubQ, async, Box.

## 3. Families with examples elsewhere (not re-run in this pass)

| Family | Where the examples are | Status |
| :--- | :--- | :--- |
| `osCache*`, `osQueue*`, `osJson*` | [[Module - OsQ, OsCache & JsonStore]] | not re-run |
| `osML*`, embeddings | [[Module - osML & Embeddings]] | not re-run |
| `osHTTP*`, `osFetchClean*`, `osOcr`, `osPlot`, `osLex`, `osYacc` | [[Module - osOcr, osPlot, osHTTP, osLex & osYacc]] | not re-run |
| `osNavMesh*`, pathfinding, NPC helpers | [[Module - Pathfinding, Experience, Materials & Avatar Utilities]] | not re-run |
| `osAI*` | [[Module - osAI & osJev]] | not re-run |
| `osDbFindVectorNearest*`, `osDbInsertMany`, async osDb | [[Module - OsDb & Vector Search]]; `tasks/karpathywiki_rag.lsl` is stale (blank notecard name, 4-argument `osDbCreateVectorIndex`) | not re-run |

To re-run any of them: copy the file to the server and `zengine_run` it, as above. Open problems are in [[Known and Unresolved Problems]].
