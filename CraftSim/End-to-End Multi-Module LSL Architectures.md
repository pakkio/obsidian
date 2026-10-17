---
tags:
  - craftsim
  - lsl
  - cookbook
  - rag
  - multi-agent
  - zext
aliases:
  - CraftSim End-to-End Examples
---

# End-to-End Multi-Module LSL Architectures

← Back to [[CraftSim]] | Related: [[Module - OsDb & Vector Search]], [[Module - osAI & osJev]], [[Module - osML & Embeddings]], [[Module - OsQ, OsCache & JsonStore]], [[Module - osOcr, osPlot, osHTTP, osLex & osYacc]], [[Module - Pathfinding, Experience, Materials & Avatar Utilities]]

This note shows how several CraftSim modules combine in one system. Facts about each API are in [[OSSL Extensions (AI, ML, OsDb, OsQ)]]; syntax is in [[ZEXT Language Extensions]]; the engine is [[ZEngine]].

> [!NOTE] Verified against code (commit `9fea6a0332`, 2026-10-16)
> The previous version of this page presented two long listings as "drawn from `tasks/os6_demo.lsl`, `tasks/mragent_rag.lsl`, and `scripts/brain.lsl`". **They were rewritten illustrations, not code from those files**, and contained calls that do not work against the real API. This page now (1) describes what the repo scripts actually do, with `file:line` references, and (2) gives corrected, API-checked adaptations, clearly labelled as such. Adaptations were checked by reading signatures/semantics; **I did not run them in-world (unverified at runtime)**.

## Prerequisites for any of these scripts
- `Allow_<function> = true` for every `os*` call in `bin/config-include/osslEnable.ini` (threat level High for osAI/osML/osDb/osCache/osQueue/osLog/osOcr; see the permission note in [[OSSL Extensions (AI, ML, OsDb, OsQ)]]).
- `[OsDb] Enabled = True` and `[JsonStore] Enabled = True` (both on in the shipped `bin/OpenSim.ini`). **`[OsQ] Enabled = true` must be added: no shipped ini contains an `[OsQ]` section**, so without it `osQueueCreate` returns an empty key and every queue call is a silent no-op.
- An OpenRouter key: `osAISetOpenRouterAPIKey("…")` in the script or `OPENROUTER_API_KEY` in the server environment. `tasks/os6_demo.lsl` and `tasks/mragent_rag.lsl` keep it in a `KEY` string constant (empty by default).

---

## 1. The 6-module Tavern Keeper NPC — what `tasks/os6_demo.lsl` really is

499 lines; one script, **synchronous** style (no `->`, no `await`). Modules and where they are used:

| Module | Real usage | Lines |
| :--- | :--- | :--- |
| osAI | `osAIChat("openai/gpt-4o-mini", g_history, 512, ["temperature", 0.85])` for greeting, `0.8` for replies; blocking call; `ERROR:` prefix checked | 159, 208 |
| osDb | `osDbCreate("memory", NPC_NAME + "_memory")` (called again in every helper — idempotent per prim+notecard), `osDbFind(db,"player","=",name)` for history, `osDbInsert` in `logToDb`, `osDbDeleteWhere(db,"type","=","chat")` for `/reset`, `osDbRowCount` for `/stats` | 63, 126, 342-344, 456, 469 |
| osQ | one queue `tavern-events`; `g_subLog = osQueueSubscribe(q, "")` (all) and `g_subAnalytics = osQueueSubscribe(q, "analytics.*")`; `osQueueSetMaxConcurrency` 10 / 5; publishes to `analytics.conversation` and `chat.event`; `drainQueues()` loops `osQueueConsume`/`osQueueAck` | 74-84, 139/182/227/240, 376-410 |
| osHTTP | `osHTTPGet("https://httpbin.org/get", …)` connectivity check; `fetchLiveContext()` calls `http://worldtimeapi.org/api/ip.txt` and `https://api.quotable.io/random?maxLength=100` | 89, 265, 283 |
| osJson | `osJsonCreateStore(quoteResp)` + `osJsonGetValue(store,"{content}")`; analytics events parsed with `{event}`, `{player}`, `{npc}`; always `osJsonDestroyStore` | 287, 398 |
| osLog | `osLogSetLevel("DEBUG")`, `osLogInfo/Debug/Warn/Error/Trace` throughout | 48+ |

Control flow: `state_entry` → `setup()` (key, osDb, queue, subscriptions, HTTP probe, `llSetTimerEvent(5.0)`); `touch_start` toggles a conversation (requires `KEY != ""`) → `startChat` (history count from osDb, publish `analytics.conversation start`, live context, system prompt, greeting via `osAIChat`, `llListen(0, "", g_toucher, "")`, `llSetTimerEvent(TIMEOUT=120)`); each owner/player chat line → `continueChat` (trim history to `MAX_HIST=6` turn pairs, `osAIChat`, `parseAIResponse` strips one `[anim=…]`/`[face=…]` tag — the actual `osNpcPlayAnimation` call is only a comment, line 327); `timer()` drains both subscriptions then, if still conversing, times out the chat and re-arms 5 s. Owner commands: `/reset`, `/stats` (`osQueueStats` → messages, subs, topics). It uses f-strings and triple-quoted `f"""…"""` JSON (ZEXT), `jump`/labels for the drain loop, and stock `osEscapeJson`.

### API facts the real script relies on (and the old listing got wrong)
- `osQueueConsume(q, sub)` returns **`[msgId, body, topic]`** (`llList2String(ev, 0/1/2)`), empty list when none; `osQueueAck` takes the key from element 0.
- Topic filter `"analytics.*"` is a **prefix** match; `""` matches everything.
- `osDbFind` returns a **list of row JSON strings**; its `…Async` form returns the rows joined with `"\n"`.
- `osHTTPGet` returns the body or an `ERROR:`-prefixed string; the script tests `llGetSubString(resp,0,5) == "ERROR:"`.

### Corrected adaptation: ZEXT version with parallel context fetch
Not in the repo; shows the same system with `await {}` and a named timer. Fixes versus the previous listing: the **named timer is declared in `state_entry`, not inside a helper function** (`RejectInlineTimer` rejects inline timers inside user functions, `ZLSLParser.cs:58`); the consume result is `[id, body, topic]`; `osDbFindAsync` yields `"\n"`-joined rows.

```lsl
string NPC_NAME   = "Marcus";
key    g_db;
key    g_queueId;
key    g_subAnalytics;
list   g_history;

greetPlayer(key av, string playerName)          // user function with await: inlined by ZFunctionInliner
{
    await {
        osDbFindAsync(g_db, "player", "=", playerName) -> string pastRows;
        osHTTPGetAsync("https://api.quotable.io/random?maxLength=80", ["Accept", "application/json"]) -> string quoteJson;
    } : 8.0 else {
        osLogWarn("Context fetch timed out; continuing with defaults.");
    }
    integer pastCount = 0;
    if (pastRows != "") pastCount = llGetListLength(llParseString2List(pastRows, ["\n"], []));

    string quote = "";
    key qStore = osJsonCreateStore(quoteJson);
    if (qStore != NULL_KEY) { quote = osJsonGetValue(qStore, "{content}"); osJsonDestroyStore(qStore); }

    osQueuePublish(g_queueId, "analytics.conversation",
        f"""{{"event":"start","player":"{osEscapeJson(playerName)}","past_msgs":{pastCount}}}""");

    g_history = ["system",
        f"You are {NPC_NAME}, a tavern keeper. Visitor {playerName} has {pastCount} past messages. Quote: '{quote}'. Keep replies under 350 chars.",
        "user", f"Greet {playerName}."];
    osAIChatAsync("openai/gpt-4o-mini", g_history, 300, ["temperature", 0.8]) -> string reply;
    llSay(0, f"{NPC_NAME}: {reply}");
    osDbInsert(g_db, f"""{{"type":"chat","ts":{llGetUnixTime()},"player":"{osEscapeJson(playerName)}","text":"{osEscapeJson(reply)}"}}""");
}

default
{
    state_entry()
    {
        osLogSetLevel("DEBUG");
        g_db           = osDbCreate("memory", NPC_NAME + "_memory");
        g_queueId      = osQueueCreate("tavern-events");
        g_subAnalytics = osQueueSubscribe(g_queueId, "analytics.*");
        osQueueSetMaxConcurrency(g_queueId, g_subAnalytics, 5);

        timer "analytics_drain" -> 5.0: {
            while (TRUE)
            {
                list ev = osQueueConsume(g_queueId, g_subAnalytics);   // [msgId, body, topic]
                if (ev == []) return;
                key store = osJsonCreateStore(llList2String(ev, 1));
                if (store != NULL_KEY)
                {
                    string evType = osJsonGetValue(store, "{event}");
                    osLogInfo(f"[Analytics] {llList2String(ev, 2)} event={evType}");
                    osJsonDestroyStore(store);
                }
                osQueueAck(g_queueId, g_subAnalytics, (key)llList2String(ev, 0));
            }
        }
    }
    touch_start(integer n) { greetPlayer(llDetectedKey(0), llDetectedName(0)); }
}
```
Notes: `->` inside user functions is supported (inlined; tests `ZAsyncBindTest.TestAsyncBindInsideUserFunction*`, `TestAsyncFanOutInsideUserFunctionCompiles`) — the comment in `tasks/craftsim_analyst_demo.lsl` ("may only live directly in an event handler body") is **stale**. Recursive suspending functions are still rejected. Each `*Async` call occupies one of the 8 shared async slots while in flight.

---

## 2. Vector RAG + chart — what the repo scripts really do

There is no file in the repo that does fetch → chunk → embed → index → plot in one script. The real pieces:

- **`tasks/craftsim_analyst_demo.lsl`** (148 lines): `osDbCreate` + `osDbInsertMany` of three facts with 3-D hand-written `embedding` arrays, `osDbCreateVectorIndex(g_db,"embedding","cosine")`, `osFetchClean(url, headers)` (falls back to a canned sentence on `ERROR:`), `osStringSlice(cleanText,0,300)`, `osDbFindVectorNearestScores(g_db,"embedding",[0.90,0.10,0.05],2)` read as **strided `[rowJson, score, …]`** (loop `i += 2`), `osAIChatAsync(…) -> string finalReport` in `touch_start`, `osQueuePublish(g_queue,"analytics.reports.daily",…)`, and `osPlotFace("bar", [496.0,380.0,4.0,55.0], "width:512,height:512,title:…,labels:Events|Cache|Nodes|FPS,colors:00FFCC|FF0055|Yellow|Green", 0)`. Caveat: it calls `osQueueCreate("analytics_bus", 0)`, but the interface only has `osQueueCreate(string name)` (`IOSSL_Api.cs:588`) — a stale arity (and `[OsQ]` must be enabled). Whether the engine rejects or ignores the extra argument at compile time was **not checked**.
- **`tasks/mragent_rag.lsl`** (151 lines): a Cue–Tag–Content "MRAgent" demo, **not** a web-ingestion pipeline. Nine hard-coded `osDbInsert` rows (3 content nodes with 3-D embeddings, 3 tags, 3 cues), `osDbCreateVectorIndex(…,"embedding","cosine")`, keyword matching with `llSubStringIndex` to pick tags and a canned query vector, `osDbFindVectorNearestScores(…, 2)` (it then iterates the strided result one element at a time and prints rows and scores as separate lines), and `osAIChat(MODEL, messages)` to synthesise. Touch runs a sample query; the listen handler runs chat text as a query.
- **`tasks/karpathywiki_rag.lsl`**: uses `osChunk(WIKI_ARTICLE, 220, 30, "\n\n")` and calls `osDbCreateVectorIndex(g_storeId, "embedding", 3, "cosine")` — four arguments, but the interface has three (`db, field, metric`); stale arity.
- **`scripts/brain.lsl`** (903 lines): the production NPC brain — `#include "brain-lib"`, `osDbCreate("chat_log", …)` with a deliberate first insert/delete to materialise the notecard (comment: osDbCreate alone does not create it, `:147-165`), `osNpcCreate/osNpcSay/osNpcStand/osNpcRemove`, an HTTP bridge with bearer token from the `config` notecard, `osJsonCreateStore`/`osJsonGetValue` for replies (`{npc_response}`, `{sl_commands}`). It does not use osAI/osQ/vector search. The `#include` requires `[ZEngine] EnableInclude = true` (set in the shipped ini).

### Corrected adaptation: web page → embeddings → OsDb vector index → bar chart
Not in the repo. Fixes versus the previous listing: `osPlotFace` **series must be numbers** (labels via `labels:`), the Scores result is **strided**, queries/embeddings are CSV strings from `osMLEmbedAsync`, and the embedding model name is registered once.

```lsl
key g_ragDb;

ingest(string url)
{
    g_ragDb = osDbCreate("web_rag", "web_rag_store");
    await {
        osMLLoadEmbeddingsAsync("emb", "openrouter", "openai/text-embedding-3-small") -> string embStatus;
        osFetchCleanAsync(url, []) -> string pageText;
    } : 20.0 else { llOwnerSay("Ingestion timed out."); return; }

    if (llGetSubString(embStatus, 0, 5) == "ERROR:") { llOwnerSay(embStatus); return; }
    list chunks = osChunk(pageText, 300, 40, "\n");
    integer n = llGetListLength(chunks);
    if (n > 15) n = 15;
    list rows = [];
    integer i;
    for (i = 0; i < n; ++i)
    {
        string c = llList2String(chunks, i);
        osMLEmbedAsync("emb", c) -> string vecCsv;               // "0.01,0.02,…"
        rows += [f"""{{"id":"c{i}","text":"{osEscapeJson(c)}","vec":[{vecCsv}]}}"""];
    }
    osDbInsertMany(g_ragDb, rows);                                // one notecard write for the batch
    osDbCreateVectorIndex(g_ragDb, "vec", "cosine");
    llOwnerSay(f"Indexed {n} chunks.");
}

ask(string question)
{
    osMLEmbedAsync("emb", question) -> string qCsv;
    list hits = osDbFindVectorNearestScores(g_ragDb, "vec", llParseString2List(qCsv, [","], []), 4);
    list scores = [];  string ctx = "";  string labels = "";
    integer i;
    for (i = 0; i < llGetListLength(hits); i += 2)               // [rowJson, score, rowJson, score, …]
    {
        string rowJson = llList2String(hits, i);
        float  score   = (float)llList2String(hits, i + 1);
        scores += [score];                                        // numbers only
        labels += (labels != "") * "|" + llJsonGetValue(rowJson, ["id"]);
        ctx    += f"\n[{llJsonGetValue(rowJson, ["id"])} {score}]: {llJsonGetValue(rowJson, ["text"])}";
    }
    osPlotFace("bar", scores, f"width:512,height:512,title:RAG similarity,labels:{labels}", 0);
    osAIChatAsync("openai/gpt-4o-mini", ["system", f"Answer only from:\n{ctx}", "user", question], 300, ["temperature", 0.2]) -> string answer;
    llSay(0, answer);
}

default { touch_start(integer n) { ingest("https://en.wikipedia.org/wiki/OpenSimulator"); ask("Which physics engines does OpenSimulator use?"); } }
```
Checked facts it depends on: `osChunk(text,size,overlap,delimiter)` and `osMLEmbed*` signatures (`IOSSL_Api.cs:639-641,699-710`); vector metric string `"cosine"`; row ids auto-assigned only when `"id"` missing, here set explicitly; result order is nearest first. Limits to expect: the whole chain shares the 8-slot async gate; each embedding is a paid HTTP round trip; the previous page's `(string)` handling `llJsonGetValue(rowJson,["_score"])` never worked because no `_score` field exists. The `(labels != "") * "|"` string repetition uses ZEXT `str * n`; replace with an `if` if you run with `ZEXT=false`.

---

## Corrections vs previous doc
- Listing 1 was **not** `os6_demo.lsl`: the real script is synchronous, uses `llSetTimerEvent` and polls three osHTTP URLs sequentially, builds a different prompt, and its queue subscribers are `""` and `"analytics.*"`.
- Listing 1 declared `timer "…" -> 5.0: {…}` inside a helper function (compile error — inline timers must be in a state/event) and treated `osQueueConsume` as returning `[msgId, body]`; it returns `[msgId, body, topic]`.
- Listing 2 was attributed to `mragent_rag.lsl`; that script is a keyword/graph demo. The `_score` JSON field, string labels in `osPlotFace` series and `osDbFindVectorNearestAsync` do not exist.
- Missing prerequisite: `[OsQ] Enabled = true` is not in any shipped config.
- Stale script facts found: `craftsim_analyst_demo.lsl` comment on `->` in functions (now supported), `osQueueCreate(name, 0)` and 4-argument `osDbCreateVectorIndex` in two repo scripts.
