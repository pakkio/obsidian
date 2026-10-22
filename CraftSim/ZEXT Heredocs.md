---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - strings
  - heredoc
aliases:
  - heredoc
  - "<< END"
  - here-document
---

# ZEXT Heredocs: `<< ID … ID`

← Back to [[ZEXT Syntax Sugar Index]] | Reference: [[ZEXT Language Extensions]] | Related: [[ZEXT JSON Literals (j-strings)]], [[ZEXT Braceless Indent Mode]] | Output: [[ZEXT Desugar - Stock LSL Output]] | Checked by: [[ZEXT Desugar Verification Report]]

> [!SUMMARY]
> `<< ID` as the **last thing on a line** starts a here-document: every following line, **verbatim**, up to the first line that contains only `ID`, is the value of the string expression. No escapes, no interpolation, no comment or preprocessor processing inside. ZEXT only.

Source: `ZLSLTokenizer.TryReadHeredoc` (the lexer) and `ZIncludeResolver.ScanStringState` / `HasHeredocTerminator` (so the preprocessor leaves the body alone). Example: `tasks/desugar_suite/sugar_05_heredocs.lsl`.

---

## 1. Rules

| Rule | Detail |
| :--- | :--- |
| **Opener** | `<<`, optional blanks, the **delimiter**, optional blanks, then the **end of the line**. Anything after the delimiter on that line makes it *not* a heredoc. |
| **Delimiter** | any run of non-space characters: `EOT`, `END`, `:pippo:`, `---`, `@@END@@`. Case-sensitive. |
| **Terminator** | the first later line whose text, **trimmed of blanks**, equals the delimiter. It may be indented and have trailing spaces. Nothing else may be on that line (`END;` does not terminate). |
| **Value** | everything between the opener line and the terminator line, **without** the last newline (and without a trailing `\r`). Inner indentation is kept; there is no de-indenting. |
| **Raw** | backslash is an ordinary character (`\n` stays two characters), quotes need no escaping, `{` is not interpolation. `//`, `/* */` and a line starting with `#define` inside the body are **just text**: the comment stripper and the preprocessor both skip it. |
| **Still the shift operator** | `<<` is a heredoc only if a terminator line exists. `x << 4` at the end of a line with no later line consisting of `4` is a shift. `<<=` is never a heredoc. |
| **Line numbers** | the body's lines are counted, so errors after the heredoc point at the right line. |
| **JSON** | `j<< ID` additionally validates the body as JSON ([[ZEXT JSON Literals (j-strings)]]). |
| **ZEXT gate** | `a heredoc string (<< ID ... ID)` / `a JSON heredoc (j<< ID ... ID)` is a ZEngine extension. |

> [!INFO] Not the console heredoc
> `zengine desugar --heredoc` (a console option for typing a script in) is unrelated; this page is about the language syntax.

## 2. Where it can appear

It is a normal string expression: an initialiser, an argument, one operand of `+`. After the terminator line the statement simply continues, so in brace syntax the `;` goes on its own line (or after the closing `)` of an enclosing call):

```lsl
string banner = << END
Report v1
END
;
```

In a braceless file the newline after `END` ends the statement ([[ZEXT Braceless Indent Mode]]).

## 3. The annotated example

```lsl
#indent
// =============================================================================================
// sugar_05_heredocs.lsl  -  << ID ... ID  here-documents
//
// `<< ID` as the LAST thing on a line starts a heredoc: every following line, verbatim, up to the
// first line that contains only ID, is the string's value.
//   * the body is raw: no escape processing, no f-string interpolation, quotes and backslashes are
//     ordinary characters, and `//`, `/* */` and `#define` inside it are just text (the comment
//     stripper and the preprocessor both skip it);
//   * the newline before the terminator is not part of the value; indentation inside is kept;
//   * the delimiter is any run of non-space characters - EOT, END, :pippo:, --- - and the terminator
//     line may be indented or have trailing blanks (it is trimmed before comparing);
//   * `j<< ID` additionally checks the body as JSON (see sugar_04_json_strings.lsl);
//   * `<<` stays the shift operator unless the delimiter ends its line AND a terminator line follows
//     later, so `x << n` at the end of a line is never mistaken for a heredoc;
//   * it is a normal string expression: `<< END` can be concatenated, passed to a function, etc.
// ZEXT only.  (Not related to `zengine desugar --heredoc`, which is console input.)
// =============================================================================================

default:
    touch_start(integer n):
        string plain = << END
Hello "world"  \n stays as backslash-n
  leading spaces are kept
#define NOT_A_DIRECTIVE 1   // and this is not a comment
END
        string odd = << :pippo:
:: colons and --- dashes ::
:pippo:
        string dash = << ---
x
---
        string ind = << STOP
body line
    STOP
        integer sh = 1 << 4                 // a shift: no line consisting of just "4" follows
        integer sh2 = 1 <<
            3
        llOwnerSay(plain)
        llOwnerSay("[" + odd + "][" + dash + "][" + ind + "]")
        llOwnerSay("len=" + (string)llStringLength(plain) + " sh=" + (string)sh + " sh2=" + (string)sh2)
```

Output (identical after desugaring). Note the literal `\n`, the `#define` line and the `//` text survive:

```text
Hello "world"  \n stays as backslash-n
  leading spaces are kept
#define NOT_A_DIRECTIVE 1   // and this is not a comment
[:: colons and --- dashes ::][x][body line]
len=121 sh=16 sh2=8
```

## 4. After desugaring

The body becomes an ordinary quoted string with `"` escaped and real newlines written as `\n`; the shifts stay shifts:

```lsl
    touch_start(integer n)
    {
        string plain = "Hello \"world\"  \\n stays as backslash-n\n  leading spaces are kept\n#define NOT_A_DIRECTIVE 1   // and this is not a comment";
        string odd = ":: colons and --- dashes ::";
        string dash = "x";
        string ind = "body line";
        integer sh = (1 << 4);
        integer sh2 = (1 << 3);
        llOwnerSay(plain);
        llOwnerSay((((((("[" + odd) + "][") + dash) + "][") + ind) + "]"));
        llOwnerSay(((((("len=" + ((string)llStringLength(plain))) + " sh=") + ((string)sh)) + " sh2=") + ((string)sh2)));
    }
```

## 5. Gotchas

1. **Trailing text after the delimiter** (`<< END // comment`) disables the heredoc.
2. **The delimiter must not appear as a line of its own in the body**, obviously; pick one that cannot (`---` is a poor choice for Markdown).
3. **`x << n` at a line end is only safe while no later line is exactly `n`.** If you write a shift whose right operand happens to be a lone word on a later line, rename it.
4. **Braces and semicolons follow the normal rules** of the surrounding syntax; the heredoc itself is one token.
5. **Edge**: a body that must end in a newline needs an empty line before the terminator (the last newline is dropped).

See also: [[ZEXT Syntax Sugar Index]].
