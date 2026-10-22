---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - strings
  - operators
  - states
aliases:
  - ZEXT strings operators states
---

# ZEXT annotated: string forms, operators and state arguments

← Back to [[ZEXT Syntax Sugar Index]] | Reference: [[ZEXT Language Extensions]] (the gated-construct table) | Other examples: [[ZEXT Constructs - One Example Each - Strings, indexing and slices]] | Output: [[ZEXT Desugar - Stock LSL Output]] | Checked by: [[ZEXT Desugar Verification Report]]

The older constructs (strings, operators, state parameters) predate the 2026-10 syntax pass, but until now their desugared form was only checked for "does it compile". These three scripts are in the equivalence suite: **original and desugared output are compared line for line**. Three real differences were found and fixed that way; they are called out below.

---

## 1. Python-style strings: `tasks/desugar_suite/sugar_07_string_forms.lsl`

| Form | Meaning |
| :--- | :--- |
| `f"{who} has {n * 2}"` | interpolation; `{expr}` is any expression, converted with `(string)` |
| `r"raw \n"` | raw: a backslash is an ordinary character |
| `"""…"""`, `'''…'''` | triple-quoted: may span lines and contain the quote character |
| `'…'` | single-quoted: the other quote is free to appear |
| `"ab" "cd"` | adjacent literals are joined at compile time |
| `"%s owns %d (%f) 100%%" % [who, n, 2.5]` | formatting: `%s` `%d` `%i` `%f` `%%`; an unknown `%x` and a trailing `%` are kept; a missing argument prints nothing |
| `"ab" * 3`, `3 * "ab"` | repetition; a count `<= 0` gives `""` |
| `s[0]`, `s[-1]` | one character (negative counts from the end); also `list[i]` |
| `s[1:3]`, `s[:2]`, `s[5:]` | slice, **stop inclusive** (same as `llGetSubString` / `llList2List`); also on lists |

```lsl
// =============================================================================================
// sugar_07_string_forms.lsl  -  Python-style strings (ZEXT only), in ordinary brace syntax
//
//   f"...{expr}..."    interpolation; {expr} is any expression, converted with (string)
//   r"..."             raw: a backslash is an ordinary character
//   """...""" '''...'''  triple-quoted: may span lines, may contain the quote character
//   '...'              single-quoted: same as "..." with the other quote free to appear inside
//   "ab" "cd"          adjacent literals are concatenated at compile time
//   "fmt" % [args]     %s %d %i %f and %% (a list on the right)
//   "ab" * 3  3 * "ab" repetition (a count <= 0 gives "")
//   s[i]  s[-1]        one character (negative counts from the end); also list[i] (l[-1] is the last item)
//   s[a:b] s[:b] s[a:] slice, STOP INCLUSIVE (same as llGetSubString / llList2List); also on lists
// =============================================================================================

default
{
    touch_start(integer n)
    {
        string who = "Ann";
        integer cnt = 3;
        llOwnerSay(f"{who} has {cnt} items, {cnt * 2} halves, upper={llToUpper(who)}");
        llOwnerSay(r"raw \n keeps the backslash");
        llOwnerSay("""triple "quoted"
spans lines""");
        llOwnerSay('single "quoted" with double quotes inside');
        llOwnerSay("adjacent " "literals " "join");
        llOwnerSay("%s owns %d (%f) 100%%" % [who, cnt, 2.5]);
        llOwnerSay("ab" * 3 + "|" + (string)(2 * "xy") + "|" + ("z" * 0) + "|");

        string s = "abcdefgh";
        llOwnerSay(s[0] + s[-1] + " " + s[1:3] + " " + s[:2] + " " + s[5:]);
        list l = [10, 20, 30, 40];
        llOwnerSay((string)l[1] + " " + llList2CSV(l[1:2]) + " " + llList2CSV(l[2:]));
        llOwnerSay((string)l[-1] + " " + (string)l[-2]);                  // negative index: from the end
    }
}
```

Output (as written = desugared):

```text
Ann has 3 items, 6 halves, upper=ANN
raw \n keeps the backslash
triple "quoted"
spans lines
single "quoted" with double quotes inside
adjacent literals join
Ann owns 3 (2.500000) 100%
ababab|xyxy||
ah bcd abc fgh
20 20, 30 30, 40
40 30
```

**Desugaring.** f-strings, raw/triple/single strings, adjacent literals and maps are already ordinary expressions after parsing. `*` and `%` have no stock operator, so the output gets **generated helper functions**, placed at the very top so no call needs a forward reference: `__zrepeat(string, integer)` (log₂ doubling loop) and `__zfmt(string, list)` (a character loop over `llGetSubString` that reproduces the native rules above). Indexing and slicing become `llGetSubString` / `llList2<Type>` / `llList2List`, chosen from the inferred type.

```lsl
    touch_start(integer n)
    {
        string who = "Ann";
        integer cnt = 3;
        llOwnerSay(((((((((string)who) + " has ") + ((string)cnt)) + " items, ") + ((string)(cnt * 2))) + " halves, upper=") + ((string)llToUpper(who))));
        llOwnerSay("raw \\n keeps the backslash");
        llOwnerSay("triple \"quoted\"\nspans lines");
        llOwnerSay("single \"quoted\" with double quotes inside");
        llOwnerSay((("adjacent " + "literals ") + "join"));
        llOwnerSay(__zfmt("%s owns %d (%f) 100%%", [who, cnt, 2.5]));
        llOwnerSay((((((__zrepeat("ab", 3) + "|") + ((string)__zrepeat("xy", 2))) + "|") + __zrepeat("z", 0)) + "|"));
        string s = "abcdefgh";
        llOwnerSay((((((((llGetSubString(s, 0, 0) + llGetSubString(s, (-1), (-1))) + " ") + llGetSubString(s, 1, 3)) + " ") + llGetSubString(s, 0, 2)) + " ") + llGetSubString(s, 5, -1)));
        list l = [10, 20, 30, 40];
        llOwnerSay(((((((string)llList2String(l, 1)) + " ") + llList2CSV(llList2List(l, 1, 2))) + " ") + llList2CSV(llList2List(l, 2, -1))));
        llOwnerSay(((((string)llList2String(l, (-1))) + " ") + ((string)llList2String(l, (-2)))));
    }
```

> [!WARNING] Divergence left in place
> ZEngine's own string→integer helper is *strict* (`(integer)"12abc"` is `0`), while real LSL, and therefore the desugared `__zfmt` through `llList2Integer`, gives `12`. `"%d" % ["12abc"]` therefore differs between the two forms. Not touched here because it would change casts everywhere; the examples avoid it.

Fixed while writing this: the native `%d`/`%f`/`%s` formatter ignored string items of the argument list (they arrive as the `LSL_String` wrapper type) and printed an empty string.

---

## 2. Expression operators: `tasks/desugar_suite/sugar_08_operators.lsl`

| Operator | Meaning | Desugared |
| :--- | :--- | :--- |
| `c ? a : b` | ternary | temp + `if`/`else` before the statement (see [[ZEXT Desugar - Stock LSL Output]] for loops and `&&`) |
| `a ?? b` | `b` when `a` is `""`, `NULL_KEY` or the JSON-invalid sentinel (tested **before** a cast, so `(integer)llJsonGetValue(…) ?? 100` works) | the same ternary |
| `x \|> f(args)` | `f(x, args)` | `f(x, args)` |
| `\|=` `&=` `^=` `<<=` `>>=` | compound bitwise/shift assignment | `x = (x op y)` |
| `boolean`, `true`, `false` | an integer alias with lowercase literals | `integer`, `1`, `0` — but see below |

```lsl
// =============================================================================================
// sugar_08_operators.lsl  -  expression-level ZEXT operators, in ordinary brace syntax
//
//   c ? a : b          ternary (right-associative)
//   a ?? b             null-coalescing: b when a is "", NULL_KEY or JSON_INVALID (tested BEFORE a cast)
//   x |> f(args)       pipeline: f(x, args)
//   x |= y  &= ^= <<= >>=   compound bitwise / shift assignment
//   boolean, true/false     lowercase literals and the boolean type (an integer alias)
// =============================================================================================

default
{
    touch_start(integer n)
    {
        integer a = 7;
        integer b = 3;
        llOwnerSay((string)(a > b ? a : b) + " " + (a < b ? "lt" : "ge"));

        string name = "";
        llOwnerSay(name ?? "anon");
        name = "Zoe";
        llOwnerSay(name ?? "anon");
        llOwnerSay((string)((integer)llJsonGetValue("{\"a\": 5}", ["b"]) ?? 100));   // missing key -> 100

        llOwnerSay("  hello  " |> llStringTrim(STRING_TRIM) |> llToUpper());

        integer f = 5;
        f |= 2;
        f &= 6;
        f ^= 1;
        f <<= 2;
        f >>= 1;
        llOwnerSay("f=" + (string)f);

        boolean flag = true;
        integer ok = false;
        llOwnerSay((string)flag + (string)ok);

        // plain LSL operators, to be sure they survive the round trip
        vector v = <1.0, 2.0, 3.0>;
        llOwnerSay((string)(v * 2.0) + " " + (string)(~a) + " " + (string)(!a) + " " + (string)(-a) + " " + (string)(a % b));
        integer c = 0;
        c++;
        ++c;
        c--;
        llOwnerSay("c=" + (string)c + " " + (string)(a << 2) + " " + (string)(a >> 1) + " " + (string)(a & b | 8));
    }
}
```

Output (as written = desugared):

```text
7 ge
anon
Zoe
100
HELLO
f=14
TRUE0
<2.000000, 4.000000, 6.000000> -8 0 -7 1
c=1 28 3 11
```

> [!WARNING] Found and fixed by this check: `boolean` prints `TRUE` / `FALSE`
> Natively `(string)flag` for a `boolean` yields `TRUE`/`FALSE` (the line `TRUE0` above: a `boolean` set to `true` followed by an `integer` set to `false`), but the desugarer used to map `boolean` to a plain `integer` and print `1`. The desugarer now spells the conversion out (`if (flag) s = "TRUE"; else s = "FALSE";`) wherever a boolean meets a string: an explicit `(string)` cast, `+` with a string, assignment to a string, and **a call argument whose parameter is a string** (it asks the registry for the parameter types, so `llOwnerSay(flag)` and a script function taking a string both match).

---

## 3. State arguments: `tasks/desugar_suite/sugar_09_state_arguments.lsl`

`state worker(integer times, string word) { … }` declares parameters; `state worker(g_note + 1, "go");` passes arguments. The arguments are evaluated in the **calling** state, stored in hidden globals, then loaded into the parameters at the top of the target state's `state_entry()` (created if the state has none). The parser performs this rewrite, so desugar prints globals and a loader and nothing else.

```lsl
// =============================================================================================
// sugar_09_state_arguments.lsl  -  states with parameters (ZEXT only)
//
//   state worker(integer times, string word) { state_entry() { ... } }     declaration
//   state worker(3, "go");                                                    arguments at the call
//
// The arguments are stored in globals, then loaded into locals at the top of the target state's
// state_entry().  They are evaluated in the calling state, before the change.
// =============================================================================================

integer g_note = 2;

default
{
    state_entry()
    {
        llOwnerSay("default state_entry");
        state worker(g_note + 1, "go");
    }
}

state worker(integer times, string word)
{
    state_entry()
    {
        integer i;
        for (i = 0; i < times; i++)
            llOwnerSay(word + (string)i);
        if (times > 2)
            state done(times - 2);
    }
}

state done(integer left)
{
    state_entry()
    {
        llOwnerSay("done with " + (string)left + " to spare");
    }
}
```

Output (as written = desugared):

```text
default state_entry
go0
go1
go2
done with 1 to spare
```

```lsl
// Desugared from ZEXT to stock LSL by `zengine desugar`.
// Generated file: comments and formatting from the original are not preserved.

integer g_note = 2;
string __zstate_worker_arg_1;
integer __zstate_worker_arg_0;
integer __zstate_done_arg_0;

default
{
    state_entry()
    {
        llOwnerSay("default state_entry");
        (__zstate_worker_arg_0 = (g_note + 1));
        (__zstate_worker_arg_1 = "go");
        state worker;
    }
}

state worker
{
    state_entry()
    {
        integer times = __zstate_worker_arg_0;
        string word = __zstate_worker_arg_1;
        integer i;
        for ((i = 0); (i < times); (i++))
        {
            llOwnerSay((word + ((string)i)));
        }
        if ((times > 2))
        {
            (__zstate_done_arg_0 = (times - 2));
            state done;
        }
    }
}

state done
{
    state_entry()
    {
        integer left = __zstate_done_arg_0;
        llOwnerSay((("done with " + ((string)left)) + " to spare"));
    }
}
```

> [!WARNING] Found and fixed by this check
> `if (times > 2) state done(times - 2);` — a state change **with arguments as the unbraced body of an `if`** — produced a compound statement that the desugarer could not print (`cannot desugar … unsupported statement InlineBlock`). It now emits the statements inside the `if`.

See also: [[ZEXT Syntax Sugar Index]].
