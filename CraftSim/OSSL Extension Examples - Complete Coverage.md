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

## 1. osDb - lifecycle, queries, import/export

### 1.1 Players table with an index, an ordered report and housekeeping
Covers `osDbList`, `osDbAttach`, `osDbCreate`, `osDbRowCount`, `osDbInsert`, `osDbCreateIndex`, `osDbFindByIndex`, `osDbFindOrderBy`, `osDbFind`, `osDbDeleteWhere`, `osDbFlush`, `osDbDropIndex`. Secondary indexes are **not persisted**, so build them on every start; `osDbFind` does a full scan and does not use them; `osDbFindByIndex` returns row **ids**, not rows.

```lsl
key db;

string rowOf(string id, string name, string zone, integer score)
{
    return "{\"id\":\"" + id + "\",\"name\":\"" + name + "\",\"zone\":\"" + zone
         + "\",\"score\":" + (string)score + "}";
}

default
{
    state_entry()
    {
        // osDbList is a flat list, stride 4: [id, nick, refcount, attachedHostsCsv, ...]
        list dbs = osDbList();
        integer i;
        for (i = 0; i < llGetListLength(dbs); i += 4)
            if (llList2String(dbs, i + 1) == "players")
                db = osDbAttach((key)llList2String(dbs, i));
        if (db == NULL_KEY || db == (key)"")
            db = osDbCreate("players", "players_data");     // nick, notecard name (both required)

        if (osDbRowCount(db) == 0)
        {
            osDbInsert(db, rowOf("p1", "Ada",   "north", 120));
            osDbInsert(db, rowOf("p2", "Bruno", "north",  80));
            osDbInsert(db, rowOf("p3", "Chen",  "south", 200));
            osDbInsert(db, rowOf("p4", "Dora",  "south",  15));
        }

        // Secondary indexes are not persisted: build them on every start.
        osDbCreateIndex(db, "zone");

        // osDbFindByIndex returns row ids, not rows.
        list north = osDbFindByIndex(db, "zone", "north");
        llOwnerSay("north rows: " + (string)llGetListLength(north));

        // top 3 scores >= 50, descending; each element is the row as JSON text
        list top = osDbFindOrderBy(db, "score", ">=", "50", "score", FALSE, 3);
        for (i = 0; i < llGetListLength(top); ++i)
            llOwnerSay("#" + (string)(i + 1) + " " + llList2String(top, i));

        list low = osDbFind(db, "score", "<", "50");       // full scan: ignores secondary indexes
        llOwnerSay("low scorers: " + (string)llGetListLength(low));

        integer gone = osDbDeleteWhere(db, "score", "<", "50");
        llOwnerSay("deleted " + (string)gone + ", remaining " + (string)osDbRowCount(db));
        osDbFlush(db);
        osDbDropIndex(db, "zone");
    }
}
```

**Output** (run 2026-10-17):

```text
north rows: 2
#1 {"id":"p3","name":"Chen","zone":"south","score":200}
#2 {"id":"p1","name":"Ada","zone":"north","score":120}
#3 {"id":"p2","name":"Bruno","zone":"north","score":80}
low scorers: 1
deleted 1, remaining 3
```

### 1.2 Backup and restore: `osDbExport`, `osDbImport`, `osDbDetach`, `osDbDrop`
`osDbDetach` persists and frees the in-memory store when the last attached prim lets go; `osDbDrop` removes the registry entry (the notecard stays). Export returns `1` here, not a row count; import returns the number of rows appended.

```lsl
key live;
key scratch;

default
{
    state_entry()
    {
        live = osDbCreate("orders", "orders_data");
        integer i;
        for (i = 0; i < 20; ++i)
            osDbInsert(live, "{\"id\":\"o" + (string)i + "\",\"qty\":" + (string)(i % 5)
                             + ",\"sku\":\"S" + (string)(i % 3) + "\"}");

        // Snapshots go to NEW notecards in THIS prim (an existing one with the same name is overwritten).
        integer a = osDbExport(live, "orders-backup.csv",   "csv");
        integer b = osDbExport(live, "orders-backup.jsonl", "jsonl");
        llOwnerSay("exported csv=" + (string)a + " jsonl=" + (string)b);

        // Restore into a second store. Import APPENDS (it never clears), and CSV values come back as strings.
        scratch = osDbCreate("orders_restore", "orders_restore_data");
        integer n = osDbImport(scratch, "orders-backup.jsonl", "jsonl");
        llOwnerSay("imported " + (string)n + ", rows now " + (string)osDbRowCount(scratch));

        // Detach persists and releases rows from memory when the last prim lets go;
        // Drop removes the registry entry (the notecard itself is left alone).
        osDbDetach(scratch);
        osDbDrop(scratch);
        osDbDetach(live);
        llOwnerSay("done");
    }
}
```

**Output** (run 2026-10-17):

```text
exported csv=1 jsonl=1
imported 20, rows now 20
done
```

---

## 2. osJev - typed decisions with Jev, SubQ and the other decision models

Background, model table and the benchmark: [[Module - osAI & osJev]], [[Decision Models Benchmark]]. The default model (`perplexity/pplx-decider-v1.2`) rate-limits bursts (`HTTP 429 ... retry after 1s`); scripts that make several calls in a row should retry, as 2.4 and 2.5 do. Run these one at a time.

### 2.1 `osJevNoul`: a probability
`osJevNoul(state, question)` returns 0.0 to 1.0, or `-1.0` on any error.

```lsl
// jev_noul.lsl - osJevNoul: how likely is a statement true? (0.0 .. 1.0, -1.0 on error or missing key)
// Run: zengine run tasks/jev/jev_noul.lsl --wait 20000
list TICKETS = [
    "My parcel arrived two weeks late and the box was crushed. I want my money back.",
    "Thanks, the parcel arrived on time and everything is perfect.",
    "Can you tell me whether the blue jacket is available in size L?"
];

default
{
    state_entry()
    {
        integer i;
        for (i = 0; i < llGetListLength(TICKETS); ++i)
        {
            string ticket = llList2String(TICKETS, i);
            float p = osJevNoul(ticket, "Is the customer asking for a refund?");
            if (p < 0.0)
                llOwnerSay("error or no key (osJevNoul returned -1.0)");
            else
                llOwnerSay((string)llRound(p * 100.0) + "% refund | " + llGetSubString(ticket, 0, 40) + "...");
        }
    }
}
```

**Output** (run 2026-10-17):

```text
100% refund | My parcel arrived two weeks late and the ...
0% refund | Thanks, the parcel arrived on time and ev...
0% refund | Can you tell me whether the blue jacket i...
```

### 2.2 `osJevChoice` and `osJevChoiceExt`: pick one id
`osJevChoice` takes plain ids; `osJevChoiceExt` takes `id, description` pairs, which helps when the ids are terse. Both return the chosen id or a string starting `ERROR`.

```lsl
// jev_choice.lsl - osJevChoice picks one id; osJevChoiceExt gives the model a description for every id.
// Both return the chosen id, or a string starting "ERROR".
default
{
    state_entry()
    {
        string state_txt = "Visitor Anna says: 'Could you come over here, I want to show you something.'";

        // plain ids: the model sees only the words
        string a = osJevChoice(state_txt, "What does the visitor want the NPC to do?",
                               ["greet", "come", "stay", "ignore"]);
        llOwnerSay("plain ids      -> " + a);

        // id, description pairs: clearer when the ids are terse
        string b = osJevChoiceExt(state_txt, "What does the visitor want the NPC to do?", [
            "greet",  "the visitor says hello or addresses the NPC",
            "come",   "the visitor asks the NPC to come over, follow, or walk to them",
            "stay",   "the visitor asks the NPC to stop, wait or stay here",
            "ignore", "anything else"
        ]);
        llOwnerSay("with meanings  -> " + b);

        if (llSubStringIndex(b, "ERROR") == 0)
            llOwnerSay("model or key problem: " + b);
    }
}
```

**Output** (run 2026-10-17):

```text
plain ids      -> come
with meanings  -> come
```

### 2.3 `osJevScore`: a position on an ordered scale
Returns 0.0 (first level) to 1.0 (last level), or `-1.0` on error; it cannot say why, so the script retries before giving up.

```lsl
// jev_score.lsl - osJevScore: position on an ordered scale, 0.0 (first level) .. 1.0 (last level).
list LEVELS = ["low", "medium", "high", "critical"];

default
{
    state_entry()
    {
        list tickets = [
            "Just wanted to say thanks, the delivery guy was lovely.",
            "My daughter's birthday is tomorrow and the gift has not arrived. Where is it?",
            "Urgent: I was charged twice and the second charge overdraws my account today."
        ];
        integer i;
        for (i = 0; i < llGetListLength(tickets); ++i)
        {
            string t = llList2String(tickets, i);
            float s = osJevScore(t, "How urgent is this ticket?", LEVELS);
            // -1.0 means error (no key, HTTP error, rate limit): osJevScore cannot say which, so retry a few times
            integer tries = 0;
            while (s < 0.0 && tries < 3)
            {
                ++tries;
                llSleep(2.0 * tries);
                s = osJevScore(t, "How urgent is this ticket?", LEVELS);
            }
            if (s < 0.0)
            {
                llOwnerSay("no score (error) | " + llGetSubString(t, 0, 40) + "...");
                jump next;
            }
            // 0.0 .. 1.0 -> level index (the last level is reached at 1.0)
            integer idx = (integer)(s * llGetListLength(LEVELS));
            if (idx > 3) idx = 3;
            llOwnerSay(llGetSubString((string)s, 0, 4) + " = " + llList2String(LEVELS, idx) + " | " + llGetSubString(t, 0, 40) + "...");
            @next;
        }
    }
}
```

**Output** (run 2026-10-17):

```text
0.011 = low | Just wanted to say thanks, the delivery g...
0.712 = high | My daughter's birthday is tomorrow and th...
0.741 = high | Urgent: I was charged twice and the secon...
```

(In the first, parallel run this script printed `-1.00 = low` for the first ticket: an error shown as "low". The retry and the `-1.0` check were added because of that.)

### 2.4 `osJevAsk`: several typed questions in one request
One model call answers a choice, a yes/no and a score together. Read the answer JSON with `llJsonGetValue`.

```lsl
// jev_ask.lsl - osJevAsk: several typed questions in ONE request (one model call, one bill).
// Question types: "choice" (criteria = id -> meaning), "noul" (yes/no probability), "score" (levels).
string QUESTIONS = "{"
    + "\"intent\":{\"type\":\"choice\",\"instructions\":\"What does the customer want?\","
    +   "\"criteria\":{\"refund\":\"asks for money back\",\"replace\":\"asks for a replacement\","
    +                "\"info\":\"asks for information only\",\"other\":\"anything else\"}},"
    + "\"angry\":{\"type\":\"noul\",\"instructions\":\"Is the customer angry?\"},"
    + "\"urgency\":{\"type\":\"score\",\"instructions\":\"How urgent is this ticket?\","
    +   "\"levels\":[\"low\",\"medium\",\"high\",\"critical\"]}"
    + "}";

default
{
    state_entry()
    {
        string ticket = "My parcel arrived two weeks late and the box was crushed. I want my money back.";
        string r = osJevAsk(ticket, QUESTIONS);

        // The default model (pplx-decider-v1.2) rate-limits bursts: "ERROR: HTTP 429 ... retry after 1s". Retry with a back-off.
        integer tries = 0;
        while (llSubStringIndex(r, "ERROR") == 0 && llSubStringIndex(r, "429") >= 0 && tries < 4)
        {
            ++tries;
            llSleep(2.0 * tries);
            r = osJevAsk(ticket, QUESTIONS);
        }
        if (llSubStringIndex(r, "ERROR") == 0) { llOwnerSay(r); return; }

        llOwnerSay("model    " + llJsonGetValue(r, ["model"]));
        llOwnerSay("intent   " + llJsonGetValue(r, ["answers", "intent", "choice"])
                 + " (" + llJsonGetValue(r, ["answers", "intent", "probabilities", "refund"]) + " refund)");
        llOwnerSay("angry    " + llJsonGetValue(r, ["answers", "angry", "noul"]));
        llOwnerSay("urgency  " + llJsonGetValue(r, ["answers", "urgency", "score"]));
        llOwnerSay("tokens   " + llJsonGetValue(r, ["usage", "input_tokens"]));
    }
}
```

**Output** (run 2026-10-17):

```text
model    perplexity/pplx-decider-v1.2-20261016
intent   refund (0.995188 refund)
angry    0.92969
urgency  0.553052
tokens   183
```

### 2.5 `osJevAskExt` and `osJevModels`: choose the model per call
The third argument is the model id; `""` means `[OSSL] DecisionModel`. `osJevModels()` lists the ids this build knows (22 here). The helper `ask()` retries on HTTP 429.

```lsl
// jev_models.lsl - osJevAskExt: choose the decision model per call, with a retry for rate limits.
// osJevModels() lists the ids this build knows; "" as the model means [OSSL] DecisionModel.
list MODELS = [
    "typesafe/jev-1.14",
    "subq-snap-preview",
    "perplexity/pplx-decider-flash-v1.2",
    "microsoft/microsoft-decision-1"
];
string QUESTIONS = "{\"intent\":{\"type\":\"choice\",\"instructions\":\"What does the customer want?\",\"criteria\":{\"refund\":\"asks for money back\",\"other\":\"anything else\"}}}";

// One call; on HTTP 429 wait 2 s, 4 s, 6 s, 8 s and try again.
string ask(string state_txt, string model)
{
    string r = osJevAskExt(state_txt, QUESTIONS, model, "");
    integer tries = 0;
    while (llSubStringIndex(r, "ERROR") == 0 && llSubStringIndex(r, "429") >= 0 && tries < 4)
    {
        ++tries;
        llSleep(2.0 * tries);
        r = osJevAskExt(state_txt, QUESTIONS, model, "");
    }
    return r;
}

default
{
    state_entry()
    {
        llOwnerSay("known decision models: " + (string)llGetListLength(osJevModels()));
        integer i;
        for (i = 0; i < llGetListLength(MODELS); ++i)
        {
            string m = llList2String(MODELS, i);
            float t0 = llGetTime();
            string r = ask("My parcel arrived two weeks late. I want my money back.", m);
            float dt = llGetTime() - t0;
            if (llSubStringIndex(r, "ERROR") == 0)
                llOwnerSay(m + " " + llGetSubString(r, 0, 90));
            else
                llOwnerSay(m + " " + llGetSubString((string)dt, 0, 4) + "s -> "
                    + llJsonGetValue(r, ["answers", "intent", "choice"])
                    + " " + llGetSubString(llJsonGetValue(r, ["answers", "intent", "confidence"]), 0, 4)
                    + " (" + llJsonGetValue(r, ["model"]) + ")");
        }
    }
}
```

**Output** (run 2026-10-17):

```text
known decision models: 22
typesafe/jev-1.14 0.297s -> refund 1 (typesafe/jev-1.14-20261014)
subq-snap-preview 0.412s -> refund 0.967 (subq-snap-preview)
perplexity/pplx-decider-flash-v1.2 0.229s -> refund 0.992 (perplexity/pplx-decider-flash-v1.2-20261016)
microsoft/microsoft-decision-1 0.799s -> refund 0.995 (microsoft/microsoft-decision-1-20261009)
```

### 2.6 SubQ (`subq-snap-preview`)
Text only: it needs `SubqApiKey`, reports no `cost`, and an image is refused before any request is sent.

```lsl
// jev_subq.lsl - SubQ (subq-snap-preview): text-only decision model, own key, own endpoint.
// Key: [OSSL] SubqAPIKey (Secrets.ini: SubqApiKey) or env SUBQ_API_KEY. Images and osJevBox return ERROR.
string Q = "{\"angry\":{\"type\":\"noul\",\"instructions\":\"Is the customer angry?\"}}";

default
{
    state_entry()
    {
        string r = osJevAskExt("Your support rep hung up on me. I want a manager now, I am furious.", Q, "subq-snap-preview", "");
        if (llSubStringIndex(r, "ERROR") == 0) { llOwnerSay(r); return; }
        llOwnerSay("angry " + llJsonGetValue(r, ["answers", "angry", "noul"])
                 + " | input tokens " + llJsonGetValue(r, ["usage", "input_tokens"]));

        // an image is refused for SubQ before any request is made
        string img = osJevAskExt("Is there a cat in {photo}?", Q, "subq-snap-preview", "{\"photo\":{\"url\":\"https://example.com/x.png\"}}");
        llOwnerSay("with image -> " + llGetSubString(img, 0, 100));
    }
}
```

**Output** (run 2026-10-17):

```text
angry 0.977033 | input tokens 102
with image -> ERROR: images and Box are not supported for SubQ models here; use TypeSafe Jev (typesafe/jev-1.14)
```

### 2.7 ZEXT: `osJevChoiceAsync` with `->`, a timeout and `else`
The script suspends at the bind while other events keep running, and resumes with the answer or runs the `else` branch after 20 s. Needs ZEXT (see [[ZEXT Language Extensions]]).

```lsl
// jev_async_bind.lsl - ZEXT: osJevChoiceAsync with the -> bind, a timeout and an else branch.
// The script suspends at the bind (other events keep running) and resumes with the result.
default
{
    state_entry()
    {
        osJevChoiceAsync("Visitor says: 'thanks, bye!'", "What does the visitor mean?",
                         ["greet", "thanks", "ignore"]) -> string answer : 20.0
        else
        {
            llOwnerSay("no answer in 20 s");
            return;
        }
        llOwnerSay("answer: " + answer);
    }
}
```

**Output** (run 2026-10-17):

```text
answer: thanks
```

### 2.8 `osJevBox` with an image
Jev 1.14 only. The request goes to TypeSafe directly (OpenRouter's decisions endpoint accepts an image but never shows it to the model), so it needs `TypeSafeApiKey`. The image is inline base64 (PNG or JPEG, each side 280-1568 px) or `{"url": "https://..."}`. The result is `[top, bottom, left, right, confidence, present]` as **fractions of the image**.

```lsl
// jev_box.lsl - osJevBox: where is the thing I describe?
// Needs a TypeSafe key ([OSSL] TypeSafeAPIKey) and Jev 1.14; returns [top, bottom, left, right, confidence, present].
// The image goes inline as base64 (PNG or JPEG, each side 280-1568 px); {"url": "https://..."} works too.
// Test image: 400x400, white, a red square at x 100-199 / y 50-149 and a blue circle (radius 50) centred at (300, 300).
// The four edges are fractions of the image (0.0 .. 1.0), not pixels. For the red square the exact answer is
// left 0.25, right 0.50, top 0.125, bottom 0.375; Jev 1.14 returned left 0.27, right 0.48, top 0.10, bottom 0.53,
// confidence 0.53, present 0.99 (2026-10-17): right place, bottom edge too low - treat boxes as approximate.
string IMG = "iVBORw0KGgo...";   // base64 of the 400x400 PNG (1614 bytes), see tasks/jev/make_box.py

default
{
    state_entry()
    {
        string assets = "{\"photo\":{\"data_base64\":\"" + IMG + "\"}}";
        list box = osJevBox("A white image with two shapes.", "the red square", assets);

        if (llGetListLength(box) < 6)
        {
            llOwnerSay("no box: " + llDumpList2String(box, " "));   // e.g. the ERROR text when the key is missing
            return;
        }
        llOwnerSay("top " + llList2String(box, 0) + " bottom " + llList2String(box, 1)
                 + " left " + llList2String(box, 2) + " right " + llList2String(box, 3));
        llOwnerSay("confidence " + llList2String(box, 4) + " present " + llList2String(box, 5));
    }
}
```

**Output** (run 2026-10-17):

```text
top 0.100000 bottom 0.530000 left 0.270000 right 0.480000
confidence 0.530000 present 0.990000
```

The red square really spans left 0.25, right 0.50, top 0.125, bottom 0.375, so the box is in the right place but the bottom edge is too low (0.53 against 0.375): treat Box output as approximate.

---

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
