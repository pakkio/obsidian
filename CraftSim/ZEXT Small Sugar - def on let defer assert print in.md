---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - sugar
aliases:
  - small sugar
  - defer
  - let
  - list comprehension
  - default arguments
---

# ZEXT small sugar: `def` `on` `let` `defer` `assert` `print` `in` and friends

← Back to [[ZEXT Syntax Sugar Index]] | Reference: [[ZEXT Language Extensions]] | Block syntax: [[ZEXT Braceless Indent Mode]] | Output: [[ZEXT Desugar - Stock LSL Output]] | Checked by: [[ZEXT Desugar Verification Report]]

Every construct here is ZEXT only (`[ZEngine] ZEXT = false` turns each into `line L:C: <what> is a ZEngine extension, not standard LSL …`). Almost all of them are **resolved in the tokenizer or parser**, so the type checker, the code generator and `zengine desugar` see plain AST and need no special support. Source: `ZLSLParser.cs`, `ZLSLTokenizer.cs`, `ZIndentTransform.cs`, plus `ZTypeInfer.cs` / `ZCILCodeGen.cs` for `let` and `in`. Example: `tasks/desugar_suite/sugar_06_small_sugar.lsl`; tests `ZSugarTest`, `ZSugarRunTest`.

| Construct | Write | Means | ZEXT message |
| :--- | :--- | :--- | :--- |
| `def` | `def f(integer a) -> integer:` | function; `-> type` is the return type (none = void) | `'def' function syntax` |
| `on` | `on touch_start(integer n):` | optional word before an event handler | `'on' before an event handler` |
| default args | `def f(integer a, integer b = 2)` | omitted trailing arguments take the default, **evaluated at the call site** | `a default argument value` |
| named args | `f(1, b: 3)` `f(b: 3, a: 1)` | arguments by name (`name: value`) | `a named argument` |
| `let` | `let x = expr` | local whose type is read off the initialiser | `'let' declarations` |
| `defer` | `defer stmt` / `defer:` + block | run at scope exit | `'defer'` |
| `assert` | `assert c, "msg"` | report and carry on | `'assert'` |
| `print` | `print(a, b)` | `llOwnerSay` of the arguments joined by a space | `print()` |
| chained compare | `0 < x <= 10` | `(0 < x) && (x <= 10)` | `chained comparison` |
| `in` | `x in list` `"ab" in s` | membership | `the 'in' operator` |
| comprehension | `[e for T v in src if c]` | build a list | `a list comprehension` |
| number literals | `0b1011` `0o17` `1_000_000` | binary, octal, digit separator | `a binary literal (0b1010)` etc. |
| `elif` `unless` `until` | `unless c:` | `else if`, `if (!(c))`, `while (!(c))` | `'unless'` etc. |

---

## 1. `def` and `on`

`def` before a top-level function name makes the return type optional (`-> type` follows the parameters; no arrow = void). `on` is a decoration in front of an event handler. In a braceless file they are headers like any other: `def f(integer a) -> integer:`, `on touch_start(integer n):`.

## 2. Default and named arguments

```lsl
def clamp(integer v, integer lo = 0, integer hi = 10) -> integer:
    …
clamp(15)                 // clamp(15, 0, 10)
clamp(5, hi: 3)           // clamp(5, 0, 3)
clamp(lo: 2, v: 1)        // all named, any order
```

- A default is kept as a **token range and re-parsed at every call that omits it**, so each call gets a fresh expression and it may name globals. It is *not* evaluated once at definition time (unlike Python).
- Named arguments use `name: value` because `name = value` is already an assignment expression in LSL. Only calls to **functions defined in the script** can be named; naming an argument of an `ll*`/`os*` function is an error.
- Resolution happens after the whole file is parsed, so calling a function that is defined **later** works.
- Errors: `call to 'f' is missing an argument for 'a'`, `'f' has no parameter named 'c'`, `parameter 'a' of 'f' was already given positionally`, `a positional argument cannot follow a named one`, `named arguments are only supported for functions defined in the script`.
- Known limit: a call written inside an f-string's `{ }` is parsed by a separate mini-parser and does **not** get defaults.

## 3. `let`

`let x = expr` declares `x` with the type of `expr`. The parser emits a declaration with a placeholder type; `ZTypeInfer` fills it in. If the initialiser has no static type (an element of a list: `let a = lst[0]`) the error is `cannot infer a type for 'let a' (give it an explicit type)`. Works in `for (let i = 0; i < n; i++)`. Not available for globals. (`ZFunctionInliner` copies declarations in four places and must keep carrying the `InferType` flag, a trap if you extend it.)

## 4. `defer`

```lsl
defer print("bye")        // one statement
defer:                    // or a block (braceless), brace form: defer { … }
    cleanup()
```

A deferred statement runs when the **block it was written in ends**, and also **before any `return`, `state x;`, `break` or `continue` that leaves that block** (a `break`/`continue` only runs the defers declared inside the loop it leaves). Several defers run last-in first-out. It does **not** run before a `jump`.

Mechanics, because they explain the corner cases: the statement is kept as a **token range** and parsed again at each exit point, so every copy is a fresh AST. A `return expr;` evaluates `expr` into a hidden temporary *first* (`let`-style), then the deferred code runs, then the function returns the temporary, so deferred code cannot change the returned value. A closure body starts with an empty defer list. A `defer` inside a deferred statement is ignored.

## 5. `assert` and `print`

- `assert cond, "message"` (or `assert(cond, "message")` when the `)` is followed by `;`) lowers to `if (!(cond)) llOwnerSay("assertion failed at line N: message")`. **It reports and carries on**; it does not stop the script. The line number is the source line of the `assert`.
- `print(a, b, …)` lowers to `llOwnerSay((string)a + " " + (string)b …)`; no arguments prints an empty line.
- Both defer to a script function of the same name (`ScanUserFunctions` looks for `name(...) {` at top level), so existing scripts that define `print()` or `assert()` are unaffected.

## 6. Chained comparison

`0 < x <= BIG` is `(0 < x) && (x <= BIG)`. The middle operand is evaluated **twice**, so it must be a variable or a literal (`chained comparison needs a variable or literal as the middle operand (it is evaluated twice)`). Under ZEXT this also changes the meaning of `a < b < c`, which in stock LSL is `(a < b) < c`.

## 7. `in`

`x in coll` is `llListFindList(coll, [x]) != -1` for a list and `llSubStringIndex(coll, x) != -1` for a string or key; any other right-hand type is `'in' needs a list or a string on the right-hand side`. It binds like a comparison; there is no `not in` (write `!(x in c)`). In `foreach (type v in list)` the `in` belongs to the loop header and is unrelated. Internally the parser emits an `__zIn(x, coll)` call that `ZCILCodeGen.LowerIn` and `ZLSLDesugar` expand once the types are known.

## 8. List comprehension

```lsl
list odds = [i for integer i in [1,2,3,4,5,6] if i % 2 == 1];
names = [(string)x + "!" for integer x in l];
```

The element type must be written (`integer i`): list items are typed at run time, so there is nothing to infer it from. It lowers to an empty list plus a `foreach`, building into a temporary when you assign to an existing variable (so the source may mention it). It is allowed only as the **initialiser of a declaration** or the **right side of `name = …;`**; anywhere else: `a list comprehension can only be the initialiser of a declaration or the right-hand side of an assignment statement`.

## 9. Number literals

`0b1011` and `0o17` are converted to the equivalent hex literal by the tokenizer (values above 32 bits are a lexer error); `1_000_000` strips the underscores (an underscore counts only between two digits, so `1_x` is `1` then `_x`). Hex literals accept separators too.

## 10. `elif` `unless` `until`

In a braceless file they are headers (`elif cond:`, `unless cond:`, `until cond:`). In brace syntax: `elif (c) { }`, `unless (c) stmt`, `until (c) stmt`. They are recognised only at the start of a statement and only when the token after the closing `)` is not `;`, so a script's own `until(5);` call is left alone.

---

## The annotated example

```lsl
#indent
// =============================================================================================
// sugar_06_small_sugar.lsl  -  the small conveniences (all ZEXT only)
//
//   def name(params) -> type:     function with an optional `def`, and `-> type` for the return type
//   on event(params):             `on` before an event handler is optional
//   integer b = 2   (parameter)   default argument value, evaluated at the call site; callers may
//   f(x, b: 3)                    name arguments (`name: value`) - only for script functions
//   let v = expr                  local whose type is read off the initialiser
//   defer stmt / defer:           runs when the enclosing block ends, and before any return /
//                                 state change / break / continue that leaves it (not before `jump`)
//   assert cond[, "message"]      reports "assertion failed at line N: message" via llOwnerSay and
//                                 carries on (it does not stop the script)
//   print(a, b, ...)              llOwnerSay of the arguments joined by spaces
//   a < b <= c                    chained comparison  (a < b) && (b <= c); b must be a variable/literal
//   x in list / "ab" in str       membership: llListFindList / llSubStringIndex
//   [e for T v in src if c]       list comprehension (needs the element type; only as the
//                                 initialiser of a declaration or the right side of `name = ...`)
//   0b1010 0o17 1_000_000         binary / octal literals, digit separators
//   elif / unless / until         see sugar_01_indent_braceless.lsl
// print and assert are plain sugar only while the script defines no function of the same name.
// =============================================================================================

def clamp(integer v, integer lo = 0, integer hi = 10) -> integer:
    if v < lo:
        return lo
    if v > hi:
        return hi
    return v

def banner(string s, string pad = "*", integer width = 20) -> string:
    string line = pad * width
    return line + "\n" + s + "\n" + line

default:
    on touch_start(integer n):
        defer print("bye")                          // last thing this handler prints
        let x = clamp(15)                           // integer
        let name = "zed"                            // string
        let v = <1.0, 2.0, 3.0>                     // vector
        print("x", x, "name", name, "v", v)
        print(clamp(-5), clamp(5, hi: 3), clamp(lo: 2, v: 1))
        print(banner("hello", width: 5))
        assert x == 10, "x should have been clamped"
        assert x == 99, "this one fails and only reports"
        if 0 < x <= 10:
            print("chained comparison holds")
        list odds = [i for integer i in [1, 2, 3, 4, 5, 6] if i % 2 == 1]
        print(llList2CSV(odds), 3 in odds, 4 in odds, "ell" in "hello")
        print(0b1011, 0o17, 0xFF, 1_000_000)
        integer m = 0
        until m >= 3:
            m++
        unless m != 3:
            print("m is 3")
        if m == 0:
            print("zero")
        elif m == 3:
            print("three")
        else:
            print("other")
```

What it printed (as written = desugared):

```text
x 10 name zed v <1.000000, 2.000000, 3.000000>
0 3 2
*****
hello
*****
assertion failed at line 45: this one fails and only reports
chained comparison holds
1, 3, 5 1 0 1
11 15 255 1000000
m is 3
three
bye
```

Things to notice in the output: `x 10` (clamped), `<1.000000, 2.000000, 3.000000>` (a vector through `print`), the **failed assert** reported and the script carrying on, `bye` printed **last** (the `defer`), `1, 3, 5 1 0 1` (comprehension, then `3 in odds`, `4 in odds`, `"ell" in "hello"`).

## After desugaring

The functions: defaults are gone (each call site got its omitted arguments), and `*` became `__zrepeat`:

```lsl
integer clamp(integer v, integer lo, integer hi)
{
    if ((v < lo))
    {
        return lo;
    }
    if ((v > hi))
    {
        return hi;
    }
    return v;
}
```

```lsl
string banner(string s, string pad, integer width)
{
    string line = __zrepeat(pad, width);
    return ((((line + "\n") + s) + "\n") + line);
}
```

and the handler: `print` and `assert` are `llOwnerSay`, the `defer` was replayed at the end, `in` became `llListFindList` / `llSubStringIndex`, `until`/`unless` became negated conditions, the comprehension a loop:

```lsl
    touch_start(integer n)
    {
        {
        }
        integer x = clamp(15, 0, 10);
        string name = "zed";
        vector v = <1.0, 2.0, 3.0>;
        llOwnerSay(((((((((((((string)"x") + " ") + ((string)x)) + " ") + ((string)"name")) + " ") + ((string)name)) + " ") + ((string)"v")) + " ") + ((string)v)));
        llOwnerSay(((((((string)clamp((-5), 0, 10)) + " ") + ((string)clamp(5, 0, 3))) + " ") + ((string)clamp(1, 2, 10))));
        llOwnerSay(((string)banner("hello", "*", 5)));
        if ((!(x == 10)))
        {
            llOwnerSay((("assertion failed at line 44" + ": ") + ((string)"x should have been clamped")));
        }
        if ((!(x == 99)))
        {
            llOwnerSay((("assertion failed at line 45" + ": ") + ((string)"this one fails and only reports")));
        }
        if (((0 < x) && (x <= 10)))
        {
            llOwnerSay(((string)"chained comparison holds"));
        }
        list odds = [];
        {
            list _lst_48_9 = [1, 2, 3, 4, 5, 6];
            integer _idx_48_9 = 0;
            for (; (_idx_48_9 < llGetListLength(_lst_48_9)); (++_idx_48_9))
            {
                integer i = llList2Integer(_lst_48_9, _idx_48_9);
                if (((i % 2) == 1))
                {
                    (odds = (odds + [i]));
                }
            }
        }
        llOwnerSay(((((((((string)llList2CSV(odds)) + " ") + ((string)(llListFindList(odds, [3]) != -1))) + " ") + ((string)(llListFindList(odds, [4]) != -1))) + " ") + ((string)(llSubStringIndex("hello", "ell") != -1))));
        llOwnerSay(((((((((string)11) + " ") + ((string)15)) + " ") + ((string)255)) + " ") + ((string)1000000)));
        integer m = 0;
        while ((!(m >= 3)))
        {
            (m++);
        }
        if ((!(m != 3)))
        {
            llOwnerSay(((string)"m is 3"));
        }
        if ((m == 0))
        {
            llOwnerSay(((string)"zero"));
        }
        else if ((m == 3))
        {
            llOwnerSay(((string)"three"));
        }
        else
        {
            llOwnerSay(((string)"other"));
        }
        {
            llOwnerSay(((string)"bye"));
        }
    }
```

See also: [[ZEXT Syntax Sugar Index]], [[ZEXT Braceless Indent Mode]].
