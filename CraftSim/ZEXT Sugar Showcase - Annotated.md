---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - examples
  - desugar
aliases:
  - sugar showcase
---

# ZEXT Sugar Showcase: one script, nearly everything, braceless, and its stock-LSL form

← Back to [[ZEXT Syntax Sugar Index]] | Output rules: [[ZEXT Desugar - Stock LSL Output]] | Checked by: [[ZEXT Desugar Verification Report]]

One 114-line script written the way the sugar is meant to be used (`#indent`, no semicolons, no braces except inside the one closure) and the stock LSL that `zengine desugar` makes of it (459 lines). Both files live in the repo (`tasks/sugar_showcase.lsl`, `tasks/sugar_showcase.desugared.lsl`); a test regenerates the desugared text on every run and fails if the committed file is stale, then runs both forms and requires **identical output**, including a real HTTP timeout.

It uses: [[ZEXT Braceless Indent Mode]], `->` with a timeout `else:` inside functions and a loop, `=>` (a `llListen` closure), [[ZEXT JSON Literals (j-strings)]], [[ZEXT Heredocs]], string `*` and `%`, string `case` ranges, a `?:` in a global initialiser, binary/octal/separator literals, default and named arguments, `let`, `defer` (also before `continue`), `assert`, `print`, `elif`/`unless`/`until`, chained comparison, `in`, a list comprehension, `foreach`, `switch`, and a slice. See [[ZEXT Small Sugar - def on let defer assert print in]].

---

## The source

```lsl
#indent
// ZEngine sugar showcase: indentation blocks, no semicolons, no braces (except the closure).
// Desugared, stock-LSL form: tasks/sugar_showcase.desugared.lsl (kept in sync by ZDesugarTest).

integer MAX_TURNS = 0b11                      // binary literal -> 3
integer BIG       = 1_000                     // digit separator
integer MASK      = 0o17 & 0xF0               // octal literal
string  MODE      = BIG > 500 ? "bulk" : "single"   // ternary in a global initialiser
string  BANNER    = << END
Report v1 "stable"
  cost: 5%   // not a comment, not a directive
END
string  CONFIG    = j'{"retries": 2, "tags": ["a", "b", "c"]}'   // JSON checked at compile time
list    g_log

def log(string msg, integer level = 1):       // default argument
    g_log += [level, msg]

def tag(string s, string left = "<", string right = ">") -> string:
    return left + s + right

def classify(integer n) -> string:
    switch n:
        case 0:
            return "zero"
        case 1..9:
            return "small"
        default:
            if n in [10, 20, 30]:
                return "round"
            return n > BIG ? "huge" : "large"

def histogram(list words) -> string:
    string out = ""
    let seen = []                             // type inferred
    foreach string w in words:
        if w in seen:
            continue
        seen += [w]
        integer c = 0
        foreach string v in words:
            if v == w:
                c++
        out += f"{w}={c} " + ("#" * c) + " "
    return llStringTrim(out, STRING_TRIM)

def fetch(string url, float limit = 0.5) -> string:
    llHTTPRequest(url, [], "") -> string body : limit else:
        log(f"timeout for {url}", 2)
        return "<timeout>"
    return body

def predict(list features) -> string:
    osMLpredictAsync("no-such-model", features) -> string r
    return r

default:
    state_entry():
        defer print("state_entry finished")   // runs on every way out of this block
        assert MODE == "bulk", "MODE should be bulk"
        print("banner is", llStringLength(BANNER), "chars; mask", MASK)
        print("retries:", llJsonGetValue(CONFIG, ["retries"]))
        llListen(7, "", NULL_KEY, "", (integer ch, string who, key src, string text) => { print("heard", text); })

    on touch_start(integer n):
        let words = ["pear", "fig", "pear", "kiwi", "fig", "pear"]
        print(histogram(words))
        print("%s has %d letters (%f)" % ["pear", 4, 0.5])
        print("=" * 12)

        list lens = [llStringLength(w) for string w in words if w != "fig"]
        let total = 0
        foreach integer x in lens:
            total += x
        print("lens", llList2CSV(lens), "total", total)

        integer i = 0
        until i >= 3:
            let label = classify(i * 7)
            print(i, tag(label), tag(label, right: "]"), tag(right: "!", s: label))
            i++

        if 0 < total <= BIG:                  // chained comparison
            print("total in range")
        elif total > BIG:
            print("too big")
        else:
            print("negative?")
        unless total in [0, 1]:
            print("not tiny")

        integer turn = 0
        while turn < MAX_TURNS:
            defer turn++                      // also runs before `continue`
            string reply
            if turn == 1:
                reply = fetch("http://127.0.0.1:59999/x")     // times out
            else:
                reply = predict([1.0 * turn])                 // answers
            switch reply[0:0]:
                case "<".."<":
                    print("turn", turn, "timed out")
                    break
                case "A".."M":
                    print("turn", turn, "reply in A..M, non-empty:", llStringLength(reply) > 0)
                    break
                default:
                    print("turn", turn, "other reply, non-empty:", llStringLength(reply) > 0)
            if turn == 2:
                print("skipping the tail of turn 2")
                continue
            print("end of turn", turn)

        print(llGetListLength(g_log) / 2, "log entries")
```

## What it printed

Identical natively and desugared (with the real 0.5 s timeout of `fetch` in the middle):

```text
banner is 65 chars; mask 0
retries: 2
state_entry finished
pear=3 ### fig=2 ## kiwi=1 #
pear has 4 letters (0.500000)
============
lens 4, 4, 4, 4 total 16
0 <zero> <zero] <zero!
1 <small> <small] <small!
2 <large> <large] <large!
total in range
not tiny
turn 0 reply in A..M, non-empty: 1
end of turn 0
turn 1 timed out
end of turn 1
turn 2 reply in A..M, non-empty: 1
skipping the tail of turn 2
1 log entries
```

Reading it: `turn 1 timed out` comes from the `else:` branch of `fetch` (it logs, then returns `"<timeout>"`, whose first character selects the `"<".."<"` case); turns 0 and 2 get the error text of the model call (first character in `"A".."M"`); the `defer turn++` runs on every pass, including the pass that `continue`s at turn 2 ("skipping the tail of turn 2" and no "end of turn 2"); `state_entry finished` is the deferred print.

---

## Selected pieces of the generated stock LSL

The helpers come first (only the ones used are emitted); then globals, then functions, then the states.

**`MODE` has a `?:` in its initialiser**, so it is declared bare and assigned once at the start of `state_entry`; `state_entry` also shows the `assert`, the `defer` (last), and the closure's registration:

```lsl
    state_entry()
    {
        if ((__zginit == 0))
        {
            (__zginit = 1);
            string __zte15;
            if ((BIG > 500)) __zte15 = "bulk"; else __zte15 = "single";
            (MODE = __zte15);
        }
        {
        }
        if ((!(MODE == "bulk")))
        {
            llOwnerSay((("assertion failed at line 60" + ": ") + ((string)"MODE should be bulk")));
        }
        llOwnerSay(((((((((string)"banner is") + " ") + ((string)llStringLength(BANNER))) + " ") + ((string)"chars; mask")) + " ") + ((string)MASK)));
        llOwnerSay(((((string)"retries:") + " ") + ((string)llJsonGetValue(CONFIG, ["retries"]))));
        __z_call_listen(7, "", NULL_KEY, "", 1);
        {
            llOwnerSay(((string)"state_entry finished"));
        }
    }
```

**The closure** became a helper and a branch of `listen`:

```lsl
integer __z_call_listen(integer chan, string name, key id, string msg, integer cb_id)
{
    (__zlisten_cb = cb_id);
    return llListen(chan, name, id, msg);
}
```

```lsl
    listen(integer chan, string name, key id, string msg)
    {
        if ((__zlisten_cb == 1))
        {
            integer ch = chan;
            string who = name;
            key src = id;
            string text = msg;
            llOwnerSay(((((string)"heard") + " ") + ((string)text)));
        }
    }
```

**`classify`**: the integer range `case 1..9`, the `in`, and the `?:`:

```lsl
string classify(integer n)
{
    integer __zswt0 = n;
    if (__zswt0 == 0) jump __zswc0_0;
    if ((__zswt0 >= 1) && (__zswt0 <= 9)) jump __zswc0_1;
    jump __zswc0_2;
    @__zswc0_0;
    return "zero";
    @__zswc0_1;
    return "small";
    @__zswc0_2;
    if ((llListFindList([10, 20, 30], [n]) != -1))
    {
        return "round";
    }
    string __zte1;
    if ((n > BIG)) __zte1 = "huge"; else __zte1 = "large";
    return __zte1;
    @__zswend0;
}
```

**`histogram`**: `foreach`, `continue` as a label, `in`, an f-string and `"#" * c`:

```lsl
string histogram(list words)
{
    string out = "";
    list seen = [];
    {
        list _lst_36_5 = words;
        integer _idx_36_5 = 0;
        for (; (_idx_36_5 < llGetListLength(_lst_36_5)); (++_idx_36_5))
        {
            string w = llList2String(_lst_36_5, _idx_36_5);
            if ((llListFindList(seen, [w]) != -1))
            {
                jump __zcnt3;
            }
            (seen += [w]);
            integer c = 0;
            {
                list _lst_41_9 = words;
                integer _idx_41_9 = 0;
                for (; (_idx_41_9 < llGetListLength(_lst_41_9)); (++_idx_41_9))
                {
                    string v = llList2String(_lst_41_9, _idx_41_9);
                    if ((v == w))
                    {
                        (c++);
                    }
                }
            }
            (out += (((((((string)w) + "=") + ((string)c)) + " ") + __zrepeat("#", c)) + " "));
            @__zcnt3;
        }
    }
    return llStringTrim(out, STRING_TRIM);
}
```

**`touch_start`** is only a stub: everything after the first suspension lives in the generated dispatch function `__zrun1_touch_start`, resumed by the `dataserver`, `http_response` and `timer` handlers below it. The `switch` over string ranges, the `defer turn++` copies, the `continue` as `jump __zdisp1`:

```lsl
    touch_start(integer n)
    {
        if ((__zstate == 0))
        {
            (__zl2_n = n);
            __zrun1_touch_start(0);
            return;
        }
    }
```

```lsl
    dataserver(key queryid, string data)
    {
        if (((__zstate == 1) && (queryid == __zreq_1)))
        {
            llSetTimerEvent(0.0);
            (__zl12___i1_fetch_body = data);
            (__zstate = 0);
            __zrun1_touch_start(9);
            return;
        }
        if (((__zstate == 2) && (queryid == __zreq_2)))
        {
            (__zl15___i3_predict_r = data);
            (__zstate = 0);
            __zrun1_touch_start(11);
            return;
        }
    }
```

```lsl
    timer()
    {
        if ((__zstate == 1))
        {
            (__zstate = 0);
            llSetTimerEvent(0.0);
            __zrun1_touch_start(10);
            return;
        }
    }
```

The dispatch function itself is 189 lines of numbered `if (__zpc == n)` arms; open `tasks/sugar_showcase.desugared.lsl` to read it, or see the smaller one in [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]].

---

## Constructs deliberately left out

- `decimal(p,s)` and closures other than the three supported call forms: desugar refuses them ([[ZEXT Desugar - Stock LSL Output]] §5).
- A `?:` that contains an async call (`c ? f() : g()` where `f` suspends): a call inside a ternary branch is not among the positions `ZFunctionInliner` is documented to expand, and I did not test it, so the example uses `if`/`else`.
- Network-dependent calls other than a local closed-port `llHTTPRequest`.
