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

# OSSL Extensions (AI, ML, OsDb, OsQ) - API surface, AI and ML

> Part of [[OSSL Extensions (AI, ML, OsDb, OsQ)]]. Sections: how the API surface is built, `osAI*`/`osJev*`, `osML*`.

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

---
Back to [[OSSL Extensions (AI, ML, OsDb, OsQ)]].
