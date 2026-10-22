---
tags:
  - craftsim
  - ossl
  - osq
  - oscache
  - osjson
  - queues
  - pubsub
  - cache
  - module
aliases:
  - OsQ
  - OsCache
  - JsonStore
  - osQueue*
  - osCache*
  - osJson*
---

# Module - OsQ, OsCache & JsonStore (Messaging, Caching & JSON)

← Back to [[CraftSim]] | Related: [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[Module - OsDb & Vector Search]], [[ZEXT Language Extensions]]

Three small in-memory region modules under `OpenSim/Region/OptionalModules/Scripting/`, all exposed through OSSL (`OSSL_Api.cs`):

1. **OsQ (`osQueue*`)**: topic queues with per-subscriber cursors, in-flight windows and ack.
2. **OsCache (`osCache*`)**: region-wide `(scope,key)` string cache with TTL and a size cap.
3. **JsonStore (`osJson*`)**: handle-addressed JSON trees with a path language.

None of the three persists anything; all state is lost on region restart.

> Verified against code (commit `9fea6a0332`, 2026-10-16).

---

## 1. OsQ - queues and pub/sub

Files: `OsQ/OsQModule.cs` (406 lines), `OsQ/OsQStore.cs` (340). OSSL: `OSSL_Api.cs:5038-5390`; signatures `IOSSL_Api.cs:588-603`; `Allow_osQueue*` lines `osslEnable.ini:401-415` (threat level `High` on all).

### Configuration (important)
`OsQModule.Initialise` (`OsQModule.cs:~64`) returns early when `[OsQ]` is missing and `m_enabled` defaults to `false`. **Neither `bin/OpenSim.ini` nor `OpenSimDefaults.ini` contains an `[OsQ]` section**, so the module is **off until you add**:

```ini
[OsQ]
    Enabled = True
```
With the module off, `osQueueCreate` returns `NULL_KEY`. (The scripts in `tasks/osq_*.lsl` rely on this being set; unverified whether any deployment config does so.)

### Model
- **Registry** (`OsQModule`): `queueId -> {Lock, OwnerHostId, Name, OsQStore}` and a `(hostId, name) -> queueId` map. `osQueueCreate(name)` is idempotent **per prim**: the same prim gets the same id back for the same name; a different prim creating the same name gets a different, separate queue. Other prims must be handed the id (`llMessageLinked`, `osQueueList()`), there is no lookup by name.
- **Messages** (`OsQStore.Publish`): `{Id (UUID), Body, Topic, TimestampTicks, ConsumedBy, InFlightBy, IsRemoved}`. Messages are retained in a list even when there are no subscribers. A new subscriber starts at **cursor 0**, so it sees every retained, un-removed message published before it subscribed.
- **Subscription**: `{Id, TopicFilter, Cursor, InFlightCount, MaxInFlight}`; every subscription has its own cursor and window.
- **Topic filter** (`TopicMatches`, `OsQStore.cs:~280`): `""` or `"*"` = everything; a filter ending in `*` is a **prefix match** (`"weather.*"` matches `weather.rain`; `"weather*"` would match `weatherX` too); otherwise exact, case-sensitive. A `*` anywhere but the end is treated literally.
- **Consume** (`OsQStore.Consume`): if `MaxInFlight > 0 && InFlightCount >= MaxInFlight` returns nothing (backpressure). Otherwise scans from the cursor for the next non-removed message that matches the filter and is not already consumed/in-flight for this subscription, advances the cursor past it, marks it in-flight, increments `InFlightCount`. Returns `[msgId, body, topic]` (all strings) or `[]`.
- **Ack**: removes the message from the subscription's in-flight set, decrements `InFlightCount`, records it in `ConsumedBy`; the message is flagged `IsRemoved` once **every subscription whose filter matches it** has acked (or if no subscription matches). Unacked messages are never redelivered or timed out; there is no visibility timeout, dead-letter or retry. The `Ack` lookup is a linear `FirstOrDefault` over the message list (O(N)).
- **MaxInFlight**: `0` = unlimited (default for new subscriptions; `DefaultMaxConcurrency` is 0 and has no OSSL setter). `osQueueSetMaxConcurrency` clamps negatives to 0.
- **Purge** clears all messages and topics and resets every cursor/in-flight counter, returning the number of live messages dropped.
- **Timeout caveat**: `osQueueConsume(q, sub, timeout)` with `timeout > 0` runs `ConsumeWithTimeout` which polls with `Thread.Sleep(50)` **while holding the queue's lock and blocking the script thread**; publishers and other consumers on that queue wait too (`OsQModule.Consume`, `OsQStore.ConsumeWithTimeout`). Prefer `timeout = 0.0` and a timer/`llSetTimerEvent` loop. It only wakes when the message count grows.

### Signatures (all verified in `IOSSL_Api.cs`)
| Function | Signature | Notes |
| :--- | :--- | :--- |
| `osQueueCreate` | `key osQueueCreate(string name)` | Blank name or disabled module: `CreateQueue` returns false but the wrapper still returns `UUID.Zero` as a string (`00000000-0000-...`), so check for that rather than relying on `NULL_KEY` alone (`OSSL_Api.cs:5042-5058`). |
| `osQueueDelete` | `void osQueueDelete(key q)` | Drops the queue and all subscriptions. |
| `osQueuePublish` | `void osQueuePublish(key q, string topic, string message)` | Null topic/body become `""`. |
| `osQueueSubscribe` | `key osQueueSubscribe(key q, string topicFilter)` | Returns subscription id. |
| `osQueueUnsubscribe` | `void osQueueUnsubscribe(key q, key subId)` | The module searches all queues for the sub. |
| `osQueueConsume` | `list osQueueConsume(key q, key sub)` and `(key q, key sub, double timeout)` | `[msgId, body, topic]` or `[]`. 2-arg form = timeout 0. |
| `osQueuePeek` | `list osQueuePeek(key q, key sub)` | Next matching message without in-flight/cursor change. |
| `osQueueAck` | `void osQueueAck(key q, key sub, key msgId)` | No-op if the message is not in-flight for that sub. |
| `osQueueSetMaxConcurrency` / `osQueueGetMaxConcurrency` | `void (q, sub, int)` / `int (q, sub)` | |
| `osQueueGetInFlightCount` / `osQueueGetCursor` | `int (q, sub)` | |
| `osQueueStats` | `list osQueueStats(key q)` | **Four** integers: `[messageCount, subscriptionCount, topicCount, defaultMaxConcurrency]` (`OsQModule.Stats`). |
| `osQueuePurge` | `int osQueuePurge(key q)` | |
| `osQueueList` | `list osQueueList()` | Stride 4: `[queueId, name, messageCount, subCount, ...]`. |

There is no `osQueueGetTopicFilter` in OSSL even though the module exposes `GetSubscriptionTopicFilter`.

### Example (from `tasks/osq_demo.lsl`)
The shipped demo creates `"osQ-demo-queue"`, subscribes `""`, `"weather.*"` and `"sports.*"`, sets `osQueueSetMaxConcurrency(queueId, subAll, 2)`, publishes six messages, consumes/acks per subscriber and finally prints stats/peek/cursor, lists queues, purges and unsubscribes. Backpressure check (logic follows from `OsQStore.Consume`):

```lsl
key q  = osQueueCreate("region-bus");
key s  = osQueueSubscribe(q, "telemetry.*");
osQueueSetMaxConcurrency(q, s, 2);
osQueuePublish(q, "telemetry.fps", "{}"); osQueuePublish(q, "telemetry.mem", "{}"); osQueuePublish(q, "telemetry.extra", "{}");
list m1 = osQueueConsume(q, s);   // fps
list m2 = osQueueConsume(q, s);   // mem
list m3 = osQueueConsume(q, s);   // [] : 2 in flight
osQueueAck(q, s, (key)llList2String(m1, 0));
m3 = osQueueConsume(q, s);        // now telemetry.extra
```
Producer/consumer split across prims: `tasks/osq_producer.lsl`, `tasks/osq_consumer.lsl`. No C# unit tests exist for OsQ (unverified claim based on `grep` of `OpenSim/**/*Test*.cs`).

---

## 2. OsCache - TTL key/value cache

File: `OsCache/OsCacheModule.cs` (247 lines); OSSL `OSSL_Api.cs:4920-5000`; `IOSSL_Api.cs:579-585`; enable lines `osslEnable.ini:87-93`; tests `OpenSim/Region/ScriptEngine/Shared/Tests/OSSL_ApiCacheTest.cs` (set/get, TTL expiry, delete/clear).

- **Enabled by default**: `m_enabled = true` (`OsCacheModule.cs:~49`); no `[OsCache]` section is needed or present in the shipped inis. Optional section:

| Key | Default | Meaning |
| :--- | :--- | :--- |
| `[OsCache] Enabled` | `true` | Disable the module. |
| `[OsCache] MaxEntries` | `10000` (`DEFAULT_MAX_ENTRIES`) | When `Count >= MaxEntries` on a `set`, expired entries are purged; if still full, the **10 %** (`Max(1, MaxEntries/10)`) least-recently-accessed entries are evicted. |

- **Storage**: one `ConcurrentDictionary<(scope,key), {Value, ExpiresAt?, LastAccessed}>` per region, shared by all scripts and owners in the region (no per-prim or per-owner isolation; scope is only a naming convention). Keys are case-sensitive.
- **TTL**: `ttlSeconds > 0` sets an absolute UTC expiry; `ttlSeconds <= 0` means **never expires** (still subject to LRU eviction). Expired entries are removed lazily on `get`/`has` and by a 30 s timer (`PurgeExpired`).
- **Blank scope or key** makes `set` return 0, `get` return `""`, `has`/`delete` return 0.
- `osCacheClear(scope)` removes entries whose scope matches **case-insensitively** and returns the count; **an empty scope clears the entire cache**. (Lookups are case-sensitive, so `clear("ScopeA")` also wipes `"scopea"` entries.)
- `osCacheSet` returns 1 on success; `osCacheGet` returns `""` on miss (cannot be told from a stored empty string, use `osCacheHas`).
- **Async**: `osCacheGetAsync` / `osCacheSetAsync` use the same 8-slot gate and `dataserver` delivery as osDb/osAI async (see [[Module - OsDb & Vector Search]] section 5).
- Values are not size-limited by this module (unverified for script memory limits).

| Function | Signature |
| :--- | :--- |
| `osCacheSet` | `int osCacheSet(string scope, string key, string value, double ttlSeconds)` |
| `osCacheGet` | `string osCacheGet(string scope, string key)` |
| `osCacheHas` | `int osCacheHas(string scope, string key)` |
| `osCacheDelete` | `int osCacheDelete(string scope, string key)` |
| `osCacheClear` | `int osCacheClear(string scope)` |
| `osCacheSetAsync` / `osCacheGetAsync` | `key (...)` same args; result by `dataserver` |

Real usage: `tasks/craftsim_async_cache_demo.lsl` sets `session/user_profile:1001` (60 s) and `session/app_config` (120 s) and then reads with `await { osCacheGetAsync(...) -> string cachedUser; osDbFindAsync(...) -> ...; osFetchCleanAsync(...) -> ...; } : 60.0 else {...}`.

Read-through pattern (illustrative, not a repo file):
```lsl
string cached(string city) {
    if (osCacheHas("weather", city)) return osCacheGet("weather", city);
    string r = osHTTPGet("https://wttr.in/" + llEscapeURL(city) + "?format=3", []);
    osCacheSet("weather", city, r, 300.0);
    return r;
}
```

---

## 3. JsonStore (`osJson*`)

Files: `JsonStore/JsonStore.cs` (1013), `JsonStoreModule.cs` (635), `JsonStoreScriptModule.cs` (815, the older `JsonXxx` LSL-ish API), `JsonStoreCommands.cs` (195, console), tests `JsonStore/Tests/JsonStoreScriptModuleTests.cs`. OSSL: `OSSL_Api.cs:4097-4240`; `IOSSL_Api.cs:537-546`; `Allow_osJson*` `osslEnable.ini:39-48`. Threat level is **VeryLow** (unlike the other modules).

### Configuration
| Key | `OpenSimDefaults.ini:2157` | shipped `OpenSim.ini:1178` | Effect |
| :--- | :--- | :--- | :--- |
| `[JsonStore] Enabled` | `False` | `True` | Module master switch (`m_enabled = false` in code). |
| `EnableObjectStore` | `False` | not set | Per-prim dynamic-attribute store (`JsonAttachObjectStore`). |
| `MaxStringSpace` | `65536` | `65536` | Max **UTF-8 bytes per store** (`SerializedByteSize`); `<=0` = unlimited. Enforced in `CreateStore` and on mutations (`JsonStoreModule.cs:~273, ~430`). |
| `MaxStoreCount` | `1024` | `1024` | Max stores in the region (shared store included); `<=0` = unlimited. |

### Behaviour
- `osJsonCreateStore(string json)` returns a store id (`NULL_KEY` on invalid JSON, over-size, or count limit). Empty string creates `{}`. Stores are region-wide by id; any script holding the id can use it. Stores created from OSSL are **not** auto-freed (the object-removal hook only destroys stores keyed by a prim UUID), so call `osJsonDestroyStore` or you will eventually hit `MaxStoreCount`.
- **JSONL support** (`ParseJsonOrJsonLines`, `JsonStore.cs:179`; `HasContentAfterFirstValue`, `:~190`): if anything other than whitespace / `//` comments follows the first complete top-level `{...}` or `[...]`, the input is split on `\n`, blank lines skipped, each line parsed on its own and the results become a **top-level array**. A line containing two values throws (store creation fails). Single-value input keeps the legacy `OSDParser.DeserializeJson` path. Before this, `OSDParser` silently kept only the first record. (The comment block in `tasks/stress/jsonl_wrap_demo.lsl` still describes the old truncation behaviour; stale. No unit test in `OpenSim/**/*Test*.cs` mentions JSONL: unverified by test.)
- **Path language** (`ParsePathExpression`, `JsonStore.cs:738`): `/` is normalized to `.`; components are `name` (letters, digits, `_` only), `{any name}` (braces for keys with other characters), `[N]` array index, `[+]` append (set only), or a bare run of digits which is treated as an array index (`"0/x"` == `[0].x`). Anything else (e.g. a key containing `-` or a space without braces) makes the path invalid and the call returns `""`/does nothing. An empty path addresses the root.
- `osJsonGetValue` returns scalars unquoted (`useJson=false`), `osJsonSetValue(store, path, value)` takes `value` as **JSON text** (`module.SetValue(..., true)`), so a string value needs its own double quotes (a helper that wraps text in `llChar(34)` keeps the script free of backslashes, see the example below), while numbers/booleans go bare (`"250"`, `"true"`) (see `tasks/stress/json_demo.lsl`).
- `osJsonGetNodeType` / `osJsonGetValueType` return the `JSON_NODETYPE_*` / `JSON_VALUETYPE_*` enum values (`-1` if module missing); `osJsonArrayLength(store, "")` is the root length; `osJsonList2Path(list)` joins elements with `/` (integers as numbers).
- All mutations take a per-store `lock(map)` and recompute `StringSpace`; a mutation that would exceed `MaxStringSpace` is rejected.

| Function | Signature |
| :--- | :--- |
| `osJsonCreateStore` / `osJsonDestroyStore` | `key (string json)` / `void (key store)` |
| `osJsonStoreExists` | `int (key store)` |
| `osJsonGetValue` | `string (key store, string path)` |
| `osJsonSetValue` | `void (key store, string path, string jsonValue)` |
| `osJsonRemoveValue` | `void (key store, string path)` |
| `osJsonGetNodeType` / `osJsonGetValueType` | `int (key store, string path)` |
| `osJsonArrayLength` | `int (key store, string path)` |
| `osJsonList2Path` | `string (list pathElements)` |

The richer `JsonXxx` methods in `JsonStoreScriptModule.cs` (`JsonReadNotecard`, `JsonWriteNotecard`, `JsonTakeValue`, `JsonRezAtRoot`...) belong to the legacy OpenSim `json*` script module (needs its own `[JsonStore]` config) and are separate from the `osJson*` surface above.

### Example (run on the live sim 2026-10-17, `tasks/zext/osjson_readable.lsl`)
```lsl
// osjson_readable.lsl - osJson* without escaped quotes: triple-quoted JSON, a quote helper, and JSON Lines on two lines.
string jstr(string s)                            // a JSON string value is its text between double quotes
{
    string dq = llChar(34);
    return dq + s + dq;
}

default
{
    state_entry()
    {
        key cfg = osJsonCreateStore("""{"name": "lamp", "tags": ["a"]}""");
        osJsonSetValue(cfg, "region", jstr("pakkio"));        // a string needs its quotes; numbers and booleans do not
        osJsonSetValue(cfg, "level", "3");
        osJsonSetValue(cfg, "on", "true");
        llOwnerSay("name " + osJsonGetValue(cfg, "name") + ", region " + osJsonGetValue(cfg, "region")
                   + ", level " + osJsonGetValue(cfg, "level") + ", on " + osJsonGetValue(cfg, "on")
                   + ", first tag " + osJsonGetValue(cfg, "tags.0"));

        // JSON Lines: one object per line, loaded as an array
        key rows = osJsonCreateStore("""{"id":1,"tags":["a"]}
{"id":2,"tags":["b"]}""");
        llOwnerSay("rows: " + (string)osJsonArrayLength(rows, "") + ", second id " + osJsonGetValue(rows, "1.id")
                   + ", its tag " + osJsonGetValue(rows, "1.tags.0"));
    }
}
```

**Output**

```text
name lamp, region pakkio, level 3, on , first tag a
rows: 2, second id 2, its tag b
```

Notes from the run: the JSON Lines text sits in a triple-quoted string over two lines and loads as a top-level array of two rows; paths such as `1.tags.0` address array elements; and `osJsonGetValue(cfg, "on")` for a stored `true` came back empty, so do not rely on reading booleans back as text.
Other demos: `tasks/stress/json_extract_stress.lsl`, `jsonstore_nested_array_demo.lsl`, `json_incr.lsl`.

---

## Corrections vs previous doc

- **OsQ is not enabled by default and has no ini section anywhere**; it needs `[OsQ] Enabled = True`. The old note did not say so. **OsCache is enabled by default** (optional `[OsCache] Enabled`, `MaxEntries` = 10000 with 10 % LRU eviction), which was missing.
- `osQueueStats` returns **four** integers (adds `defaultMaxConcurrency`), not three. `osQueueList` is stride 4 `[id, name, msgCount, subCount]`. `osQueueGetCursor`, `osQueueUnsubscribe` and the 2-arg `osQueueConsume` exist and were missing/misdescribed.
- Topic filter is only exact / `""` / `"*"` / trailing-`*` prefix; the old text implied general wildcards. Queue lookup is per creating prim, not global by name. New subscribers see retained history from cursor 0. Timeout consume blocks under the queue lock.
- Old example comment "`dropped_if_unacked`" suggested unacked messages are dropped; they are never dropped or redelivered.
- OsCache: `ttl <= 0` means no expiry; `osCacheClear("")` clears everything; clear matches scope case-insensitively. Old read-through example was fine in intent but `resp[0:5]` slicing is ZEXT-specific and unverified, so it was replaced by a simpler one.
- JsonStore: old claim "fixing `OSDParser`'s truncation" is right but the old demo path `tasks/stress/json_demo.lsl` does **not** contain JSONL (it builds config objects and arrays); the example was rewritten. Added defaults (`Enabled` False in Defaults, `EnableObjectStore`), VeryLow threat level, path grammar (`[+]`, `{key}`, digits-as-index, `/` allowed), that `osJsonSetValue` takes JSON text, and that OSSL-created stores are never auto-destroyed. `osJsonGetNodeType`/`GetValueType` return enum ints.
- `osJsonCreateStore` failure modes (invalid JSON, size, count) added.

## Verified against code (commit 9fea6a0332, 2026-10-16)
Read: `OsQModule.cs`, `OsQStore.cs`, `OsCacheModule.cs`, `JsonStore.cs` (parse + path), `JsonStoreModule.cs` (config, create), OSSL wrappers, `osslEnable.ini`, ini files, demo scripts, `OSSL_ApiCacheTest.cs`. Not executed: no script or test was run for this pass.
