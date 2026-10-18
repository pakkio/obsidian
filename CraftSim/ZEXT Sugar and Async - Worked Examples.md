---
tags:
  - craftsim
  - zext
  - zengine
  - async
  - syntactic-sugar
  - examples
aliases:
  - ZEXT examples
  - async bind examples
  - await parallel
---

# ZEXT Sugar and Async - Worked Examples

← Back to [[CraftSim]] | Reference: [[ZEXT Language Extensions]] | Engine: [[ZEngine]] | More examples: [[OSSL Extension Examples - Complete Coverage]]

[[ZEXT Language Extensions]] is the reference (which construct, which parser line, which error). This page is the tour: small scripts you can run, what they print, and what they turn into. **Every script was run on the live simulator on 2026-10-17** with the MCP tool `zengine_run` (console: `zengine run <file> --wait <ms>`); the `zengine desugar` outputs were captured with `run_console_command`. The files are in the repo under `tasks/zext/`.

Two ideas to keep in mind:

- **Sugar** is syntax with a plain-LSL equivalent: ternary, `??`, `|>`, indexing and slices, maps, f-strings, `switch`, `foreach`. It makes scripts shorter and does not change what they can do.
- **`->` and `await { }`** are different: they let a script wait for an `*Async` call in straight-line code, without writing the `dataserver` / `timer` plumbing by hand. They are how a script talks to a slow service (a model, a database, a web page) without blocking the region.

---

## 1. Value syntax

Boolean, ternary, `??`, pipeline, index and slice, map literal, compound assignment, raw, triple-quoted and adjacent strings, `%` formatting and `*` repetition on strings.

```lsl
// sugar_values.lsl - ZEXT value syntax: types, ternary, ??, |>, index and slice, maps, string forms, compound assignment.
default
{
    state_entry()
    {
        boolean ok = true;                       // 'boolean' is an integer; lowercase true / false are accepted
        integer a = 7;
        integer b = 3;
        integer biggest = a > b ? a : b;         // ternary
        string name = "";
        string who = name ?? "guest";            // ?? : blank string, NULL_KEY and JSON_INVALID count as missing
        string loud = "hello" |> llToUpper();    // pipeline: the left side becomes the FIRST argument
        llOwnerSay(f"ok={ok} biggest={biggest} who={who} loud={loud}");

        string s = "abcdefgh";
        llOwnerSay("s[0:3]=" + s[0:3] + " s[-1]=" + s[-1] + " s[2:]=" + s[2:] + " s[:1]=" + s[:1]);   // stop is INCLUSIVE

        list cfg = ["name" => "lamp", "level" => 3];     // map literal: a strided list
        llOwnerSay("cfg has " + (string)llGetListLength(cfg) + " entries, [0]=" + (string)cfg[0] + " [3]=" + (string)cfg[3]);

        integer flags = 1;
        flags |= 4;                              // compound bitwise assignment
        flags <<= 1;
        llOwnerSay("flags=" + (string)flags);    // (1|4)<<1 = 10

        string raw = r"C:\temp\x";               // raw: backslashes are literal
        string two = """first
second""";                                       // triple-quoted: may span lines
        string glued = "ab" "cd";                // adjacent literals concatenate
        llOwnerSay(raw + " | " + (string)llStringLength(two) + " chars | " + glued);

        llOwnerSay("%d logs of %s" % [3, "oak"]);   // string % list
        llOwnerSay("ab" * 3);                       // string * integer
    }
}
```

**Output** (run 2026-10-17):

```text
ok=TRUE biggest=7 who=guest loud=HELLO
s[0:3]=abcd s[-1]=h s[2:]=cdefgh s[:1]=ab
cfg has 4 entries, [0]=name [3]=3
flags=10
C:\temp\x | 12 chars | abcd
3 logs of oak
ababab
```

Things this run shows:

- **A slice stop is inclusive**: `s[0:3]` is `abcd` (four characters), like `llGetSubString(s, 0, 3)`. `s[-1]` is the last character; `s[2:]` and `s[:1]` leave one end open.
- **A map is a strided list**: `["name" => "lamp", "level" => 3]` has 4 entries, key, value, key, value.
- **`??` treats a blank string as missing**, not only a null: `name ?? "guest"` gave `guest` for `""`.
- **`x |> f(y)` puts `x` first**: `"hello" |> llToUpper()` is `llToUpper("hello")`.
- `"%d logs of %s" % [3, "oak"]` needs a list on the right; `"ab" * 3` needs an integer.

## 2. Control flow

`switch` with fall-through, `continue` / `break`, `foreach`, list destructuring, state parameters and arguments.

```lsl
// sugar_control.lsl - switch with fall-through, break / continue, foreach, list destructuring, state parameters.
default
{
    state_entry()
    {
        integer i;
        for (i = 1; i <= 4; ++i)
        {
            switch (i)
            {
                case 1:
                    llOwnerSay("one");
                case 2:                          // falls through from case 1
                    llOwnerSay("one or two");
                    break;
                case 3:
                    llOwnerSay("three");
                    break;
                default:
                    llOwnerSay("other: " + (string)i);
            }
        }

        // continue and break in a while loop
        integer n = 0;
        while (n < 10)
        {
            ++n;
            if (n % 2 == 0) continue;            // skip even numbers
            if (n > 7) break;                    // stop after 7
            llOwnerSay("odd " + (string)n);
        }

        foreach (string fruit in ["apple", "pear", "plum"])
            llOwnerSay("fruit " + fruit);

        [string label, integer val] = ["level", 3];    // list destructuring
        llOwnerSay(label + "=" + (string)val);

        state run(5);                            // state arguments
    }
}

state run(integer speed)                         // state parameters
{
    state_entry()
    {
        llOwnerSay("running at speed " + (string)speed);
    }
}
```

**Output** (run 2026-10-17):

```text
one
one or two
one or two
three
other: 4
odd 1
odd 3
odd 5
odd 7
fruit apple
fruit pear
fruit plum
level=3
running at speed 5
```

Case 1 prints `one` and then falls into case 2 (`one or two`), exactly like C; case 2 reached directly prints only `one or two`.

**Two traps found while writing it**

- **`key` is a type, not a name.** A first version used `[string key, integer val] = ...`; the compiler answered with ten errors starting `list unpacking currently requires explicit type declarations for all variables`, which points at the wrong thing. Renaming the variable fixed all of them. In LSL `key`, `list`, `vector`, `string`... cannot be identifiers.
- **Destructuring needs a type for every variable**: `[string label, integer val] = ...`, never `[label, val] = ...`.

## 3. What the sugar turns into

`zengine desugar <file>` rewrites a script to stock LSL. The value example above, minus its last two lines (see below), becomes:

```lsl
integer ok = 1;
integer __zte0;
if ((a > b)) __zte0 = a; else __zte0 = b;
integer biggest = __zte0;
string __zte1;
if ((((name != "") && (name != "00000000-0000-0000-0000-000000000000")) && (name != "﷐"))) __zte1 = name; else __zte1 = "guest";
string who = __zte1;
string loud = llToUpper("hello");
...
llGetSubString(s, 0, 3)        // s[0:3]
llGetSubString(s, (-1), (-1))  // s[-1]
llGetSubString(s, 2, -1)       // s[2:]
list cfg = ["name", "lamp", "level", 3];
(flags = (flags | 4));
string raw = "C:\\temp\\x";
string two = "first\nsecond";
string glued = ("ab" + "cd");
```

- **`%` and `*` on strings cannot be lowered.** `zengine desugar` refuses a script that uses them: `string '%' formatting has no stock-LSL equivalent (it would need a synthesized helper function)`, and the same for `*`. That is why `tasks/zext/sugar_lowerable.lsl` is `sugar_values.lsl` without those two lines.
- **Lowering is not always identical.** `zengine desugarcheck` runs original and lowered source and compares their output. For `sugar_lowerable.lsl` it reports **FAIL**: the original prints `ok=TRUE`, the lowered one prints `ok=1`, because `boolean` becomes a plain `integer`, and an f-string prints a ZEXT boolean as `TRUE` but the integer as `1`. Everything else matched.

## 4. The `->` bind

`call(...) -> name;` fires an `*Async` function, suspends the script, and resumes with the reply in `name`. Events keep being delivered while it waits; the engine matches the reply to the request key, so concurrent scripts and binds never see each other's answers.

```lsl
// async_basic.lsl - the -> bind: call an *Async function and continue with its result, in straight-line code.
// The script is suspended at every bind (other events keep being delivered); the engine matches the reply
// to the request key, so two scripts, or two binds, never see each other's answers.
default
{
    state_entry()
    {
        // 1) plain bind: the variable is declared by the bind
        osCacheSetAsync("demo", "greeting", "hello", 60.0) -> string stored;
        llOwnerSay("stored: " + stored);

        // 2) with a timeout: if nothing arrives within 5 s, run the else branch
        osCacheGetAsync("demo", "greeting") -> string got : 5.0 else
        {
            llOwnerSay("no answer in 5 s");
            return;
        }
        llOwnerSay("got: " + got);

        // 3) a timeout that is too short: the else branch runs and the script carries on from there
        osJevNoulAsync("Water is wet.", "Is the statement true?") -> string p : 0.01 else
        {
            llOwnerSay("gave up after 0.01 s (the model needs about 0.3 s)");
            return;
        }
        llOwnerSay("unexpected: " + p);
    }
}
```

**Output** (run 2026-10-17):

```text
stored: 1
got: hello
gave up after 0.01 s (the model needs about 0.3 s)
```

- `-> string stored` declares the variable; `-> name` reuses one already declared.
- `: 5.0 else { ... }` bounds the wait. If the reply does not arrive in time, the `else` branch runs instead.
- Not exercised here: multi-value binds `-> (a, b)` / `-> [a, b]` (the payload is split on commas) and `-> type name` inside user functions; see [[ZEXT Language Extensions]] section 2.1.

The same bind, lowered to stock LSL (`tasks/jev/jev_async_bind.lsl`): a global `__zstate`, a saved request key, `llSetTimerEvent(20.0)` for the timeout, and handlers `dataserver` and `http_response` that compare the key and continue, plus a `timer()` that runs the `else` branch. This is the state machine you would otherwise write by hand.

## 5. `await { }`: calls in parallel

Several calls inside `await { }` are all fired immediately; the script resumes when **every** one has answered, or runs the `else` branch if the optional timeout `: seconds` expires first.

```lsl
// async_parallel.lsl - await { ... }: fire several async calls at once and resume when ALL have answered.
// The same three decision-model calls, first one after the other with ->, then together with await.
string Q = """{"intent":{"type":"choice","instructions":"What does the customer want?","criteria":{"refund":"asks for money back","other":"anything else"}}}""";
string TICKET = "My parcel arrived two weeks late and the box was crushed. I want my money back.";

string pick(string r)
{
    return llJsonGetValue(r, ["answers", "intent", "choice"]);
}

default
{
    state_entry()
    {
        string a; string b; string c;
        float t0;

        // one after the other: the waits add up
        t0 = llGetTime();
        osJevAskExtAsync(TICKET, Q, "typesafe/jev-1.14", "") -> a;
        osJevAskExtAsync(TICKET, Q, "subq-snap-preview", "") -> b;
        osJevAskExtAsync(TICKET, Q, "perplexity/pplx-decider-flash-v1.2", "") -> c;
        llOwnerSay("sequential: " + llGetSubString((string)(llGetTime() - t0), 0, 4) + " s -> "
                   + pick(a) + " " + pick(b) + " " + pick(c));

        // all at once: the wait is the slowest one; the optional  : timeout else { }  applies to the whole block
        string pa; string pb; string pc;
        t0 = llGetTime();
        await {
            osJevAskExtAsync(TICKET, Q, "typesafe/jev-1.14", "") -> pa;
            osJevAskExtAsync(TICKET, Q, "subq-snap-preview", "") -> pb;
            osJevAskExtAsync(TICKET, Q, "perplexity/pplx-decider-flash-v1.2", "") -> pc;
        } : 15.0 else
        {
            llOwnerSay("not all answers arrived in 15 s");
            return;
        }
        llOwnerSay("parallel:   " + llGetSubString((string)(llGetTime() - t0), 0, 4) + " s -> "
                   + pick(pa) + " " + pick(pb) + " " + pick(pc));
    }
}
```

**Output** (run 2026-10-17):

```text
sequential: 1.096 s -> refund refund refund
parallel:   0.473 s -> refund refund refund
```

Three decision-model calls (Jev 1.14, SubQ, Perplexity flash) took 1.096 s one after the other and 0.473 s together; the parallel time is about that of the slowest call, not the sum. It is a single run, and the gain depends on how different the call durations are.

How it is lowered (`zengine desugar`): the three requests are sent in one go and each reply sets a flag, and the handler continues only when all flags are set:

```lsl
(__zgot_4_0 = 0);  (__zreq_4_0 = osJevAskExtAsync(TICKET, Q, "typesafe/jev-1.14", ""));
(__zgot_4_1 = 0);  (__zreq_4_1 = osJevAskExtAsync(TICKET, Q, "subq-snap-preview", ""));
(__zgot_4_2 = 0);  (__zreq_4_2 = osJevAskExtAsync(TICKET, Q, "perplexity/pplx-decider-flash-v1.2", ""));
llSetTimerEvent(15.0);
...
if (queryid == __zreq_4_1) { (__zgot_4_1 = 1); (pb = data); }
if (((__zgot_4_0 && __zgot_4_1) && __zgot_4_2)) { llSetTimerEvent(0.0); ... continue ... }
```

The three sequential binds before it became a chain of states (`__zstate` 1, 2, 3), each handler sending the next request. So many binds in a row work in `state_entry`; the documented limit is that **lowering** refuses a bind inside a loop, an `if` / `else` or a user function (the engine itself runs those).

## 6. Named timers

`timer "name" -> seconds: { body }` declares an independent cadence where it is used. Named timers coexist with each other and with an explicit `timer()`. (Unnamed `timer -> n: { }` cannot be mixed with an explicit `timer()` in the same state.)

```lsl
// timers.lsl - named timers: several independent cadences in one state, written where they are used.
// timer "name" -> seconds: { body }   is hoisted into its own handler; named timers coexist with each other
// and with an explicit timer() event. Run with a wait of about 4500 ms.
integer fast;
integer slow;

default
{
    state_entry()
    {
        llOwnerSay("start");

        timer "fast" -> 0.5:
        {
            ++fast;
            llOwnerSay("fast tick " + (string)fast);
        }

        timer "slow" -> 1.5:
        {
            ++slow;
            llOwnerSay("slow tick " + (string)slow + " (fast count so far " + (string)fast + ")");
        }
    }
}
```

**Output** (run 2026-10-17):

```text
start
fast tick 1
fast tick 2
fast tick 3
slow tick 1 (fast count so far 3)
fast tick 4
fast tick 5
fast tick 6
slow tick 2 (fast count so far 6)
fast tick 7
fast tick 8
```

The 0.5 s and 1.5 s cadences stay in step (three fast ticks per slow tick) in the 4.5 s the run allowed.

The lowered form is **not self-contained stock LSL**: it still calls `__zSetNamedTimer("fast", 0.5)` in `state_entry`, a compiler-internal function that the output does not define, and its `timer()` reads `__ztimer_fast_next` variables that nothing sets without it. Treat `zengine desugar` of named timers as a view of the structure, not as a script to run elsewhere.

## 7. Rate limits when you try these

The three-model examples call the decision endpoints; `perplexity/pplx-decider-v1.2` (the default model) answers `HTTP 429` on bursts, so run them one at a time. These use other models and did not hit it. See [[Known and Unresolved Problems]].

## 8. Writing JSON without escaping every quote

The `osJev*` and `osDb*` functions take JSON as text, and in plain LSL that means a backslash in front of every quote, which is hard to write and harder to read. Two pieces of ZEXT sugar remove it. Both were run on 2026-10-17 and give the same question, which the engine accepts (`tasks/zext/json_readable.lsl`, which also builds the old escaped form to compare against).

```lsl
// json_readable.lsl - three ways to write the same JSON question, from the awkward one to the readable ones.
default
{
    state_entry()
    {
        // (the repo script also builds the old escaped form as "old" and compares each result with it)

        // 1) triple-quoted string: quotes need no escaping, and it may span lines
        string a = """{"intent":{"type":"choice","instructions":"What does the customer want?","criteria":{"refund":"asks for money back","other":"anything else"}}}""";

        // 2) the same, laid out over several lines
        string b = """
        {
          "intent": {
            "type": "choice",
            "instructions": "What does the customer want?",
            "criteria": {
              "refund": "asks for money back",
              "other":  "anything else"
            }
          }
        }
        """;

        // 3) map literals (=>) plus llList2Json: no quotes to escape, no braces to balance
        string c = llList2Json(JSON_OBJECT, [
            "intent" => llList2Json(JSON_OBJECT, [
                "type"         => "choice",
                "instructions" => "What does the customer want?",
                "criteria"     => llList2Json(JSON_OBJECT, [
                    "refund" => "asks for money back",
                    "other"  => "anything else"
                ])
            ])
        ]);

        llOwnerSay("1 triple-quoted  identical to old: " + (string)(a == old));
        llOwnerSay("2 multi-line     identical to old: " + (string)(b == old) + " (length " + (string)llStringLength(b) + " vs " + (string)llStringLength(old) + ")");
        llOwnerSay("3 llList2Json    identical to old: " + (string)(c == old));
        llOwnerSay("3 text: " + c);

        // all four must be accepted by the real thing
        string t = "My parcel arrived two weeks late. I want my money back.";
        llOwnerSay("a   -> " + llJsonGetValue(osJevAskExt(t, a,   "typesafe/jev-1.14", ""), ["answers", "intent", "choice"]));
        llOwnerSay("b   -> " + llJsonGetValue(osJevAskExt(t, b,   "typesafe/jev-1.14", ""), ["answers", "intent", "choice"]));
        llOwnerSay("c   -> " + llJsonGetValue(osJevAskExt(t, c,   "typesafe/jev-1.14", ""), ["answers", "intent", "choice"]));
    }
}
```

**Output** (run 2026-10-17; `old` is the escaped form in the repo script):

```text
1 triple-quoted  identical to old: 1
2 multi-line     identical to old: 0 (length 280 vs 142)
3 llList2Json    identical to old: 1
3 text: {"intent":{"type":"choice","instructions":"What does the customer want?","criteria":{"refund":"asks for money back","other":"anything else"}}}
old -> refund
a   -> refund
b   -> refund
c   -> refund
```

- **Triple-quoted string (`"""..."""`)**: quotes inside need no escape and the text may span lines. On one line the result is **byte-identical** to the escaped version. Laid out over several lines it differs only by whitespace (280 characters against 142), which JSON ignores. Note that escapes such as `\t` are still processed inside it; use a raw string for literal backslashes.
- **Map literal plus `llList2Json`**: `["type" => "choice", ...]` is a strided list and `llList2Json(JSON_OBJECT, ...)` turns it into JSON text. Nested objects are built by nesting the calls. The result is **identical** to the escaped string, compact, with no quotes or braces to balance. The calls run at run time, so build the string in an event, not in a global initialiser.
- Which to use: triple-quoted when you want the JSON visible as JSON (copy it from documentation); `llList2Json` when parts of it are computed. An f-string is a poor fit for JSON, because its braces are interpolation.
- There is no dedicated JSON literal (something like `json { intent: { type: "choice" } }`); it would be a change to the ZEngine parser.

All the other examples in the vault use these two forms. One example per ZEXT construct, 29 in all, is in [[ZEXT Constructs - One Example Each]].
