---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - async
  - closures
aliases:
  - "->"
  - "=>"
  - async bind
  - fat arrow
  - closures
---

# ZEXT Arrows: `->` async bind, `=>` maps and closures, `timer ->`

← Back to [[ZEXT Syntax Sugar Index]] | Reference: [[ZEXT Language Extensions]] (§2 has the runtime internals) | Engine: [[ZEngine]] | Output: [[ZEXT Desugar - Stock LSL Output]] | Checked by: [[ZEXT Desugar Verification Report]]

Three different arrows, easy to confuse:

| Arrow | Where | Meaning |
| :--- | :--- | :--- |
| `call(...) -> name` | statement | **async bind**: suspend the handler until that request's reply arrives, bind the payload |
| `timer -> N:` / `timer "x" -> N:` | statement | **inline timer**: arm a timer here, the indented block is its handler |
| `["k" => v]` | inside `[ ]` | **map literal**: notation for a flat key/value list |
| `(params) => { }` | last argument of 3 calls | **closure** (a callback) |
| `x \|> f()` | expression | pipeline (not an arrow, listed for completeness; see [[ZEXT Language Extensions]]) |

---

## 1. `->` the async bind

Many OpenSim/CraftSim calls cannot return their result; they return a **request key** and answer later in a `dataserver` (or `http_response`) event. `f(...) -> x` calls `f`, **suspends the rest of the event handler**, and resumes it, with the payload in `x`, when *that* request's reply arrives. No hand-written event handler, no key matching.

| Form | Meaning |
| :--- | :--- |
| `f() -> x` | `x` is a string |
| `f() -> string x` | typed name (the payload is still text; the type is declarative) |
| `f() -> (a, b)` or `-> [a, b]` | the payload is CSV-split in order; a name past the end gets `""` |
| `f() -> x : 2.5 else:` | give up after 2.5 s and run the `else` block, then **carry on** after the statement (use `return` inside it to stop) |
| `await:` + `call -> x` lines | fire several requests together, resume when **all** have answered (brace syntax: `await { … } : 15.0 else { … }`) |

It works in an event handler, **inside `if`/`while`/`for`/`switch` bodies**, and **inside user functions** (those are expanded into their callers). Recursion through a suspending function is a compile error. Natively, a waiting handler resumes selectively: the engine takes only the reply with the matching request key out of the event queue and leaves every other queued event for normal dispatch afterwards (`TryConsumeMatchingEvent`).

> [!INFO] Where the replies come from
> `osMLpredictAsync("no-such-model", …)` answers through `dataserver` with an error text and needs no network, so it is the test source below (the server needs `Allow_osMLpredictAsync`). `llHTTPRequest` to a closed local port stands in for a request that times out. See [[ZEXT Language Extensions]] §2.1 for `listen` and `http_response` binds.

### The annotated example

```lsl
#indent
// =============================================================================================
// sugar_02_async_arrows.lsl  -  the async-bind arrow  `call(...) -> name`
//
// Many OpenSim/CraftSim calls do not return their result: they return a request key and deliver the
// answer later in a `dataserver` (or `http_response`) event.  `f(...) -> x` calls f, SUSPENDS the rest
// of the event handler, and resumes it - with the payload bound to x - when THAT request's reply
// arrives.  No hand-written dataserver handler, no key matching.
//
//   f() -> x                      x is a string
//   f() -> string x               typed name (the payload is still text; the type is declarative)
//   f() -> (a, b)                 the payload is CSV-split across the names, in order
//   f() -> x : 2.5 else:          give up after 2.5 s and run the else block, then carry on
//   await: / call -> x lines      fire several requests at once, resume when ALL have answered
//
// Works inside if / while / for / switch bodies and inside user functions (those are expanded into
// their callers).  Recursion through a suspending function is a compile error.
//
// Here the async source is osMLpredictAsync("no-such-model", ...): it answers with an error text, so no
// network is needed (the server needs Allow_osMLpredictAsync).  llHTTPRequest to a closed local port
// stands in for a request that times out.
//
// `zengine desugar` turns all of this into stock LSL: see sugar_02_async_arrows in the docs for how.
// =============================================================================================

// A user function that suspends - fine: it is inlined into the handlers that call it.
string ask(string tag, float x = 1.0):
    osMLpredictAsync("no-such-model", [x]) -> string answer
    return tag + ":" + (string)(answer != "")

default:
    touch_start(integer n):
        // 1. single bind
        osMLpredictAsync("m", [1.0]) -> string r
        llOwnerSay("1 single " + (string)(r != ""))

        // 2. destructuring bind: the payload is split on commas; surplus names get ""
        osMLpredictAsync("m", [2.0]) -> (a, b)
        llOwnerSay("2 destructure second-empty=" + (string)(b == ""))

        // 3. a timeout and an else block (no return in it, so execution continues below)
        llHTTPRequest("http://127.0.0.1:59999/x", [], "") -> string body : 0.4 else:
            llOwnerSay("3 timed out")
        llOwnerSay("3 continues after the else block")

        // 4. inside a loop, an if/else, and through the suspending function above
        integer i
        for i = 0; i < 2; i++:
            if i == 1:
                llOwnerSay("4 loop " + ask("via-function"))
            else:
                osMLpredictAsync("m", [3.0]) -> string t
                llOwnerSay("4 loop " + (string)i + " " + (string)(t != ""))

        // 5. await: both requests are in flight together; resume when both replies are in
        await:
            osMLpredictAsync("m", [4.0]) -> string x
            osMLpredictAsync("m", [5.0]) -> string y
        llOwnerSay("5 await " + (string)(x != "") + (string)(y != ""))
```

What it printed, identically as written and as desugared:

```text
1 single 1
2 destructure second-empty=1
3 timed out
3 continues after the else block
4 loop 0 1
4 loop via-function:1
5 await 11
```

### How `zengine desugar` lowers it

Stock LSL cannot suspend. When a bind sits directly in the handler body the handler is split into segments (the old "level 2" transform); when it sits **inside a loop, an `if`, a `switch` or an inlined function** the handler is **flattened into numbered blocks run by a generated dispatch function**:

```text
touch_start(n)  ──►  copies its parameters to globals, calls __zrun1_touch_start(0)
__zrunN(pc)         while (1) { if (pc==0) {…; pc = 3;} else if (pc==3) {…start request…; __zstate = K; return;} … }
dataserver / http_response / timer  ──►  match __zstate and the request key, bind the payload,
                                          __zstate = 0, call __zrunN(<block after the request>)
```

- every local, event parameter and bound name that lives across a suspension becomes a **uniquely named global** (`__zl<n>_<name>`), so shadowing and re-entered loops keep their meaning;
- `if`/`while`/`for`/`switch` become `pc` assignments; `break`/`continue`/`jump` become a `pc` change plus `jump __zdispN` (a label at the end of the loop body);
- the timeout `else` is flattened into the same graph, so a `return`, `continue` or `jump` inside it works;
- `state x;` inside such code cannot be a function body statement in LSL, so it sets `__zpend` and the **caller** performs the state change.

The generated dispatch function for the example (every block is one `if (__zpc == n)` arm; the arms that end in `(__zstate = K); return;` are the suspension points):

```lsl
__zrun1_touch_start(integer __zpc)
{
    while (1)
    {
        if ((__zpc == 0))
        {
            (__zreq_1 = osMLpredictAsync("m", [1.0]));
            (__zstate = 1);
            return;
        }
        else if ((__zpc == 1))
        {
            llOwnerSay(("4 loop " + __zl8___hv0));
            (__zpc = 12);
        }
        else if ((__zpc == 2))
        {
            llOwnerSay(("1 single " + ((string)(__zl3_r != ""))));
            (__zreq_2 = osMLpredictAsync("m", [2.0]));
            (__zstate = 2);
            return;
        }
        else if ((__zpc == 3))
        {
            llOwnerSay(("2 destructure second-empty=" + ((string)(__zl5_b == ""))));
            (__zreq_3 = llHTTPRequest("http://127.0.0.1:59999/x", [], ""));
            llSetTimerEvent(0.4);
            (__zstate = 3);
            return;
        }
        else if ((__zpc == 4))
        {
            llOwnerSay("3 continues after the else block");
            (__zl7_i = 0);
            (__zl7_i = 0);
            (__zpc = 6);
        }
        else if ((__zpc == 5))
        {
            {
                llOwnerSay("3 timed out");
            }
            (__zpc = 4);
        }
        else if ((__zpc == 6))
        {
            if ((__zl7_i < 2))
            {
                (__zpc = 7);
            }
            else
            {
                (__zpc = 9);
            }
        }
        else if ((__zpc == 7))
        {
            if ((__zl7_i == 1))
            {
                (__zpc = 10);
            }
            else
            {
                (__zpc = 11);
            }
        }
        else if ((__zpc == 8))
        {
            (__zl7_i++);
            (__zpc = 6);
        }
        else if ((__zpc == 9))
        {
            (__zgot_6_0 = 0);
            (__zreq_6_0 = osMLpredictAsync("m", [4.0]));
            (__zgot_6_1 = 0);
            (__zreq_6_1 = osMLpredictAsync("m", [5.0]));
            (__zstate = 6);
            return;
        }
        else if ((__zpc == 10))
        {
            (__zl8___hv0 = "");
            (__zl9___i1_ask_tag = "via-function");
            (__zl10___i1_ask_x = 1.0);
            (__zreq_4 = osMLpredictAsync("no-such-model", [__zl10___i1_ask_x]));
            (__zstate = 4);
            return;
        }
        else if ((__zpc == 11))
        {
            (__zreq_5 = osMLpredictAsync("m", [3.0]));
            (__zstate = 5);
            return;
        }
        else if ((__zpc == 12))
        {
            (__zpc = 8);
        }
        else if ((__zpc == 13))
        {
            (__zl8___hv0 = ((__zl9___i1_ask_tag + ":") + ((string)(__zl11___i1_ask_answer != ""))));
            {
                (__zpc = 1);
                jump __zdisp1;
            }
            (__zpc = 1);
        }
        else if ((__zpc == 14))
        {
            llOwnerSay(((("4 loop " + ((string)__zl7_i)) + " ") + ((string)(__zl12_t != ""))));
            (__zpc = 12);
        }
        else if ((__zpc == 15))
        {
            llOwnerSay((("5 await " + ((string)(__zl13_x != ""))) + ((string)(__zl14_y != ""))));
            return;
        }
        else
        {
            return;
        }
        @__zdisp1;
    }
}
```

and the handlers that resume it:

```lsl
    dataserver(key queryid, string data)
    {
        if (((__zstate == 1) && (queryid == __zreq_1)))
        {
            (__zl3_r = data);
            (__zstate = 0);
            __zrun1_touch_start(2);
            return;
        }
        if (((__zstate == 2) && (queryid == __zreq_2)))
        {
            list __zsplit_2_data = llCSV2List(data);
            (__zl4_a = llList2String(__zsplit_2_data, 0));
            (__zl5_b = llList2String(__zsplit_2_data, 1));
            (__zstate = 0);
            __zrun1_touch_start(3);
            return;
        }
        if (((__zstate == 3) && (queryid == __zreq_3)))
        {
            llSetTimerEvent(0.0);
            (__zl6_body = data);
            (__zstate = 0);
            __zrun1_touch_start(4);
            return;
        }
        if (((__zstate == 4) && (queryid == __zreq_4)))
        {
            (__zl11___i1_ask_answer = data);
            (__zstate = 0);
            __zrun1_touch_start(13);
            return;
        }
        if (((__zstate == 5) && (queryid == __zreq_5)))
        {
            (__zl12_t = data);
            (__zstate = 0);
            __zrun1_touch_start(14);
            return;
        }
        if ((__zstate == 6))
        {
            if ((queryid == __zreq_6_0))
            {
                (__zgot_6_0 = 1);
                (__zl13_x = data);
            }
            if ((queryid == __zreq_6_1))
            {
                (__zgot_6_1 = 1);
                (__zl14_y = data);
            }
            if ((__zgot_6_0 && __zgot_6_1))
            {
                (__zstate = 0);
                __zrun1_touch_start(15);
                return;
            }
        }
    }
```

```lsl
    timer()
    {
        if ((__zstate == 3))
        {
            (__zstate = 0);
            llSetTimerEvent(0.0);
            __zrun1_touch_start(5);
            return;
        }
    }
```

> [!WARNING] Two honest limits of the lowered form
> - Event parameters become globals when the handler is entered, so `llDetectedKey(0)` and friends are only meaningful **before** the first suspension (after it the dataserver event has no detection data). ZEngine's own `->` has the same property.
> - The lowered handler runs only when nothing is pending (`if (__zstate == 0)`), so a second trigger of the same event **while it waits is dropped**; native `->` leaves such events queued and runs them after the handler resumes. Single-flight scripts are unaffected. (Read from the code, not measured.)

---

## 2. `=>` the fat arrow: two unrelated jobs

### 2.1 Map literal `["key" => value]`

Notation only. `["red" => 1, "green" => 2]` **is** the flat list `["red", 1, "green", 2]`; read it back with `llListFindList` and `llList2<T>` at stride 2. It can appear anywhere a list literal can (initialisers, `+=`, arguments). The tokenizer/parser produce a plain `ListLit`, so desugar prints an ordinary list. ZEXT gate: `the map '=>' arrow syntax`.

### 2.2 Closure `(params) => { body }`

A closure is accepted as the **last argument of exactly three calls**:

| Call | Closure parameters | Becomes |
| :--- | :--- | :--- |
| `llSetTimerEvent(seconds, closure)` | none | `__z_call_timer(sec, id)` + a branch of `timer()` |
| `llListen(chan, name, id, msg, closure)` | up to four: `(integer c, string n, key k, string text)`, bound in order to the event's `chan name id msg` | `__z_call_listen(…, id)` + a branch of `listen()` |
| `llSensor(name, id, type, range, arc, closure)` | one: `(integer found)`, bound to `num_detected` | `__z_call_sensor(…, id)` + a branch of `sensor()` |

The parser **expands the closure at parse time** into an ordinary helper function, a global callback id (`__ztimer_cb`, `__zlisten_cb`, `__zsensor_cb`) and an `if (__z…_cb == n) { body }` appended to that state's event handler. Consequences:

- the body runs **in the event handler**, so it sees globals and its own parameters but **not the enclosing handler's locals** (ZEngine has no environment capture);
- a closure anywhere else is a compile error, and a closure outside any state now gives `a closure can only be written inside an event handler of a state` instead of a crash;
- inside a braceless file the closure is brace-delimited and needs its own semicolons (indentation rules are off inside `( )`);
- because the expansion happens in the parser, **desugar needs no closure support of its own**: it only ever sees the helper function and the event branch.

```lsl
#indent
// =============================================================================================
// sugar_03_fat_arrow.lsl  -  the two jobs of  =>
//
// (1) map literal:  ["key" => value, ...]  is just a flat list  ["key", value, ...]  (a strided list).
//     It is notation only: use llListFindList / llList2<T> with stride 2 to read it back.
//
// (2) closure:  ( [type name, ...] ) => { body }  as the LAST argument of exactly three calls:
//         llSetTimerEvent(seconds, closure)
//         llListen(chan, name, id, msg, closure)      closure params = (chan, name, id, msg), as many as you list
//         llSensor(name, id, type, range, arc, closure)   closure param  = (num_detected)
//     The parser rewrites each into an ordinary helper function + a global callback id + an
//     `if (id == n)` branch added to the timer()/listen()/sensor() handler.  So a closure body runs in
//     that event handler: it can use globals and its own parameters, NOT the enclosing handler's locals.
//     A closure anywhere else is a compile error.  A closure body is brace-delimited (it sits inside
//     parentheses, where the indentation rules are switched off) and needs its own semicolons.
// =============================================================================================

integer ticks
list palette = ["red" => 1, "green" => 2, "blue" => 3]

integer lookup(list m, string name):
    integer at = llListFindList(m, [name])
    if at < 0:
        return -1
    return llList2Integer(m, at + 1)

default:
    state_entry():
        // a listen closure: registers a callback; nothing in this test ever says anything on channel 5
        llListen(5, "", NULL_KEY, "", (integer ch, string who, key src, string text) => { llOwnerSay("heard " + text); })

    touch_end(integer n):
        // a sensor closure (a scan only happens on touch_end, which the test never sends)
        llSensor("", NULL_KEY, AGENT, 10.0, PI, (integer found) => { llOwnerSay("sensed " + (string)found); })

    touch_start(integer n):
        // map literal
        llOwnerSay(llList2CSV(palette))
        llOwnerSay("green=" + (string)lookup(palette, "green") + " mauve=" + (string)lookup(palette, "mauve"))
        palette += ["alpha" => 9]
        llOwnerSay("pairs=" + (string)(llGetListLength(palette) / 2))

        // timer closure: runs in timer() every 0.5 s until it switches the timer off
        llSetTimerEvent(0.5, () => { ticks++; llOwnerSay("tick " + (string)ticks); if (ticks >= 2) llSetTimerEvent(0.0); })
```

Output (as written = desugared):

```text
red, 1, green, 2, blue, 3
green=2 mauve=-1
pairs=4
tick 1
tick 2
```

The listen closure after desugaring: the helper, the callback id, and the handler branch:

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
            llOwnerSay(("heard " + text));
        }
    }
```

---

## 3. `timer ->` inline and named timers

`timer -> 0.5:` arms `llSetTimerEvent(0.5)` **at that statement** and moves the indented block into the state's `timer()`. Several unnamed inline timers append in source order; mixing one with a hand-written `timer()` is a compile error (LSL has one timer). `timer "name" -> 2.0:` is a **named timer**: its own cadence, any number of them, they coexist with a hand-written `timer()` and with each other (native scheduling: `ZEngine.m_namedTimerHeap`).

```lsl
#indent
// =============================================================================================
// sugar_10_inline_timers.lsl  -  `timer -> N:`  (an arrow, again)
//
//   timer -> 0.5:            arms llSetTimerEvent(0.5) right here and moves the indented body into
//       body                 this state's timer() handler.  Several unnamed inline timers append in
//                            source order; mixing one with a hand-written timer() is a compile error
//                            (LSL has a single timer).
//   timer "name" -> 2.0:     a NAMED timer: its own cadence, any number of them, and they coexist with a
//       body                 hand-written timer().
//
// In #indent form the header line ends with ':' and the body is the indented block.
// =============================================================================================

integer n1

default:
    state_entry():
        timer -> 0.5:
            n1++
            llOwnerSay("tick " + (string)n1)
            if n1 >= 3:
                llSetTimerEvent(0.0)
```

```text
tick 1
tick 2
tick 3
```

### Named timers after desugaring

Stock LSL has one timer, so the lowered script keeps a **due time per name** and one `timer()` polls them at the **finest interval armed so far** (`__ztimer_tick`). The due times stay on a fixed grid (`next += interval`) so polling jitter cannot skip a period. Source and result:

```lsl
// desugar-quiet: 9000
// desugar-timeout: 7000
// (ZDesugarTest.TestDesugarSuiteIsEquivalent watches 7 s here so the pulses at 2, 4 and 6 s are compared.)
// A named timer: "pulse" fires every 2.0 s, independently of any other timer().  Desugared, stock LSL
// has a single timer, so the output keeps a due time per name and one timer() polls them (at the
// finest interval armed so far).
default { state_entry()
{
    llOwnerSay("started");
    timer "pulse" -> 2.0: {
        llOwnerSay("pulse");
    };
} }
```

```text
started
pulse
pulse
pulse
```

```lsl
// Desugared from ZEXT to stock LSL by `zengine desugar`.
// Generated file: comments and formatting from the original are not preserved.

integer __zstate = 0;
float __ztimer_pulse_int = 0.0;
float __ztimer_pulse_next = 0.0;
float __ztimer_tick = 0.0;

default
{
    state_entry()
    {
        llOwnerSay("started");
        __ztimer_pulse_int = 2.0;
        __ztimer_pulse_next = llGetTime() + __ztimer_pulse_int;
        if ((__ztimer_tick == 0.0) || (__ztimer_pulse_int < __ztimer_tick)) __ztimer_tick = __ztimer_pulse_int;
        llSetTimerEvent(__ztimer_tick);
        {
        }
    }
    timer()
    {
        if (((__ztimer_pulse_next > 0.0) && (llGetTime() >= __ztimer_pulse_next)))
        {
            (__ztimer_pulse_next = (__ztimer_pulse_next + __ztimer_pulse_int));
            if ((__ztimer_pulse_next <= llGetTime()))
            {
                (__ztimer_pulse_next = (llGetTime() + __ztimer_pulse_int));
            }
            {
                llOwnerSay("pulse");
            }
        }
    }
}
```

> [!WARNING] Found and fixed while writing this page
> The first version of this note found the named-timer lowering **never fired**: `zengine desugar` left the compiler-internal `__zSetNamedTimer("pulse", 2.0)` call in the output (which ZEngine itself still understood, so the old test, which only checked that the output compiled, passed) and nothing ever set the `…_next` due time. It now emits real statements (`__ztimer_pulse_int = 2.0; __ztimer_pulse_next = llGetTime() + …; llSetTimerEvent(__ztimer_tick);`). The suite test compares the three pulses at 2, 4 and 6 s. Cadence in the lowered form is only as fine as the shortest interval armed.

See also: [[ZEXT Braceless Indent Mode]] (the `await:` / `else:` header forms), [[ZEXT Syntax Sugar Index]].
