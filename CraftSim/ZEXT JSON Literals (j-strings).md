---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - json
  - strings
aliases:
  - j-strings
  - j"..."
  - JSON-like strings
---

# ZEXT JSON literals: `j"..."`, `j'...'`, `j<< ID`

← Back to [[ZEXT Syntax Sugar Index]] | Reference: [[ZEXT Language Extensions]] | Related: [[ZEXT Heredocs]], [[ZEXT Sugar - Strings, Operators and States (annotated)]] | Output: [[ZEXT Desugar - Stock LSL Output]] | Checked by: [[ZEXT Desugar Verification Report]]

> [!SUMMARY]
> A string literal prefixed with **`j`** is an ordinary string (you get exactly the characters you wrote) that the **compiler also checks is valid, strict JSON**. A typo becomes a compile error with the position, instead of a mysterious `JSON_INVALID` at run time. The value feeds `llJsonGetValue` / `llJsonSetValue` / `osJson*` unchanged. ZEXT only.

Source: `ZLSLTokenizer.cs` (`MarkJson`, the prefix scan in `Next()`), `ZLSLParser.cs` (reports `Token.LexError`). Example: `tasks/desugar_suite/sugar_04_json_strings.lsl`. Tests: `ZEnginePythonStringsTest.TestJson*`, the suite run in [[ZEXT Desugar Verification Report]].

---

## 1. The forms

| Written | Escapes | Notes |
| :--- | :--- | :--- |
| `j'{"a": 1}'` | processed like any string | **the natural form**: JSON's own double quotes need no escaping |
| `j"{\"a\": 1}"` | processed | write `\"` for each JSON quote; `\\` for a JSON backslash needs `\\\\` |
| `j'''…'''` / `j"""…"""` | processed | triple quotes may span lines and contain the quote character |
| `jr'{"p": "C:\\temp"}'` | **raw** | backslashes are kept verbatim, so a JSON `\\` stays `\\` |
| `j<< EOT` … `EOT` | none (raw) | heredoc form, see [[ZEXT Heredocs]]; the body is checked as JSON |

- The prefix is a contiguous run of prefix letters directly before the quote, case-insensitive: `j`, `jr`, `rj`, `s` and `f` exist for the other string forms. **`j` cannot be combined with `f`** (an f-string is built at run time, not a literal): that is a lexer error.
- The checker is `System.Text.Json` in its strict mode: no comments, no trailing commas, no single-quoted JSON strings, numbers as JSON numbers. Any valid JSON **value** is accepted (an object, an array, `3`, `"x"`, `true`, `null`), not only objects.
- The text is **not minified or normalised**: whitespace and key order are exactly as written.
- Adjacent literals concatenate as usual, but **each literal piece is validated on its own**, so `j'{"a":' j'1}'` fails (the first piece is not complete JSON). Build long JSON from one literal or from `llJsonSetValue`.

## 2. Errors

These are the exact messages (captured from the compiler); the caret points at the literal:

```text
default { state_entry() { string s = j'{"a": }'; } }
(1,38) Error: invalid JSON in literal: '}' is an invalid start of a value. LineNumber: 0 | BytePositionInLine: 6.

default { state_entry() { string s = j'[1, 2,]'; } }
(1,38) Error: invalid JSON in literal: The JSON array contains a trailing comma at the end which is not supported in this mode. Change the reader options. LineNumber: 0 | BytePositionInLine: 6.

string s = j<< E          // body: {oops}
(1,38) Error: invalid JSON in literal: 'o' is an invalid start of a property name. Expected a '"'. LineNumber: 0 | BytePositionInLine: 1.

string s = jf"x";
(1,38) Error: unexpected 'j-prefix cannot be combined with f-string'
```

The text after `invalid JSON in literal:` is the .NET JSON reader's own message, so its wording can change with the .NET version; `LineNumber` / `BytePositionInLine` are relative to the literal's text, while the `(line,col)` prefix is the literal's position in your file.

With `[ZEngine] ZEXT = false` the same literal gets **two** errors, the gate first and then the JSON check:

```text
(1,38) Error: a JSON literal (j"..." / j'...') is a ZEngine extension, not standard LSL (set [ZEngine] ZEXT = true to allow it)
(1,38) Error: invalid JSON in literal: …
(1,38) Error: a JSON heredoc (j<< ID ... ID) is a ZEngine extension, not standard LSL (set [ZEngine] ZEXT = true to allow it)
```

## 3. The annotated example

```lsl
#indent
// =============================================================================================
// sugar_04_json_strings.lsl  -  j"..." JSON-like string literals
//
// A string literal prefixed with `j` is ordinary text (you get the characters back unchanged) that the
// COMPILER also checks is valid, strict JSON.  A typo is a compile error with the parser's message
// ("invalid JSON in literal: ...") instead of a mysterious JSON_INVALID at run time.
//
// Every string form takes the prefix:
//     j'...'      single quotes: the natural choice, JSON's own double quotes need no escaping
//     j"..."      double quotes: escapes are processed like any string, so write \" for JSON quotes
//     j"""..."""  triple quotes: may span lines
//     jr'...'     raw: backslashes are kept as written (so a JSON \\ stays \\)
//     j<< ID      heredoc form: see sugar_05_heredocs.lsl
// Not allowed together with f"" (an f-string is not a literal).  ZEXT only.
//
// The value is a plain string, so it feeds llJsonGetValue / llJsonSetValue / osJson* unchanged.
// =============================================================================================

default:
    touch_start(integer n):
        string a = j'{"name": "Ann", "tags": ["x", "y"], "n": 3}'
        string b = j"{\"quote\": \"he said \\\"hi\\\"\"}"
        string c = jr'{"path": "C:\\temp"}'
        string d = j"""{
  "multi": "line",
  "ok": true
}"""             // spans lines; opened right after the quotes
        string e = j<< EOT
{ "heredoc": [1, 2, 3] }
EOT
        llOwnerSay(llJsonGetValue(a, ["name"]) + " " + llJsonGetValue(a, ["tags", 1]) + " " + llJsonGetValue(a, ["n"]))
        llOwnerSay(llJsonGetValue(b, ["quote"]))
        llOwnerSay(llJsonGetValue(c, ["path"]))
        llOwnerSay(llJsonGetValue(d, ["multi"]) + " " + (string)(llJsonGetValue(d, ["ok"]) == JSON_TRUE))   // JSON true reads back as the JSON_TRUE sentinel
        llOwnerSay(llJsonGetValue(e, ["heredoc", 2]))
        llOwnerSay("len a=" + (string)llStringLength(a) + " c=" + c)
```

Output (as written = after desugaring):

```text
Ann y 3
he said "hi"
C:\temp
line 1
3
len a=43 c={"path": "C:\\temp"}
```

## 4. After desugaring

A j-string is just a string value to every later pass, so desugar prints a normal quoted literal with the JSON quotes escaped. Nothing about the `j` survives, and the result needs no ZEXT:

```lsl
    touch_start(integer n)
    {
        string a = "{\"name\": \"Ann\", \"tags\": [\"x\", \"y\"], \"n\": 3}";
        string b = "{\"quote\": \"he said \\\"hi\\\"\"}";
        string c = "{\"path\": \"C:\\\\temp\"}";
        string d = "{\n  \"multi\": \"line\",\n  \"ok\": true\n}";
        string e = "{ \"heredoc\": [1, 2, 3] }";
        llOwnerSay(((((llJsonGetValue(a, ["name"]) + " ") + llJsonGetValue(a, ["tags", 1])) + " ") + llJsonGetValue(a, ["n"])));
        llOwnerSay(llJsonGetValue(b, ["quote"]));
        llOwnerSay(llJsonGetValue(c, ["path"]));
        llOwnerSay(((llJsonGetValue(d, ["multi"]) + " ") + ((string)(llJsonGetValue(d, ["ok"]) == JSON_TRUE))));
        llOwnerSay(llJsonGetValue(e, ["heredoc", 2]));
        llOwnerSay(((("len a=" + ((string)llStringLength(a))) + " c=") + c));
    }
```

## 5. Gotchas

1. **Validation is per literal, at compile time only.** A string you build at run time is not checked; `j'...'` is not a type, the value is a plain `string`.
2. **`j"…"` escapes are *string* escapes first.** To put a JSON `\"` inside a double-quoted j-string you need `\\\"` (backslash, backslash, backslash, quote). Prefer `j'…'`, or `jr'…'` when the JSON has backslashes.
3. **A heredoc body is raw**, so `j<<` is the easiest way to paste a long JSON document: no escaping at all.
4. **It is a compile-time check, not a schema.** `j'{"a": 1}'` and `j'{"b": 2}'` are both fine; use your own checks for shape.
5. **Not related to `JSON_*` constants or the `[JsonStore]` module.** `osJson*` still needs `[JsonStore] Enabled = True`; `llJson*` works everywhere.

See also: [[ZEXT Heredocs]], [[ZEXT Syntax Sugar Index]].
