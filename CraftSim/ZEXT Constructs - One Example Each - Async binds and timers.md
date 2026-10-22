---
tags:
  - craftsim
  - zext
  - zengine
  - examples
  - constructs
---

# ZEXT Constructs - One Example Each - Async binds and timers

> Part of [[ZEXT Constructs - One Example Each]]. Sections: `->` binds, `await { }`, named and inline timers.

## 24. The `->` bind in a function

A memoised classifier: three binds in a row inside an ordinary function (cache lookup, model call with a timeout, cache store). Round 2 never touches the model.

```lsl
// c24_bind_memo.lsl - the -> bind inside a user function: a memoised classifier.
// classify() looks in the cache first and only asks the model on a miss, then stores the answer.
// Three binds in a row, in a function, read like ordinary synchronous code.
string QUESTION = """{"intent":{"type":"choice","instructions":"What is the visitor doing?","criteria":{"greet":"says hello","ask":"asks a question","complain":"is unhappy","other":"anything else"}}}""";

string classify(string text)
{
    osCacheGetAsync("memo", text) -> string hit;                  // 1) cache lookup ("" on a miss)
    if (hit != "") return hit + " (cache)";

    osJevAskExtAsync(text, QUESTION, "typesafe/jev-1.14", "") -> string reply : 15.0 else
    {
        return "no answer";                                         // 2) the model, bounded by a timeout
    }
    string label = llJsonGetValue(reply, ["answers", "intent", "choice"]);
    osCacheSetAsync("memo", text, label, 300.0) -> string stored;   // 3) remember it for five minutes
    return label + " (model)";
}

default
{
    state_entry()
    {
        list lines = ["Good morning everyone!", "Where can I find the library?", "This lag is unbearable."];
        integer round;
        for (round = 1; round <= 2; ++round)
        {
            float t0 = llGetTime();
            string out = "";
            foreach (string line in lines)
                out += classify(line) + " | ";
            llOwnerSay("round " + (string)round + " in " + llGetSubString((string)(llGetTime() - t0), 0, 4) + " s: " + out);
        }
    }
}
```

**Output** (run 2026-10-17):

```text
round 1 in 1.034 s: greet (model) | ask (model) | complain (model) |
round 2 in 0.001 s: greet (cache) | ask (cache) | complain (cache) | 
```

## 25. `: seconds else { }` in a retry loop

Asking a model with a patience that grows (0.1 s, 0.4 s, 0.9 s, 1.6 s). A reply that arrives after its attempt timed out is matched by request key and ignored, so it cannot be taken for the answer to the next attempt.

```lsl
// c25_bind_timeout.lsl - ": seconds else { }" in a retry loop: ask with a short patience that grows each time.
// A late reply to an attempt that already timed out is matched by request key and ignored, so it can never be
// mistaken for the answer to a later attempt.
string QUESTION = """{"intent":{"type":"choice","instructions":"What does the customer want?","criteria":{"refund":"asks for money back","other":"anything else"}}}""";

default
{
    state_entry()
    {
        string answer = "";
        integer attempt = 0;
        while (answer == "" && attempt < 4)
        {
            ++attempt;
            float patience = 0.1 * attempt * attempt;                // 0.1 s, 0.4 s, 0.9 s, 1.6 s
            float t0 = llGetTime();
            osJevAskExtAsync("My parcel never arrived. I want a refund.", QUESTION, "typesafe/jev-1.14", "")
                -> string reply : patience else
            {
                llOwnerSay(f"attempt {attempt}: no answer within {patience} s");
            }
            if (reply != "")
            {
                answer = llJsonGetValue(reply, ["answers", "intent", "choice"]);
                llOwnerSay(f"attempt {attempt}: '{answer}' after {llGetSubString((string)(llGetTime() - t0), 0, 4)} s");
            }
        }
        if (answer == "") llOwnerSay("gave up");
    }
}
```

**Output** (run 2026-10-17):

```text
attempt 1: no answer within 0.100000 s
attempt 2: no answer within 0.400000 s
attempt 3: 'refund' after 0.257 s
```

Attempt 2 timed out although the model usually answers in about 0.3 s: the budget is a real deadline, and latency varies run to run.

## 26. Binding several variables `-> (a, b, c)` / `-> [a, b]`

Records kept in the cache as `id,amount,currency` are read back into three typed variables with one bind; a missing key leaves them at their defaults. The bracket form does the same.

```lsl
// c26_bind_multi.lsl - a bind to several variables:  -> (a, b)  or  -> [a, b].
// The reply is split on commas in order and converted to each variable's type. Here a tiny order queue keeps
// "id,amount,currency" records in the cache, and each is read back into three typed variables at once.
default
{
    state_entry()
    {
        osCacheSetAsync("orders", "o-1001", "1001,249.9,EUR", 60.0) -> string stored1;
        osCacheSetAsync("orders", "o-1002", "1002,15.5,USD", 60.0) -> string stored2;

        float eur = 0.0;
        float usd = 0.0;
        foreach (string slot in ["o-1001", "o-1002", "o-9999"])
        {
            osCacheGetAsync("orders", slot) -> (integer id, float amount, string currency);
            if (id == 0)
                llOwnerSay(slot + ": not found");
            else
            {
                llOwnerSay(f"order {id}: {amount} {currency}");
                if (currency == "EUR") eur += amount; else usd += amount;
            }
        }
        llOwnerSay(f"open amounts: {eur} EUR and {usd} USD");

        // the same with the bracket form and a trailing text field that keeps its comma-free part only
        osCacheSetAsync("orders", "pt", "3,4", 60.0) -> string stored3;
        osCacheGetAsync("orders", "pt") -> [integer x, integer y];
        llOwnerSay(f"point ({x}, {y}), distance from origin squared {x * x + y * y}");
    }
}
```

**Output** (run 2026-10-17):

```text
order 1001: 249.900000 EUR
order 1002: 15.500000 USD
o-9999: not found
open amounts: 249.900000 EUR and 15.500000 USD
point (3, 4), distance from origin squared 25
```

## 27. `await { }`: calls in parallel

A majority vote over three models: all three calls leave together and the script resumes when every answer is in (or the timeout block runs). Each ticket took about the time of the slowest model, not the sum.

```lsl
// c27_await_quorum.lsl - await { }: ask three models at once and take a majority vote.
// All three calls leave together; the script resumes when every answer is in, or the timeout block runs.
string QUESTION = """{"intent":{"type":"choice","instructions":"What does the customer want?","criteria":{"refund":"asks for money back","replace":"asks for a replacement","info":"asks for information only","other":"anything else"}}}""";

string pick(string reply)
{
    return llJsonGetValue(reply, ["answers", "intent", "choice"]);
}

string vote(string text)
{
    string a; string b; string c;
    await
    {
        osJevAskExtAsync(text, QUESTION, "typesafe/jev-1.14", "") -> a;
        osJevAskExtAsync(text, QUESTION, "subq-snap-preview", "") -> b;
        osJevAskExtAsync(text, QUESTION, "perplexity/pplx-decider-flash-v1.2", "") -> c;
    } : 15.0 else
    {
        return "no quorum";
    }

    list ballots = [pick(a), pick(b), pick(c)];
    string winner = "";
    integer best = 0;
    integer i;
    for (i = 0; i < 3; ++i)
    {
        string candidate = llList2String(ballots, i);
        integer count = 0;
        integer j;
        for (j = 0; j < 3; ++j)
            if (llList2String(ballots, j) == candidate) ++count;
        if (count > best)
        {
            best = count;
            winner = candidate;
        }
    }
    return winner + " (" + (string)best + "/3: " + llList2CSV(ballots) + ")";
}

default
{
    state_entry()
    {
        list tickets = [
            "My parcel arrived broken. I want my money back.",
            "The lamp I received does not work, please send me a new one.",
            "What are your opening hours on Sunday?"
        ];
        foreach (string t in tickets)
        {
            float t0 = llGetTime();
            string result = vote(t);
            llOwnerSay(llGetSubString((string)(llGetTime() - t0), 0, 4) + " s  " + result);
        }
    }
}
```

**Output** (run 2026-10-17):

```text
0.432 s  refund (3/3: refund, refund, refund)
0.320 s  replace (3/3: replace, replace, replace)
0.311 s  info (3/3: info, info, info)
```

## 28. Named timers `timer "name" -> seconds: { }`

A worker that beats every 0.4 s and hangs after three beats, and a watchdog that checks every second whether the count moved. Two independent cadences in one state, written where they start.

```lsl
// c28_named_timers.lsl - timer "name" -> seconds: { body } : independent cadences in one state.
// A worker beats every 0.4 s but "hangs" after its third beat; a watchdog checks every 1.0 s that the beat count moved.
// Named timers coexist with each other and with an explicit timer() event. Run with a wait of about 4500 ms.
integer beats;
integer lastSeen;
integer healthy = TRUE;
integer alarms;

default
{
    state_entry()
    {
        llOwnerSay("start");

        timer "worker" -> 0.4:
        {
            if (healthy)
            {
                ++beats;
                llOwnerSay("beat " + (string)beats);
                if (beats == 3)
                {
                    healthy = FALSE;                        // from now on the worker does nothing
                    llOwnerSay("worker hangs");
                }
            }
        }

        timer "watchdog" -> 1.0:
        {
            if (beats == lastSeen)
            {
                ++alarms;
                llOwnerSay("watchdog: no progress since the last check (alarm " + (string)alarms + ")");
            }
            else
                llOwnerSay("watchdog: " + (string)(beats - lastSeen) + " beats since the last check");
            lastSeen = beats;
        }
    }
}
```

**Output** (run 2026-10-17):

```text
start
beat 1
beat 2
watchdog: 2 beats since the last check
beat 3
worker hangs
watchdog: 1 beats since the last check
watchdog: no progress since the last check (alarm 1)
watchdog: no progress since the last check (alarm 2)
```

## 29. Inline timers `timer -> seconds: { }`

A countdown written at the place it starts. The body becomes the state's `timer()` handler (`zengine desugar` shows `llSetTimerEvent(0.5)` in `state_entry` and the body in `timer()`).

```lsl
// c29_inline_timer.lsl - timer -> seconds: { body } (unnamed): a countdown written where it is started.
// The body becomes the state's timer() handler. An unnamed inline timer cannot be mixed with an explicit
// timer() event in the same state (compile error); use named timers for that (see c28).
// Run with a wait of about 3500 ms.
integer remaining = 5;

default
{
    state_entry()
    {
        llOwnerSay("countdown from " + (string)remaining);
        timer -> 0.5:
        {
            --remaining;
            if (remaining > 0)
                llOwnerSay(f"{remaining}...");
            else
            {
                llSetTimerEvent(0.0);
                llOwnerSay("liftoff");
            }
        }
    }
}
```

**Output** (run 2026-10-17):

```text
countdown from 5
4...
3...
2...
1...
liftoff
```

An unnamed inline timer cannot be combined with an explicit `timer()` in the same state; named timers (28) can.

---
Back to [[ZEXT Constructs - One Example Each]].
