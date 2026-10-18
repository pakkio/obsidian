---
tags:
  - craftsim
  - ossl
  - osai
  - osjev
  - openrouter
  - decision-models
  - module
aliases:
  - osAI
  - osAI*
  - osJev
  - osJev*
  - osJevAskExt
---

# Module - osAI & osJev (OpenRouter LLMs, Image Generation & Typed Decisions)

← Back to [[CraftSim]] | Related: [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[Decision Models Benchmark]], [[MCP Server & Live Diagnostics]], [[Module - osML & Embeddings]], [[Build, Configuration & Self-Hosting]]

Two families of `os*` functions that talk to model providers:

- **`osAI*`**: chat, completion, model listing and text-to-image through an OpenAI-compatible endpoint (OpenRouter by default).
- **`osJev*`**: *typed decisions*. You pass a state and a set of typed questions (choice, yes/no probability, score) and get back validated, calibrated answers. The name comes from the Jev decision model; since 2026-10-09 the same API also drives other decision models (Microsoft, GPT-6 Luna, Perplexity pplx-decider, SubQ).

Both families are `ThreatLevel.High` (they spend money), so each function must be enabled in `osslEnable.ini` (`Allow_osJev*`, `Allow_osAI*`).

Verified against `README.md`, `CLAUDE.md`, `OpenSim/Framework/DecisionClient.cs` and the live sim on **2026-10-17**. Line numbers of `OSSL_Api.cs` were not re-checked.

## 1. Keys and configuration

| Setting | Meaning |
|---|---|
| `osAISetOpenRouterAPIKey(key)` / env `OPENROUTER_API_KEY` / `[OSSL] OpenRouterApiKey` (`${Secrets\|OpenRouterKey}`) | OpenRouter key, in that priority order. One key serves `osAI*` and `osJev*`. No `.env` scanning. |
| `[OSSL] AIBaseURL` | endpoint of `osAI*` (default `https://openrouter.ai/api/v1`; any OpenAI-compatible provider works) |
| `[OSSL] ImageModel` | default model of `osAIImage` when a script passes `""` |
| `[OSSL] DecisionModel` | default decision model. **`perplexity/pplx-decider-v1.2`** since 2026-10-17 (was `pplx-decider-v1.1-27b`, retired by OpenRouter; before 2026-10-16 it was the chat model `perplexity/sonar`) |
| `[OSSL] DecisionBaseURL` | decision endpoint base (default OpenRouter) |
| `[OSSL] DecisionProtocol` | `auto` (by model id), `chat` or `decisions` |
| `[OSSL] DecisionsURL` | overrides the derived decisions endpoint |
| `[OSSL] DecisionBoxModel` | model for `osJevBox` (default `typesafe/jev-1.14`) |
| `[OSSL] TypeSafeAPIKey` (`${Secrets\|TypeSafeApiKey}`) / env `TYPESAFE_API_KEY`, `TypeSafeURL` | needed for images and Box (§6.3) |
| `[OSSL] SubqAPIKey` (`${Secrets\|SubqApiKey}`) / env `SUBQ_API_KEY`, `SubqURL` | needed for `subq-*` models (§6.4) |
| env `OPENROUTER_DECISION_MODEL` | model used by the MCP tool `choose_best_tool` |

`osJevSetAPIKey` is a deprecated no-op. `JevModel`, `JevBaseURL` and `JevApiKey` were replaced by `DecisionModel` / `DecisionBaseURL`.

## 2. `osAI*` functions

| Function | Notes |
|---|---|
| `osAIChat(model, messages, maxTokens, options)` / `...Async` | chat completion. Images: after a content string add `"image_url"` and a URL, e.g. `["user","Describe this","image_url","https://…"]`. `options` keys include `temperature`, `top_p`, `seed`, `stop`, `reasoning_effort`. |
| `osAIChatExt(...)` / `...Async` | same, returns the full JSON (content, reasoning, model, usage) |
| `osAICompletion(model, prompt, maxTokens)` / `...Async` | simple text completion |
| `osAIImage(prompt, model, face, extraParams)` / `...Async` | generates an image and draws it on a prim face through the `imagedata` renderer; returns the texture asset UUID or an `ERROR:` string |
| `osAIListModels()` | model ids from the provider's `/models` |
| `osAISetOpenRouterAPIKey(key)` | sets the global key |

The `*Async` variants return a key at once and deliver the result through `dataserver`; in ZEXT scripts they can be bound with `->` (see [[ZEXT Language Extensions]]).

## 3. `osJev*` functions

| Function | Returns |
|---|---|
| `osJevAsk(state, questionsJson)` | answer JSON (§4) |
| `osJevNoul(state, instructions)` | probability 0..1 that the statement is true; **`-1.0` on any error or missing key** |
| `osJevChoice(state, instructions, options)` | the chosen id |
| `osJevChoiceExt(state, instructions, optionsWithDescriptions)` | same, with a description per option (list of id, description pairs) |
| `osJevScore(state, instructions, levels)` | 0.0 (lowest level) to 1.0 (highest) |
| `osJevAskExt(state, questionsJson, model, assetsJson)` | like `osJevAsk`, with a **per-call model** and optional images |
| `osJevBox(state, target, assetsJson)` | `[top, bottom, left, right, confidence, present]` (Jev 1.14 only, needs an image) |
| `osJevModels()` | the decision model ids known to this build |
| every function above has an `*Async` form (not `osJevModels`) | key returned at once, result via `dataserver` |

## 4. Questions and answers

Questions map (the "System One" shape):

```json
{
  "intent":  {"type":"choice","instructions":"What does the customer want?",
              "criteria":{"refund":"asks for money back","replace":"asks for a replacement","info":"asks for information","other":"anything else"}},
  "angry":   {"type":"noul","instructions":"Is the customer angry?"},
  "urgency": {"type":"score","instructions":"How urgent is this ticket?","levels":["low","medium","high","critical"]}
}
```

Limits: 1-255 `criteria` per choice, 2-10 `levels` per score, a maximum number of questions per request (`MaxQuestions`). `box` is the fourth type.

Answer:

```json
{"answers":{
   "intent":{"type":"choice","choice":"refund","confidence":1,"probabilities":{"refund":1,"replace":0,"info":0,"other":0}},
   "angry":{"type":"noul","noul":0.83},
   "urgency":{"type":"score","score":0.55,"confidence":0.64}},
 "model":"typesafe/jev-1.14-20261014",
 "usage":{"input_tokens":419,"output_tokens":77,"cost":0.0000176}}
```

Read it in LSL with `llJsonGetValue(r, ["answers","intent","choice"])`. A score runs 0.0 (first level) to 1.0 (last level); the benchmark script turned it into a level with `floor(score * 4)` (capped at 3) for four levels.

## 5. Validation and errors

`DecisionClient` validates the reply of a chat model: a choice that was not offered is an error, numbers are clamped to 0..1. Keep this if you touch the client; a chat model can return anything.

- **A missing key never throws.** `AskAsync` returns `"ERROR: OpenRouter API key not set …"` before any HTTP request. `osJevNoul` then returns `-1.0`, `osJevChoice*` returns the `ERROR:` string, `osJevAsk*` passes it through, and MCP `choose_best_tool` falls back to a keyword heuristic (`provider: keyword-fallback`). The startup log names the missing key. Pinned by `DecisionClientTests.MissingKeyAndBadQuestionsFailBeforeAnyRequest`.
- **HTTP errors come back as text**, e.g. `ERROR: HTTP 404: {"error":{"message":"No endpoints found for perplexity/pplx-decider-v1.1-27b."…`. Always test for `llSubStringIndex(r, "ERROR") == 0`.
- **`429` on `pplx-decider-v1.2`**: `Rate limit exceeded for model 'pplx-decider-v1.2'; retry after 1s`. It happens even with a gap of a second between calls. `DecisionClient` does not retry; scripts should (§8).

## 6. Decision models

### 6.1 Which model id goes where

| Id prefix | Route |
|---|---|
| `typesafe/`, `~typesafe/`, `jev-`, `microsoft/microsoft-decision`, `openai/gpt-6-luna-decisions`, `perplexity/pplx-decider`, `cloudflare/clef`, `liquid/d1`, `upstage/solar-decide`, … | OpenRouter `/api/alpha/decisions` (key: OpenRouter) |
| `subq-`, `subq/` | `https://api.subq.ai/v1/systemone` (key: SubQ) |
| anything else (e.g. `perplexity/sonar`) | `chat/completions`, still usable by naming it |

`osJevModels()` returns `DecisionClient.KnownDecisionModels`: Jev (`typesafe/jev-1.14`, `~typesafe/jev-latest`, `typesafe/jev-1.13`), `microsoft/microsoft-decision-1`, `openai/gpt-6-luna-decisions`, `perplexity/pplx-decider-v1.2` and `…-flash-v1.2`, Cloudflare Clef (`clef`, `clef-flash`, `clef-omni`), `liquid/d1`, Upstage `solar-decide(-flash)`, Inception `mercury-decide`, Respan `span-01(-lite)`, `togethercomputer/tev1-4b-experimental`, `nace-ai/drex-v1.5`, `jaredpalmer/kev-4b`, and `subq-snap-preview`. The live list is `https://openrouter.ai/api/v1/models?output_modalities=decisions`. Only the ones named in [[Decision Models Benchmark]] were measured.

### 6.2 Choosing a model

From the 30-ticket run (details and caveats in [[Decision Models Benchmark]]):

| Goal | Model |
|---|---|
| Best accuracy | `subq-snap-preview` (30/30 intent, 30/30 anger, 24/30 exact urgency; ~$11 per 1000 calls) |
| Best value | `perplexity/pplx-decider-flash-v1.2` (0.26 s, $1.90 per 1000, 28/30 intent) |
| Accurate and cheap | `perplexity/pplx-decider-v1.2` ($3.80 per 1000; rate-limited) |
| Not worth it | `openai/gpt-6-luna-decisions` (22× flash), `microsoft/microsoft-decision-1` (weakest urgency) |

### 6.3 Images and Box go to TypeSafe, not OpenRouter

OpenRouter's decisions endpoint returns 200 for a request with an image but never shows the image to the model (identical answers and token counts with and without it, on Jev, Clef and Microsoft). So requests with `assetsJson` or `osJevBox` need `[OSSL] TypeSafeAPIKey` and go to `api.typesafe.ai/v1/systemone` with the model renamed (`typesafe/jev-1.14` → `jev-1.14.0`). With no key, or a non-Jev model, they return `ERROR:` before any request. The client checks what TypeSafe would reject: each image side 280-1568 px (read from the PNG/JPEG header), and every asset key must appear inside a state *value* or a question's `instructions`. `osJevBox` writes the key into its own question and takes exactly one image.

### 6.4 SubQ (`subq-snap-preview`)

Added 2026-10-17. Not on OpenRouter: the client sends `subq-*` / `subq/*` ids (the wire id has no prefix) to `https://api.subq.ai/v1/systemone` with `[OSSL] SubqAPIKey`, else `SUBQ_API_KEY`; `SubqURL` overrides the endpoint. `OSSL_Api` also publishes the key as `DecisionClient.ConfiguredSubqKey`, which is how `choose_best_tool` (`OPENROUTER_DECISION_MODEL=subq-snap-preview`) shares it. **Text only**: images and Box return `ERROR:` (its image contract is not implemented). It answers with `output_tokens: 0` and no `cost` field. Pricing: $0.05 per 1M input tokens, output free (subq.ai/pricing); public preview. Verified live against the API on 2026-10-17; pinned against a stub by `DecisionClientTests.SubqGoesStraightToSubqWithItsOwnKeyAndIsTextOnly`.

## 7. MCP and console

- MCP tool **`choose_best_tool(query, category)`** uses `DecisionClient` with `DefaultModel` (`pplx-decider-v1.2`) or `OPENROUTER_DECISION_MODEL`; see [[MCP Server & Live Diagnostics]].
- Console: `jev choose <query>` and `jev status` (decision-model key, model and catalog status).
- MCP tool **`zengine_run`** compiles and runs an LSL file on the simulator and returns everything the script said; it is how the benchmark was run.

## 8. Examples

Eight more examples, one per function family (`osJevNoul`, `osJevChoice(Ext)`, `osJevScore`, `osJevAsk`, `osJevAskExt` + `osJevModels`, SubQ, the ZEXT `->` bind, `osJevBox` with an image), were run on the live sim on 2026-10-17 and are listed with their output in [[OSSL Extension Examples - Complete Coverage]] section 2; the files are in `tasks/jev/`.

**Compare models on the same ticket (verified live 2026-10-17)**

```lsl
list MODELS = ["typesafe/jev-1.14", "subq-snap-preview", "perplexity/pplx-decider-flash-v1.2"];
string Q = """{"intent":{"type":"choice","instructions":"What does the customer want?","criteria":{"refund":"asks for money back","other":"anything else"}}}""";

default
{
    state_entry()
    {
        integer i;
        for (i = 0; i < llGetListLength(MODELS); ++i)
        {
            string m = llList2String(MODELS, i);
            float t0 = llGetTime();
            string r = osJevAskExt("My parcel arrived two weeks late. I want my money back.", Q, m, "");
            llOwnerSay(m + " " + (string)(llGetTime() - t0) + "s " + r);
        }
    }
}
```

**Retry on a rate limit (used by the benchmark)**

```lsl
string r = osJevAskExt(state, QUESTIONS, model, "");
integer tries = 0;
while (llSubStringIndex(r, "ERROR") == 0 && llSubStringIndex(r, "429") >= 0 && tries < 4)
{
    ++tries;
    llSleep(2.0 * tries);
    r = osJevAskExt(state, QUESTIONS, model, "");
}
```

**NPC that decides** - `scripts/npc_jev_greeter.lsl` classifies what visitors say with `osJevChoiceExtAsync` (bound with `->`) into a fixed list of actions, so whatever an avatar types the NPC can only do one of those actions.

## 9. Gotchas

- Declare counters with an initialiser inside loops (`integer ok = 0;`); an uninitialised one kept its value across iterations in ZEngine (observed while building the benchmark).
- `zengine_run` / `zengine run --wait` waits at most 60 s; a script calling the network 70 times sequentially does not fit. Split by model.
- A decision model's answers are probabilities, not truth: two models can disagree by 0.2-0.4 on anger or urgency of the same text. Compare on labelled data before choosing.
- `scripts/jev_tool_picker.py` is a separate TypeSafe client and was not migrated to `DecisionClient`.
