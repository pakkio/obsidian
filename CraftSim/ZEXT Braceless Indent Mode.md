---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - indent
  - braceless
aliases:
  - "#indent"
  - braceless LSL
  - Python-style blocks
---

# ZEXT Braceless Indent Mode (`#indent`)

← Back to [[ZEXT Syntax Sugar Index]] | Reference: [[ZEXT Language Extensions]] | Engine: [[ZEngine]] | Output: [[ZEXT Desugar - Stock LSL Output]] | Checked by: [[ZEXT Desugar Verification Report]]

> [!SUMMARY] In one paragraph
> Put `#indent` on a line of its own and the whole file may be written Python-style: **a line that ends in `:` opens a block, a dedent closes it, a newline ends a statement**, and the parentheses around conditions are optional. There are no `{ }` and no `;`. Nothing downstream knows: before parsing, a token pass turns the indentation back into braces and semicolons, so the compiler, the type checker and `zengine desugar` all see ordinary LSL. ZEXT only (`[ZEngine] ZEXT = false` rejects it).

Source: `OpenSim/Region/ScriptEngine/ZEngine/ZIndentTransform.cs` (`ApplyIndent`), hooked into `ZLSLTokenizer.Tokenize`. Runnable example: `tasks/desugar_suite/sugar_01_indent_braceless.lsl` (annotated below). Tests: `ZSugarTest.TestIndent*`, `ZDesugarTest.TestIndentModeDesugars`, and the suite run described in [[ZEXT Desugar Verification Report]].

---

## 1. Turning it on

```lsl
#indent
// the rest of the file is brace-less
```

- `#indent` must be **alone on its line** (leading blanks and a trailing `// comment` are fine). It is recognised by the tokenizer, not the preprocessor, so it can sit after `#define` lines or comments, but it switches the **whole file**, not "from here on".
- With `ZEXT = false` the include/macro pass reports `the preprocessor ('#indent') is a ZEngine extension…` and the parser reports `indent mode (#indent) is a ZEngine extension, not standard LSL (set [ZEngine] ZEXT = true to allow it)`.
- `#define`, `#include`, `#if` still work (they run before tokenising, line by line).
- Explicit `{ }` and `;` stay legal inside an indented file, so the two styles can be mixed (a closure body is the common case, see [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]]).

---

## 2. The rules

| Rule | Detail |
| :--- | :--- |
| **Block opener** | A logical line whose **last token is `:`** is a *header*. The `:` is dropped and a `{` is emitted; the lines indented deeper than the header are its body; a dedent emits the `}`. |
| **Not a header** | `case x:` and a `default:` that is *inside* something (a `switch`). Those are labels; their statements are indented under them but open no block. A top-level `default:` and `state name:` **are** headers. |
| **Statement end** | At the start of every logical line, if the previous token is not `;` `{` `}` `:` a `;` is inserted. |
| **Continuation** | A line does **not** end the statement when it ends with a binary/unary operator, `=` or a compound assignment, `,`, `.`, `?` or `!`, or while `(` / `[` are still open. The next physical line (any indentation) continues it. |
| **Comparison signs** | `<` and `>` are deliberately **not** continuation operators: `vector v = <1,2,3>` legitimately ends a statement with `>`. |
| **Conditions** | `if` `elif` `while` `until` `unless` `for` `foreach` `switch` take their condition without parentheses; if it is already one parenthesised group it is left alone, otherwise `( )` is added. |
| **`elif`** | `else if`. |
| **`unless c:`** | `if (!(c))`. |
| **`until c:`** | `while (!(c))`. |
| **`do:`** | `do {`. The loop's closing `while cond` is the next line **at the same indentation, with no colon** (it gets its parentheses and `;`). |
| **`else:`** | needs the colon (`else:`), exactly like the others. `else if cond:` also works. |
| **Events, functions, states** | `touch_start(integer n):`, `string f(integer a):`, `def f(a) -> t:`, `state name:`, `default:`, `on event(...):` are all just headers. |
| **Inline timers** | `timer -> 0.5:` and `timer "name" -> 2.0:` keep their colon (the brace syntax needs it) and then get `{`. |
| **Empty body** | a header with no deeper line after it becomes `{ }`. |
| **`break` `continue` `switch`** | these are *contextual* keywords the lexer only recognises before `;` or `(`; the pass promotes them when a line is just `break` / `continue` or starts a `switch … :` header. |
| **Blank lines / comments** | ignored (comments are stripped by the tokenizer, so they take no part in indentation). |
| **Indentation unit** | the **column of the first token** of a line. A tab counts as one column, so mix tabs and spaces at your own risk; keep one style. |
| **Errors** | `inconsistent indentation` (a dedent that matches no open level), `missing condition before ':'`, `unexpected text after 'else'` / `'do'`. They are reported with the line and column. |

### What the pass produces

```
  integer total = 0            →   integer total = 0 ;
  for i = 1; i <= n; i++:      →   for ( i = 1 ; i <= n ; i ++ ) {
      total += i               →       total += i ;
  return total                 →   } return total ;
```

(Positions are kept: every inserted token takes the line of the token it follows, so compile errors point at the right source line.)

---

## 3. The annotated example

Every construct, no braces, no semicolons. This is the real file; `ZDesugarTest.TestDesugarSuiteIsEquivalent` runs it as written and as desugared and compares what the prim says.

```lsl
#indent
// =============================================================================================
// sugar_01_indent_braceless.lsl  -  the whole control-flow vocabulary, with no { } and no ;
//
// `#indent` (a line of its own, ZEXT only) switches the file to Python-style blocks:
//   * a line that ENDS WITH ':' opens a block; the lines indented deeper than it are its body;
//     a dedent closes it.  (`case x:` / a nested `default:` are labels, not block openers.)
//   * a newline ends a statement - unless the line ends with an operator or a comma, or is
//     inside ( ) / [ ], in which case the statement simply continues on the next line.
//   * parentheses around the condition of if/elif/while/for/foreach/switch/unless/until are optional.
//   * `else if` can be written `elif`.  `unless c:` is `if (!(c))`, `until c:` is `while (!(c))`.
// Explicit { } ; are still accepted inside an indented file, so the two styles can be mixed.
// Everything is rewritten to ordinary tokens before parsing: the compiler never sees the indentation.
// =============================================================================================

integer g_count

// A function: `type name(params):`.  No { } and no `return x;` semicolon.
string grade(integer score):
    if score >= 90:
        return "A"
    elif score >= 75:
        return "B"
    else:
        return "C"

integer sum_to(integer n):
    integer total = 0
    integer i
    for i = 1; i <= n; i++:                 // the for-header keeps its own semicolons
        total += i
    return total

default:                                    // `default:` and `state name:` open a state
    state_entry():                          // an event handler is just `name(params):`
        g_count = 0

    touch_start(integer num):
        // ---- while with continue / break -------------------------------------------------
        integer i = 0
        while i < 10:
            i++
            if i % 2 == 0:
                continue
            if i > 7:
                break
            g_count += i
        llOwnerSay("while: " + (string)g_count + " i=" + (string)i)

        // ---- do / while: the closing `while` line has no colon -----------------------------
        integer k = 0
        do:
            k += 3
        while k < 10
        llOwnerSay("do-while k=" + (string)k)

        // ---- foreach, unless, until ------------------------------------------------------
        list names = ["ann", "bob", "cy"]
        foreach string nm in names:
            unless nm == "bob":
                llOwnerSay("name " + nm)
        integer n = 5
        until n <= 2:
            n--
        llOwnerSay("until n=" + (string)n)

        // ---- switch: case bodies are indented under the case line; fall-through as in C ---
        integer sel
        for sel = 0; sel < 4; sel++:
            switch sel:
                case 0:
                    llOwnerSay("zero")
                case 1:
                    llOwnerSay("one (also reached from zero)")
                    break
                case 2..3:
                    llOwnerSay("two or three")
                    break
                default:
                    llOwnerSay("other")

        // ---- continuation lines ------------------------------------------------------------
        integer big = 1 +
            2 +
            3
        list l = [1, 2,
                  3, 4]
        llOwnerSay(grade(95) + grade(80) + grade(10) + " " + (string)sum_to(4) + " " + (string)big + " " + (string)llGetListLength(l))

        // ---- jump / label (a label line is just `@name`) -----------------------------------
        integer tries = 0
        @again
        tries++
        if tries < 3:
            jump again
        llOwnerSay("tries=" + (string)tries)

        state second                        // a state change is an ordinary statement

state second:
    state_entry():
        llOwnerSay("now in state second")
```

Output (captured from the run; the desugared script prints exactly the same):

```text
while: 16 i=9
do-while k=12
name ann
name cy
until n=2
zero
one (also reached from zero)
one (also reached from zero)
two or three
two or three
ABC 10 6 4
tries=3
now in state second
```

---

## 4. What the compiler actually sees

Same file after the token pass, and after `zengine desugar` (stock LSL). The functions:

```lsl
string grade(integer score)
{
    if ((score >= 90))
    {
        return "A";
    }
    else if ((score >= 75))
    {
        return "B";
    }
    else
    {
        return "C";
    }
}
```

```lsl
integer sum_to(integer n)
{
    integer total = 0;
    integer i;
    for ((i = 1); (i <= n); (i++))
    {
        (total += i);
    }
    return total;
}
```

`switch` and `case 2..3:` become a jump-dispatch chain with labels in source order, so C-style fall-through is preserved; `break` is a jump to the end label:

```lsl
    touch_start(integer num)
    {
        integer i = 0;
        while ((i < 10))
        {
            (i++);
            if (((i % 2) == 0))
            {
                jump __zcnt3;
            }
            if ((i > 7))
            {
                jump __zbrk2;
            }
            (g_count += i);
            @__zcnt3;
        }
        @__zbrk2;
        llOwnerSay(((("while: " + ((string)g_count)) + " i=") + ((string)i)));
        integer k = 0;
        do
        {
            (k += 3);
        }
        while ((k < 10));
        llOwnerSay(("do-while k=" + ((string)k)));
        list names = ["ann", "bob", "cy"];
        {
            list _lst_59_9 = names;
            integer _idx_59_9 = 0;
            for (; (_idx_59_9 < llGetListLength(_lst_59_9)); (++_idx_59_9))
            {
                string nm = llList2String(_lst_59_9, _idx_59_9);
                if ((!(nm == "bob")))
                {
                    llOwnerSay(("name " + nm));
                }
            }
        }
        integer n = 5;
        while ((!(n <= 2)))
        {
            (n--);
        }
        llOwnerSay(("until n=" + ((string)n)));
        integer sel;
        for ((sel = 0); (sel < 4); (sel++))
        {
            integer __zswt12 = sel;
            if (__zswt12 == 0) jump __zswc12_0;
            if (__zswt12 == 1) jump __zswc12_1;
            if ((__zswt12 >= 2) && (__zswt12 <= 3)) jump __zswc12_2;
            jump __zswc12_3;
            @__zswc12_0;
            llOwnerSay("zero");
            @__zswc12_1;
            llOwnerSay("one (also reached from zero)");
            jump __zswend12;
            @__zswc12_2;
            llOwnerSay("two or three");
            jump __zswend12;
            @__zswc12_3;
            llOwnerSay("other");
            @__zswend12;
        }
        integer big = ((1 + 2) + 3);
        list l = [1, 2, 3, 4];
        llOwnerSay(((((((((grade(95) + grade(80)) + grade(10)) + " ") + ((string)sum_to(4))) + " ") + ((string)big)) + " ") + ((string)llGetListLength(l))));
        integer tries = 0;
        @again;
        (tries++);
        if ((tries < 3))
        {
            jump again;
        }
        llOwnerSay(("tries=" + ((string)tries)));
        state second;
    }
```

---

## 5. Gotchas

1. **A line that ends in `:` is a header.** That includes a ternary split across lines:
   ```lsl
   string s = cond ? "a" :      // ← parsed as a block header, not a continuation
       "b"
   ```
   Keep `? :` on one line, or wrap it in parentheses.
2. **No leading-operator continuation.** A line that *starts* with `+` or `&&` begins a new statement. End the previous line with the operator instead (as `integer big = 1 +` does in the example).
3. **`vector v = <1,2,3>` newline is fine; a line ending in `<` or `>` is not a continuation**, by design (see table).
4. **Dedent to a level that was never opened** is `inconsistent indentation`, not a silent re-parent.
5. **The condition of `for` keeps its own `;`**: `for i = 0; i < 3; i++:`. Only the final `:` is the block opener.
6. **`else` / `elif` must line up with the `if`** they belong to; the dedent is what closes the previous block.
7. **Closures stay braced.** Inside `( )` the pass is switched off, so `() => { a(); b(); }` needs its own semicolons ([[ZEXT Arrows - Async Bind, Fat Arrow and Timers]]).
8. **Heredocs and triple-quoted strings** may start at column 1 inside an indented block: their lines belong to the string token, not to the block structure ([[ZEXT Heredocs]]).

---

## 6. Implementation notes

- `ZLSLTokenizer.TryReadIndentDirective` sets the mode; `Tokenize()` then runs `ZIndentTransform.ApplyIndent` and always runs `RewriteKeywords` (brace-mode `elif (c)`, `unless (c)`, `until (c)`).
- The logical line is found with `LineEnd`: it keeps going while a bracket is open or the last token is a continuation operator. `Token.EndLine` (added for this) lets a multi-line string count as part of its line.
- Levels are a stack of `(column, braced, isDo)`; a deeper line after a non-header (e.g. a `case` body) pushes an *unbraced* level, so dedenting out of it emits nothing.
- `ZTokenNotes` (on `tokens[0].Notes`) carries two things to the parser: ZEXT uses to report when the gate is on, and indentation errors.
- `elif` / `unless` / `until` as brace-mode keywords only fire at statement start and only when the token after the closing `)` is not `;`, so a script's own function called `until(5);` keeps working (and a declaration `until(integer x) { … }` is recognised by its type keyword).

See also: [[ZEXT Syntax Sugar Index]], [[ZEXT Small Sugar - def on let defer assert print in]].
