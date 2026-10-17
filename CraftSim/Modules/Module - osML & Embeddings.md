---
tags:
  - craftsim
  - ossl
  - osml
  - machine-learning
  - embeddings
  - onnx
  - word2vec
  - glove
  - module
aliases:
  - osML
  - osML*
  - osMLEmbed
---

# Module - osML & Embeddings (On-Sim ML, XGBoost, PCA & Vectors)

← Back to [[CraftSim]] | Related: [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[Module - OsDb & Vector Search]], [[Module - osAI & osJev]]

CraftSim has a region-wide **on-sim ML engine** written in plain C# (no ML library) inside `OSSL_Api.cs`, plus a **text-embedding engine** with four backends (`word2vec`, `glove`, local `onnx` via `Microsoft.ML.OnnxRuntime`, and `openrouter`). Models live in two static registries - `m_mlModels` (`OSSL_Api.cs:~6587`) and `m_embedModels` (`:8772`), both case-insensitive `ConcurrentDictionary`s - so a model trained or loaded by one script is usable by every script in the process (not just the region: the dictionaries are static).

Verified against code (commit `9fea6a0332`, 2026-10-16). `OSSL_Api.cs` = `OpenSim/Region/ScriptEngine/Shared/Api/Implementation/OSSL_Api.cs`.

---

## 1. Architecture

### 1.1 Layout of `OSSL_Api.cs` (current line numbers)
| Piece | Lines |
| :--- | :--- |
| `#region osML Functions`, `MLModel`, `MlTree` | 6581-6660 |
| `MlNameSeed` (FNV-1a), `MlShuffleIndices` | 6663, 6678 |
| Parsing / cleaning / encoding: `MlParseRows` 6696, `MlParseTokens` 6793, `MlCleanRows` 6835, `MlPrepare` 6913, `MlEncodeRow` 7030, `MlModelEncodeRow` 7051 | 6696-7080 |
| `MlTrainPca` | 7113 |
| `MlTrainKMeans` / `MlInertia` / `MlAutoK` | 7243 / 7358 / 7377 |
| `MlTrainCategorize` / `MlTrainRegression` / `MlSolveLinearSystem` | 7427 / 7457 / 7507 |
| `MlTrainBoost` / `MlFitTree` | 7562 / 7617 |
| Predict helpers: categorize 7746, regression 7796, boost 7807, cluster 7820 | 7746-7860 |
| `MlEvaluateClassification` / `MlEvaluateRegression` | 7862 / 7961 |
| Public API: `osMLtrain` 8089, `osMLpredict` 8221, `osMLevaluate` 8328, `osMLlist` 8382, `osMLdrop` 8401, `osMLclean` 8420, `osMLsplit` 8532, `*Async` 8627-8730 | 8041-8730 |
| Embeddings (model registry, fetch, parsers, backends, API) | 8772-9565 |

### 1.2 Determinism
`MlNameSeed` hashes the model name with FNV-1a (not `string.GetHashCode()`); k-means uses `Random(MlNameSeed(name) ^ (n*31) ^ (d*17))`, `osMLsplit` shuffles with `Random(seed)` (Fisher-Yates). The code comments say results are stable within a runtime but not guaranteed identical across .NET versions.

### 1.3 Algorithms in `osMLtrain`
Signature: `string osMLtrain(string name, string modelType, LSL_List data[, int k[, int clean[, int epochs]]])` (overloads 3/4/5/6 args, `:8041-8089`). Types (case-insensitive): `pca cluster autocatalogue categorize regression xgboost`. Returns `"OK: <type> model '<name>' trained (<rows> rows x <cols> ...)"` or `"ERROR: ..."`. Threat level High.

Unsupervised (`data` = nested list of numeric rows, `MlParseRows`; needs >= 2 rows; **`clean` is ignored** for these three types):
- `pca` (`MlTrainPca`, `:7113`): mean-centre, sample covariance (`/(n-1)`), cyclic **Jacobi** eigen-decomposition, convergence `maxOff < 1e-12` with a cap of `50*d*d` rotations. **All** components with eigenvalue > 1e-12 are kept, sorted descending; `k` is not used to truncate. Predict returns one projection per kept component.
- `cluster` (`MlTrainKMeans`, `:7243`): **k-means++** seeding + Lloyd iterations; max iterations = `epochs` if > 0 else **100**. `k <= 0` -> `MlAutoK`.
- `MlAutoK` (`:7377`): fits k = 1..min(10, n), computes inertia, and picks the k in [2, maxK-1] with the largest second difference `inertia[k-1] + inertia[k+1] - 2*inertia[k]`. Returns 1 if fewer than 2 candidates.
- `autocatalogue`: same k-means, plus `AutoThreshold = 1.5 * mean(distance of each training row to its nearest centroid)` (`:~8196`). On predict, a sample with `sqrt(dist) > AutoThreshold` appends itself as a new centroid, re-checked under a per-model `GrowthLock` (`MlPredictCluster`, `:7820`). Note growth is lock-protected but the model is still mutated by `osMLpredict` (predict is not read-only for this type).

Supervised (`MlPrepare`, `:6913`): last column = label/target, the rest are features; string feature columns are one-hot encoded (mirrored at predict time via `CatMaps`/`CatOffsets`); needs >= 2 columns and >= 2 rows after cleaning.
- `categorize` (`MlTrainCategorize`, `:7427`): **k-NN** with plain **majority vote** (not distance-weighted), `k` = neighbours (`k <= 0` -> **5**, capped at row count); neighbours ordered by insertion sort on squared distance; a tie goes to the label first seen among the nearest neighbours.
- `regression` (`MlTrainRegression`, `:7457`): closed-form ridge, `w = (XᵀX + λI)⁻¹Xᵀy` with intercept column, **λ = 1e-6 added to every diagonal entry including the bias**, solved by `MlSolveLinearSystem` (Gaussian elimination, `:7507`). Singular -> `ERROR: failed to fit regression`.
- `xgboost` (`MlTrainBoost`, `:7562`): gradient-boosted regression trees on residuals; `eta = 0.1`; rounds = `epochs` if > 0, else `k` if > 0, else **24**. **No upper cap of 200 exists in the code.** Each tree `MlFitTree(X, residual, maxDepth = 3)` (`:7617`): exhaustive split search on midpoints of unique sorted feature values, MSE gain, leaf = mean residual; a node needs **>= 4 samples** (`minSize = 4`) and positive gain to split. (This is a small GBRT, not the XGBoost library: no regularisation, no column subsampling.)

`clean` bitmask (`MlPrepare` and `osMLclean`): `1` drop rows with a non-finite/short numeric value, `2` drop exact duplicates, `4` z-score numeric feature columns (std < 1e-12 -> 1.0). Supervised training applies `mode & ~4` in `MlCleanRows` and does z-scoring in `MlPrepare`; the model stores `NumMean/NumStd` to re-apply on predict. The status string lists dropped rows (`"(prep: drop-invalid-dedupe; dropped N rows)"`).

### 1.4 Predict / evaluate return shapes
`osMLpredict(string name, LSL_List row)` (`:8221`). Unknown model -> `["ERROR: model '<name>' not found"]`.
| Type | Result |
| :--- | :--- |
| `pca` | `[float proj_0, ...]` |
| `cluster` | `[int clusterIndex, float distance]` (Euclidean) |
| `autocatalogue` | `[int clusterIndex, float distance, int isNewEntry]` |
| `categorize` | `[string label, int votes, float distance]` |
| `regression` / `xgboost` | `[float value]` |

`osMLevaluate(name, testData)` (`:8328`) - only supervised types, otherwise `["ERROR: osMLevaluate only supports categorize/regression/xgboost models ..."]`:
- `categorize` -> `["categorize", n, accuracy, macroPrecision, macroRecall, macroF1, K, <nested list of K labels, ordinal-sorted>, <flat K*K confusion matrix, index = actual*K + predicted>]`.
- `regression`/`xgboost` -> `[typeTag, n, mae, rmse, r2]`.

Async delivery (`dataserver`, comma-joined strings): `osMLpredictAsync`, `osMLevaluateAsync` (categorize: `"categorize,n,acc,P,R,F1,K,label1..labelK,cell0..cell(K*K-1)"`), `osMLtrainAsync`, `osMLcleanAsync` (rows joined by `\n`, fields by `,`; `"OK: cleaned (no rows)"` for empty). Floats formatted `0.000000` in some paths. All use the shared 8-slot `m_aiRequestGate`; saturation yields a key string `ERROR: too many in-flight requests (max 8)...` that never completes (see [[Module - osAI & osJev]] section 1.3). `osMLsplit`, `osMLlist`, `osMLdrop` have no async form.

### 1.5 Other functions
- `osMLclean(data[, mode=3])` `:8415-8420`: returns the cleaned rows in the same nested/flat shape; `["ERROR: all rows dropped by cleaning"]` if nothing remains.
- `osMLsplit(data, trainFrac[, evalFrac[, seed]])` `:8522-8532`: requires nested rows; `trainFrac` in (0,1], `evalFrac >= 0`, sum <= 1; sizes via `Math.Round(..., AwayFromZero)`; returns the flat list `[trainN, evalN, testN, row_0, ... row_(n-1)]` (train rows, then eval, then test). Defaults `evalFrac=0`, `seed=0`.
- `osMLlist()` -> strided `[name, type, rows, cols, ...]`; `osMLdrop(name)` -> `1`/`0`.

### 1.6 Embedding guardrails (`[OSSL]` keys read at `OSSL_Api.cs:257-260`)
| Key | Default | Meaning |
| :--- | :--- | :--- |
| `EmbedMaxFetchBytes` | 2 GB | Download cap; `FetchToDiskFromUrl` (`:8802`) streams to disk, checks `Content-Length` and the actual bytes read. |
| `EmbedMaxMemoryBytes` | 2 GB (`<= 0` disables) | `CheckEmbedMemoryBudget` (`:8917`): file size (+ other loaded word2vec/GloVe tables, counted as `count*dim*4`) must fit, else `ERROR: ... would exceed [OSSL] EmbedMaxMemoryBytes ...; osMLEmbedDrop a model first`. **Not applied to `onnx`**. |
| `EmbedCacheDirectory` | `osMLEmbedCache` (relative to base dir) | Download cache; file name = SHA-256 of the source URL + `.dat`, so re-loading the same URL skips the download. |
| `EmbedKeepCacheFiles` | `false` | When false the cached file is deleted after parsing (the ONNX model file is **not** deleted: the vocab file is, the model is kept while the session is live - check before relying on this; unverified). |

`m_embedLoadGate` (`SemaphoreSlim(2,2)`, `MaxConcurrentEmbedLoads = 2`) applies only to `osMLLoadEmbeddingsAsync`; sync load is ungated.

---

## 2. osML embedding API (`OSSL_Api.cs:9358-9565`)
| Function | Signature | Notes |
| :--- | :--- | :--- |
| `osMLLoadEmbeddings` | `string (name, method, source[, vocabSource])` | `method`: `word2vec`, `glove`, `onnx`, `openrouter`. `openrouter`: `source` = model id (e.g. `openai/text-embedding-3-small`), nothing downloaded. Others: `source` must be an `http(s)://` URL (**local file paths are not accepted**). word2vec auto-detects text vs binary (`LooksLikeWord2VecText`, no control bytes in first 4 KB); binary parser `ParseWord2VecBinary` (`:8928`) caps a word at 256 bytes. GloVe = text, no header, dim inferred. `onnx` optional 4th arg = vocab URL (one token per line, id = line number). |
| `osMLLoadEmbeddingsAsync` | `LSL_Key (name, method, source[, vocabSource])` | 2-slot gate. |
| `osMLEmbed` | `LSL_List (name, text)` | Returns floats, or a single `"ERROR: ..."` element. |
| `osMLEmbedAsync` | `LSL_Key (name, text)` | Delivers comma-joined floats. |
| `osMLEmbedList` | `LSL_List ()` | Strided `[name, method, dim]`. **dim is 0 for `onnx` and `openrouter`** (it is only set for word2vec/glove). |
| `osMLEmbedDrop` | `LSL_Integer (name)` | Disposes the ONNX session; `1`/`0`. |

Backend behaviour:
- **word2vec/glove** (`EmbedAverageLookup`, `:9123`): lowercase + whitespace tokenisation only (no punctuation stripping), **mean** of found word vectors, OOV words skipped; `ERROR: all words out of vocabulary` if none match.
- **onnx** (`EmbedOnnx`, `:9198`): single `int64` `input_ids` `[1, seq]`; tokens through the vocab file (unknown -> id 0) or, without a vocab, the FNV-1a `% 30000` fallback (`OnnxFallbackTokenId`, `:9097`, collision-prone). The first output is mean-pooled over its sequence axis. `ValidateOnnxContract` (`:9170`) at load time requires exactly one input named `input_ids`, int64, rank 2, and a float32 first output of rank 2 or 3. It validates the *signature only*; semantic correctness for a given real model is the caller's responsibility (no CLS/SEP, no real tokenizer).
- **openrouter** (`EmbedOpenRouter`, `:9274`): `POST {AIBaseURL}/embeddings` `{model, input}` with the **static key from `osAISetOpenRouterAPIKey`** (env var is not used); returns `data[0].embedding`.

Neither a similarity function nor a vector index lives here: use [[Module - OsDb & Vector Search]] for nearest-neighbour search over embeddings.

---

## 3. Examples

### Example A: supervised pipeline (mirrors `tasks/stress/osml_supervised.lsl`)
That script (verified to exist) exercises `osMLtrain(... "regression", lin, 10, 3)`, `osMLtrain("quad2","xgboost",quad,0,0,60)`, `osMLclean(corpus,3)` and `osMLsplit(bigTerrain,0.6,0.2,42)` + `osMLevaluate`. Condensed:

```lsl
default
{
    state_entry()
    {
        // Ridge with clean=3 (drop invalid + dedupe)
        list dirty = [];
        integer x;
        for (x = 0; x <= 10; ++x) dirty += [[x, 2 * x + 1]];
        dirty += [[3, 7]];            // duplicate -> dropped
        dirty += [["nan", 0.0]];      // non-numeric in a numeric column -> dropped
        llOwnerSay(osMLtrain("linear_model", "regression", dirty, 10, 3));
        llOwnerSay(llList2CSV(osMLpredict("linear_model", [100.0])));   // ~201

        // GBRT, 60 boosting rounds (k=0, clean=0, epochs=60)
        list quad = [];
        for (x = 0; x <= 10; ++x) quad += [[x, x * x]];
        osMLtrain("quad_xgb", "xgboost", quad, 0, 0, 60);
        llOwnerSay(llList2CSV(osMLpredict("quad_xgb", [5.5])));         // near 30.25, tree-quantised

        // Split + held-out evaluation
        list parts = osMLsplit(bigTerrain, 0.6, 0.2, 42);
        integer trainN = llList2Integer(parts, 0);
        integer evalN  = llList2Integer(parts, 1);
        list trainRows = llList2List(parts, 3, 2 + trainN);
        list testRows  = llList2List(parts, 3 + trainN + evalN, -1);
        osMLtrain("biome_clf", "categorize", trainRows, 1);
        llOwnerSay(llDumpList2String(osMLevaluate("biome_clf", testRows), " "));

        osMLdrop("linear_model"); osMLdrop("quad_xgb"); osMLdrop("biome_clf");
    }
}
```
(`bigTerrain` = nested rows `[x, y, "label"]` as in the stress script.) The `[["nan", 0.0]]` row works because a column is numeric only if every row parses; read `MlCleanRows` (`:6835`) before relying on cleaning of mixed columns - a text token in a column makes that whole column categorical in `MlPrepare` unless the row is dropped first (unverified edge case).

### Example B: OpenRouter embeddings
```lsl
default
{
    touch_start(integer n)
    {
        osAISetOpenRouterAPIKey("sk-or-...");   // required: embeddings do not read the env var
        osMLLoadEmbeddingsAsync("sem_emb", "openrouter", "openai/text-embedding-3-small")
            -> string loadRes : 15.0 else { llOwnerSay("load timed out"); return; }
        if (llSubStringIndex(loadRes, "ERROR:") == 0) { llOwnerSay(loadRes); return; }

        await {
            osMLEmbedAsync("sem_emb", "ZEngine compiles LSL directly to native CIL") -> string vecAStr;
            osMLEmbedAsync("sem_emb", "Bytecode compiler and JIT script execution")  -> string vecBStr;
        } : 15.0 else { llOwnerSay("Embedding request timed out."); return; }

        list vecA = llParseString2List(vecAStr, [","], []);
        llOwnerSay("dim=" + (string)llGetListLength(vecA));
        osMLEmbedDrop("sem_emb");
    }
}
```
Use `osMLLoadEmbeddingsAsync(name, "glove", "https://.../glove.6B.50d.txt")` for a local table; check `osMLEmbedList()` for loaded names (dim is 0 for openrouter/onnx). Not executed in this review (needs a paid key).

---

## 4. Limits and known gaps
- Unsupervised types ignore `clean`; PCA ignores `k`.
- **Probable bug**: `MlPredictCategorize` sorts neighbour indices (`order`) but returns `Math.Sqrt(dist[0])`, i.e. the distance to training row 0, not to the nearest neighbour (`OSSL_Api.cs:~7786`). The doc'd "nearestDistance" is therefore unreliable. Not fixed (repo untouched).
- k-NN is brute force O(n) per query with an O(n²) insertion sort; `MlFitTree` split search is O(features · n²) per node; large datasets should use the async forms (capped at 8 in flight).
- Models and embeddings are in-memory only: lost on restart, shared across all scripts, never persisted.
- ONNX embeddings need a model that matches the minimal `input_ids` contract; embeddings sources must be URLs.
- Tests: `OpenSim/Region/ScriptEngine/Shared/Tests/OSSL_ApiMLTest.cs` (14 `public void` members) and stress scripts `tasks/stress/osml_supervised.lsl`, `osml_cluster1000.lsl`. Tests not re-run in this review.

## Verified against code (commit `9fea6a0332`, 2026-10-16)
Read: all training/predict/evaluate/clean/split/async code and the whole embeddings region of `OSSL_Api.cs`, `[OSSL]` init, `bin/config-include/osslEnable.ini` (`Allow_osML*` keys at lines 100-125). No scripts were executed.

## Corrections vs previous doc
- **Line refs** (`7150-8023`, `7330-7740`, ...) were stale; the ML region spans 6581-8730 and embeddings 8772-9565.
- **PCA**: `maxSweeps = 100` was wrong (cap is `50*d*d`; tolerance 1e-12); all positive-eigenvalue components are kept (no top-k).
- **k-means**: default max iterations is **100** (not 50); `epochs` overrides it. `MlAutoK` searches k in [1..min(10,n)] and takes the largest second difference, not "[2..min(8,n-1)]".
- **k-NN**: not distance-weighted; plain majority vote, default k=5.
- **Ridge**: lambda is added to all diagonal entries including the bias.
- **XGBoost**: `minLeaf=2` is wrong (a node needs >= 4 samples); no max-200 epoch cap; rounds fall back to `k`, then 24.
- **Autocatalogue**: threshold is 1.5 x mean training distance (this part was right); added that predict mutates the model.
- **Clean mode**: applies to supervised types only; mode 3 is default for `osMLclean` only.
- **Embeddings**: sources must be URLs (no local paths); `openrouter` uses only the setter key; memory guard excludes ONNX; `EmbedCacheDirectory` and `EmbedKeepCacheFiles` keys were undocumented; list `dim` is 0 for onnx/openrouter.
- **Async**: gate is the shared 8-slot gate; saturation returns an `ERROR:` key. Example B in the old doc compared nothing and treated the load result as a sync return; the example now uses the real `->`/dataserver contract.
- Old Example A's comment "expected ~30.25" overstated accuracy for a depth-3 tree ensemble; marked as approximate.
