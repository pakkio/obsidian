---
tags:
  - craftsim
  - ossl
  - osocr
  - osplot
  - oshttp
  - oslex
  - osyacc
  - module
aliases:
  - osOcr
  - osPlot
  - osHTTP
  - osLex
  - osYacc
  - osFetchClean
---

# Module - osOcr, osPlot, osHTTP, osLex & osYacc

← Back to [[CraftSim]] | Related: [[OSSL Extensions (AI, ML, OsDb, OsQ)]], [[Module - osAI & osJev]], [[ZEXT Language Extensions]]

Verified against code (commit `9fea6a0332`, 2026-10-16).

This note covers document/vision extraction (`osOcr`), native chart rendering (`osPlot` / `ChartRenderModule`), HTTP client and HTML cleaning (`osHTTP*` / `osFetchClean*`), regex lexing and Earley parsing (`osLex` / `osYacc`), and per-prim logging (`osLog*`).

All implementations live in one partial class: `OpenSim/Region/ScriptEngine/Shared/Api/Implementation/OSSL_Api.cs` (14185 lines). Script-facing declarations are in `.../Api/Interface/IOSSL_Api.cs` and forwarded by `.../Api/Runtime/OSSL_Stub.cs`. Because the API is on the shared OSSL surface, the functions work from YEngine scripts as well as ZEngine (`CLAUDE.md:231`, `YEngineOsslBindingTest.cs`).

## Threat levels (from `CheckThreatLevel` calls)

| Function(s) | ThreatLevel | Source |
| :--- | :--- | :--- |
| `osOcr`, `osOcrAsync` | High | `OSSL_Api.cs:10158`, `:10233` |
| `osPlot`, `osPlotFace` | VeryLow | `OSSL_Api.cs:1082` |
| `osHTTP*`, `osFetchClean*` (sync and async) | Moderate | `OSSL_Api.cs:9726-9805` |
| `osLex`, `osYacc` (both overloads) | Low | `OSSL_Api.cs:11404`, `:11516`, `:11550` |
| `osLog*`, `osLogSetLevel` | High | `OSSL_Api.cs:5473-5541` |

## Async model (shared by osOcrAsync, osHTTP*Async, osFetchCleanAsync, osAI*Async)

- `StartAIRequest` / `StartGatedRequest` (`OSSL_Api.cs:6268` onward) run the work on `Task.Run` behind a process-wide semaphore `m_aiRequestGate` with `MaxConcurrentAIRequests = 8` (`:6253`).
- When all 8 slots are busy the function does not block: it returns the string `ERROR: too many in-flight requests (max 8), try again shortly` as the "key".
- Otherwise a random UUID key is returned immediately and `DeliverAIResult` posts a `dataserver(key, result)` event to the script (`:6305-6320`). The result is always a string; failures are strings starting `ERROR:`.
- In ZEngine scripts the `->` async-bind sugar can await that event inline (`tasks/async_sugar_sample.lsl:16`: `osCacheGetAsync("demo","greeting") -> value;`). The `: <timeout> else { ... }` suffix used in the previous version's examples was not verified (unverified) and is no longer used below.

---

## 1. `osOcr` - Image and PDF text extraction

### Signatures (`IOSSL_Api.cs:899-904`)
- `string osOcr(string source, string prompt)` - engine defaults to `"llm"`.
- `string osOcr(string source, string prompt, string engine)`
- `LSL_Key osOcrAsync(string source, string prompt)` / `LSL_Key osOcrAsync(string source, string prompt, string engine)` - result arrives via `dataserver`.

### Flow (`DoOcr`, `OSSL_Api.cs:10161-10216`)
1. Empty `source` returns `ERROR: source is empty`.
2. `source` starting with `http://` or `https://` is fetched by `FetchOcrBytesFromUrl` (`:9863`, shared `m_httpClient`, binary-safe). Anything else is treated as the name or UUID of an item in the calling prim's own task inventory (`FetchOcrBytesFromInventory`, with `SearchTaskInventoryForAnyAssetId` at `:9917` - not limited to notecards). Texture assets (JPEG2000) are decoded and re-encoded as PNG by `DecodeIfTextureAsset` (`:9971`) so both engines get a usable raster.
3. `engine` is trimmed and lower-cased; empty means `"llm"`. Anything other than `llm` / `tesseract` returns `ERROR: unsupported engine '<x>' (use llm or tesseract)` (`OcrImage`, `:10215`).
4. PDF detection (`IsPdf`): `ExtractPdfText` (`:10061`) reads the text layer with UglyToad PdfPig (`PackageReference PdfPig 0.1.16`, `Directory.Build.props:19`) via `ContentOrderTextExtractor`, **up to `MaxPdfPages = 200`** (`:10018`). If it yields nothing the result is `ERROR: PDF has no extractable text layer` (const `NoPdfTextLayerError`, `:10022`).
5. For that no-text-layer case `ExtractPdfImages(data, MaxPdfOcrImages = 10)` (`:10028`, `:10021`) pulls embedded page images (PNG, or raw JPEG) in page order - no native PDF rasterizer is used - and each image is OCR'd separately; texts are joined with `\n`. If there are no images the error is `<no text layer error> and no embedded images to OCR`. If every image fails the first error is returned.
6. Non-PDF data goes straight to `OcrImage`.

### Engines
- `"llm"` (`OcrViaLlm`, `:10099`): builds an OpenRouter multimodal `messages` array (text part + `image_url` data-URL, MIME sniffed by `SniffImageMimeType`) and calls `CallOpenRouter(model, messages, 2048)` directly - deliberately not through public `osAIChat`, so the threat-level audit trail says `osOcr`. Model is hard-coded `"openai/gpt-4o-mini"` in both `osOcr` and `osOcrAsync` (`:10160`, `:10234`) - there is no config key or parameter to change it. Default prompt when empty: `"Transcribe all text in this image."`. Needs `osAISetOpenRouterAPIKey()` first (`:8889`).
- `"tesseract"` (`OcrViaTesseract`, `:10120`): `Tesseract` NuGet 5.2.0 (`Directory.Build.props:17`), language hard-coded `"eng"`, `EngineMode.Default`, image loaded with `Pix.LoadFromMemory`. The `prompt` argument is ignored. Without `[OSSL] TesseractDataPath` it returns `ERROR: [OSSL] TesseractDataPath is not configured`.

### Configuration
| Key | Default | Where |
| :--- | :--- | :--- |
| `[OSSL] TesseractDataPath` | `""` (engine unavailable) | read at `OSSL_Api.cs:245`; a comment there notes missing/mismatched data silently gives empty or garbage text |
| `MaxOcrFetchBytes` | 25 MiB | **compile-time `const`** (`:9857`), not an ini key. Enforced on `Content-Length` and while streaming (`:9886`, `:9902`) |

`bin/preflight.sh:82-85` warns if `TesseractDataPath` is set but is not a directory. A similar startup warning list exists at `OSSL_Api.cs:8884-8889` (missing path / non-directory / missing OpenRouter key).

### Tests
`OSSL_ApiOcrTest.cs` (empty source, unknown inventory source, unsupported engine, tesseract with no data path, async error via dataserver, plus live tesseract and live LLM tests) and `tests/OpenSim.Tests/OsOcrPdfTests.cs`. Note the test named `TestPdfSourceReturnsNotYetAvailable` (line 109) predates the PDF implementation; its name is stale (body not re-read).

### Example
```lsl
default
{
    touch_start(integer n)
    {
        string pdfUrl = "https://example.com/invoices/invoice_2026_scan.pdf";
        string prompt = "Extract the invoice number, date, and line-item totals as JSON.";

        // Text-layer PDF -> text; scanned PDF -> up to 10 embedded images OCR'd.
        osOcrAsync(pdfUrl, prompt, "llm") -> string extracted;   // ZEngine async-bind sugar

        if (llGetSubString(extracted, 0, 5) == "ERROR:")
            osOcrAsync(pdfUrl, "", "tesseract") -> extracted;    // local fallback (prompt ignored)

        llOwnerSay("Extracted:\n" + extracted);
    }
}
```

---

## 2. `osPlot` and `ChartRenderModule` - chart textures

Files: `OpenSim/Region/CoreModules/Scripting/Plot/ChartRenderModule.cs` (635 lines), API side `OSSL_Api.cs:1075-1155`, tests `OSSL_ApiPlotTest.cs` (line chart, multi-series bar with legend, explicit x:y scatter, pie, empty series error, malformed pair error, per-face).

- `ChartRenderModule` is an `ISharedRegionModule` + `IDynamicTextureRender` registered for content-type `"chart"` (`:79`, registered in `RegionLoaded` `:139-145`). Modelled on `VectorRenderModule`: System.Drawing (GDI+; `libgdiplus` on Linux) draws onto a 32bpp bitmap, then `OpenJPEG.EncodeFromImage` produces JPEG2000 (`:369`). Rendering and disposal are done under locks because concurrent libgdiplus disposal has corrupted the heap. Output is marked **not reusable** so refreshed data is never served from the dynamic-texture cache.
- Known gap: on hosts without `libgdiplus`, the five `osPlot`/`ChartRenderModule` tests fail with `DllNotFoundException` (`CLAUDE.md:45`).

### Signatures (`IOSSL_Api.cs:138-140`)
- `string osPlot(string chartType, LSL_List series, string extraParams)` - same as `osPlotFace(..., -1)` (all sides).
- `string osPlotFace(string chartType, LSL_List series, string extraParams, int face)`.
- Return value: the **texture updater UUID** as string (not the texture asset UUID, per the doc comment above `osPlot`), or `ERROR: ...` (`series data is empty`, `every series must be a non-empty list of numbers or [x,y] pairs`, `every point in an [x,y]-style series must be a 2-element list`, `dynamic texture manager not available`).

### Series format (`PlotSeriesToWireFormat`, `:1106`)
There are **no series name strings** in the list. Accepted shapes:
- flat list of numbers `[1, 4, 9]` - one series, x = index;
- list of lists `[[1,2,3],[3,2,1]]` - several series;
- points as `[x, y]` 2-element lists inside a series.

Internally encoded as `x:y,x:y;...` (series separated by `;`, points by `,`, floats formatted `0.######` invariant). Series and slice names go in the `labels:` extra param.

### `extraParams` (`ParseExtraParams`, `ChartRenderModule.cs:184-250`)
The API prepends `type:<chartType lowercased>`. `chartType`: `line` (default and fallback for unknown values), `bar`, `scatter`, `pie` (draws only the first series, as slices).

| Key | Default | Notes |
| :--- | :--- | :--- |
| `width` / `height` | 512 / 384 | clamped to 32..2048 |
| `bgcolor` (`bgcolour`) | white | hex `RRGGBB` (forced opaque) or .NET known colour name; bad value keeps fallback |
| `lossless` | false | `true` for lossless JPEG2000 |
| `title`, `xlabel`, `ylabel` | empty | `_` is converted to a space (params are comma-separated) |
| `legend` | **true** | only the literal `true` enables it |
| `labels` | none | pipe-separated names per series or slice |
| `colors` (`colours`) | built-in palette | pipe-separated hex/name list |

Format is `key:value` pairs separated by commas; pairs without `:` are ignored.

### Example (corrected)
```lsl
list g_fps = [];

default
{
    state_entry() { llSetTimerEvent(2.0); }

    timer()
    {
        g_fps += llGetRegionFPS();
        if (llGetListLength(g_fps) > 20)
            g_fps = llList2List(g_fps, -20, -1);   // sliding window

        // one series = flat list of numbers; names via labels:
        osPlotFace("line", g_fps,
            "width:512,height:384,bgcolor:black,title:Live_SimFPS,xlabel:Tick,ylabel:FPS,labels:SimFPS", 0);

        // pie: first series only, labels name the slices
        osPlotFace("pie", [[ 40.0, 60.0 ]], "title:Memory,labels:Used|Free", 1);
    }
}
```
The previous doc's `osPlotFace("line", ["SimFPS", list], ...)` form is not accepted by `PlotSeriesToWireFormat`.

---

## 3. `osHTTP*` and `osFetchClean*`

Code: `#region osHTTP Functions`, `OSSL_Api.cs:9566-9850`. A single static `HttpClient m_httpClient` is shared (also reused by the OCR fetch helper). No custom timeout is set in this code, so the .NET default applies (unverified).

### Signatures (`IOSSL_Api.cs:654-666, 896-897`)
| Function | Signature | Notes |
| :--- | :--- | :--- |
| `osHTTPGet` | `string osHTTPGet(string url)` and `(string url, LSL_List headers)` | returns body, even for non-2xx codes |
| `osHTTPPost` | `string osHTTPPost(string url, string body [, LSL_List headers])` | |
| `osHTTPRequest` | `string osHTTPRequest(string method, string url, string body, LSL_List headers)` | any verb; body only attached for POST / PUT / PATCH / DELETE (exact upper-case compare) |
| `osHTTPRequestExt` | `LSL_List osHTTPRequestExt(method, url, body, headers)` | `[statusCode (int), body, contentType]`; bad URL gives `[0,"",""]`, exception gives `[0,"ERROR: ...",""]`; contentType is the media type only |
| `osHTTPGetAsync`, `osHTTPPostAsync` | key-returning, optional headers | result via `dataserver` |
| `osFetchClean` / `osFetchCleanAsync` | `(string url [, LSL_List headers])` | GET then `CleanHtml` |

There is **no** async variant of `osHTTPRequest` / `osHTTPRequestExt`.

### Behaviour details
- URL must start with `http://` or `https://`, else `ERROR: URL must start with http:// or https://`; empty URL gives `ERROR: URL is empty`.
- Headers: strided `[name, value, ...]` (`ParseHttpHeaders`, `:9575`); empty keys/values skipped; `Host` and `Content-Length` are ignored; headers added with `TryAddWithoutValidation`.
- Body content type defaults to `text/plain; charset=utf-8`; a `Content-Type` entry in headers overrides it.
- `CleanHtml` (`:9818`): removes `<script>` / `<style>` blocks, turns `<br|p|div|tr|li|h1-6>` into newlines, strips all other tags, `WebUtility.HtmlDecode`, collapses whitespace per line and drops blank lines. `FetchClean` passes empty or `ERROR:` bodies through unchanged.
- Tests: `OSSL_ApiLexYaccTest.cs` also holds `TestOsFetchCleanSanitization` and a live Wikipedia fetch test. Demos: `tasks/oshttp_demo.lsl`, `tasks/test_fetch_clean.lsl`, `tasks/test_http_client.lsl`.

### Example
```lsl
summarize_webpage(string url)
{
    osFetchCleanAsync(url, ["User-Agent", "CraftSim-Bot/1.1"]) -> string cleanText;
    if (llGetSubString(cleanText, 0, 5) == "ERROR:") { llOwnerSay("Fetch failed: " + cleanText); return; }

    string excerpt = llGetSubString(cleanText, 0, 1999);
    osAICompletionAsync("openai/gpt-4o-mini", "Summarize in 3 bullets:\n\n" + excerpt, 250) -> string summary;
    llSay(0, "Summary of " + url + ":\n" + summary);
}
```
The `osAICompletionAsync` signature is covered in [[Module - osAI & osJev]] and was not re-checked here.

---

## 4. `osLex` and `osYacc`

Code: `OSSL_Api.cs:11395-11736`; declarations `IOSSL_Api.cs:889-893`; tests `OSSL_ApiLexYaccTest.cs` (`TestOsLexTokenization`, `TestOsLexExpressionWithStar`, `TestOsYaccGrammarParsing`); demo `scripts/oslex_yacc_demo.lsl`.

### `osLex(string input, LSL_List rules)` (`:11402`)
- `rules` is strided `[Name, Regex, Name, Regex, ...]`. Empty input or fewer than 2 rule entries returns an empty list.
- Each pattern is compiled as `^(<pattern>)` with `RegexOptions.Compiled` and cached in `m_lexRuleCache` (static `ConcurrentDictionary`, key = all rule strings joined with a control-character separator; never evicted).
- **Longest match wins** (ties go to the earlier rule), not first match (`:11443-11452`). Zero-length matches are ignored.
- A rule name starting with `SKIP`, `IGNORE` (case-insensitive) or `_` is consumed but emits no token.
- An unmatched character emits `["ERROR", "<char>", line, col]` and advances one char. An invalid regex shouts `osLex invalid regex pattern for '<name>': ...` to the owner and returns an empty list.
- Output: stride-4 list `[name, value, line (integer), col (integer), ...]`, 1-based; `\n` increments the line and resets the column.

### `osYacc` (`:11514`, `:11548`)
- `osYacc(string input, LSL_List lexRules, string grammarSpec, string startSymbol)` calls `osLex` then the token overload. It does **not** abort on `ERROR` tokens; an ERROR token simply fails to match and the parse fails.
- `osYacc(LSL_List tokens, string grammarSpec, string startSymbol)` accepts either stride-4 lexer output (detected when `tokens.Length >= 4` and `Data[2] is LSL_Integer`) or a plain list of terminal strings.
- Grammar text is split on `;`, `\n`, `\r`; lines starting with `#` or `//` are comments; separator `->` or `::=`; alternatives split on `|`; symbols split on whitespace. Terminals match a token by its type name, by its literal value, or by either with surrounding `'` / `"` stripped. A symbol is a non-terminal only if it appears as an LHS.
- Algorithm: Earley chart (`OsEarleyProd`, `OsEarleyItem`, predictor / completer / scanner loops `:11629-11700`) with a synthetic `_ROOT_ -> start` rule; left recursion works. The tree is built as items complete (`BuildAstList`, `:11724`).
- Result: nested lists `[LHS, child, child, ...]`; children are sub-lists (non-terminals) or token value strings (terminals). Errors: `["ERROR","Invalid parameters supplied to osYacc."]`, `["ERROR","No valid grammar rules found in osYacc specification."]`, `["ERROR","osYacc syntax error: Failed to parse token stream at input token length N."]`.
- Known gaps: no `epsilon` / `ε` keyword exists in the source (the previous doc claimed one). An empty alternative would give an empty RHS, but nullable rules are not special-cased in the Earley loop (unverified). Ambiguous grammars return a single tree. No explicit input-size cap (unverified).

### Example (from `OSSL_ApiLexYaccTest.cs` / `scripts/oslex_yacc_demo.lsl`)
```lsl
list lexRules = [ "SKIP_WS", "\\s+", "IDENT", "[a-zA-Z_][a-zA-Z0-9_]*", "INT", "[0-9]+",
                  "PLUS", "\\+", "STAR", "\\*", "LPAREN", "\\(", "RPAREN", "\\)" ];
string grammar = "Expr -> Expr PLUS Term | Term ; Term -> Term STAR Factor | Factor ; Factor -> INT | IDENT | LPAREN Expr RPAREN ;";
list ast = osYacc("5 + 10 * (2 + x)", lexRules, grammar, "Expr");
if (llList2String(ast, 0) == "ERROR") llOwnerSay("parse error: " + llList2String(ast, 1));
else llOwnerSay("AST: " + llList2CSV(ast));
```

---

## 5. `osLog*` - per-prim leveled logging

Code: `OSSL_Api.cs:5389-5545`; demo `tasks/oslog_demo.lsl`.

- API (`IOSSL_Api.cs:606-612`): `osLogSetLevel(string)`, `osLog(string level, string msg)`, `osLogTrace/Debug/Info/Warn/Error(string)`.
- Valid levels: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `NONE`. Invalid level resets to `INFO` with a warning. Default threshold when never set: `INFO`. Stored per **prim UUID** in a static `m_logLevels` dictionary.
- `EmitLog`: message truncated to 1023 chars; log4net line `[osLog][LEVEL] <prim name> (<uuid>): msg` (TRACE and DEBUG both go to `m_log.Debug`); **and** a `DEBUG_CHANNEL` shout (`World.SimChat` plus `IWorldComm.DeliverMessage`), so the owner sees it in-world.
- Server console: `log browse [<level>] [from <dt>] [to <dt>] [latest <N> min|hour] [<count>] [<keyword>]` (`OpenSim/Framework/Servers/ServerBase.cs:340`). Levels ALL/TRACE/DEBUG/INFO/WARN/ERROR/FATAL, default INFO, count default 20 max 200, keyword is a substring or `~regex`. Only events that pass the root logger level are captured (ring buffer in `LogBuffer.cs`).

---

## Corrections vs previous doc
- `MaxOcrFetchBytes` is a hard-coded const (25 MiB), not an `[OSSL]` ini setting; the OCR model (`openai/gpt-4o-mini`) is fixed in code.
- `osOcr` also accepts in-world inventory item names / UUIDs (including textures, which are decoded), not just URLs; `prompt` is ignored by tesseract; language is fixed to `eng`.
- `osPlot`: series lists contain numbers or `[x,y]` pairs only (no name strings - use `labels:`); default size is 512x384, not 512x512; `legend` defaults to true; `_` stands for spaces in title/labels; pie uses only the first series; `lossless`, `labels`, `colors` keys were undocumented. ThreatLevel is VeryLow.
- `osHTTPGet`/`Post` also have header-less overloads; there are no async `osHTTPRequest*` variants.
- `osLex` uses longest-match; `osYacc` does not abort on lexer ERROR tokens; the `epsilon` / `ε` keyword does not exist; line ranges `11395-11473` / `11509-11736` were approximate.
- `osLog` is ThreatLevel High, supports level `NONE`, truncates to 1023 chars and also shouts on `DEBUG_CHANNEL`.
- The old telemetry example misused the `osPlotFace` series argument.
