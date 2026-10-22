---
tags:
  - craftsim
  - zext
  - zengine
  - examples
  - constructs
---

# ZEXT Constructs - One Example Each - Control flow and destructuring

> Part of [[ZEXT Constructs - One Example Each]]. Sections: `switch`, `break`/`continue`, `foreach`, destructuring, closures.

## 19. `switch` / `case` / `default`

A command interpreter that switches on a string (two names for one case) and a loyalty-points function that falls through on purpose: a gold member also gets the silver and bronze bonuses.

```lsl
// c19_switch.lsl - switch / case / default on strings and integers, with grouped cases and deliberate fall-through.
string run(string cmd, list args)
{
    switch (cmd)
    {
        case "add":
        case "plus":                                   // two names, one meaning
            return (string)(llList2Integer(args, 0) + llList2Integer(args, 1));
        case "neg":
            return (string)(-llList2Integer(args, 0));
        case "echo":
            return llDumpList2String(args, " ");
        default:
            return "unknown command '" + cmd + "'";
    }
    return "";
}

// fall-through on purpose: a higher tier includes the benefits of the lower ones
integer loyaltyPoints(integer tier)
{
    integer pts = 0;
    switch (tier)
    {
        case 3: pts += 30;                              // gold: also gets the silver and bronze bonuses
        case 2: pts += 20;
        case 1: pts += 10;
            break;
        default:
            pts = -1;
    }
    return pts;
}

default
{
    state_entry()
    {
        llOwnerSay("add 2 3   -> " + run("add", [2, 3]));
        llOwnerSay("plus 40 2 -> " + run("plus", [40, 2]));
        llOwnerSay("neg 7     -> " + run("neg", [7]));
        llOwnerSay("echo      -> " + run("echo", ["hello", "switch"]));
        llOwnerSay("jump      -> " + run("jump", []));

        integer tier;
        for (tier = 0; tier <= 3; ++tier)
            llOwnerSay("tier " + (string)tier + " -> " + (string)loyaltyPoints(tier) + " points");
    }
}
```

**Output** (run 2026-10-17):

```text
add 2 3   -> 5
plus 40 2 -> 42
neg 7     -> -7
echo      -> hello switch
jump      -> unknown command 'jump'
tier 0 -> -1 points
tier 1 -> 10 points
tier 2 -> 30 points
tier 3 -> 60 points
```

## 20. `break` and `continue`

The first twelve primes (`continue` skips even numbers, `break` leaves the divisor loop at the first hit) and the longest Collatz chain below 30 (an endless `while (TRUE)` that ends itself with `break`).

```lsl
// c20_break_continue.lsl - break and continue: the first 12 primes, and the longest Collatz chain below 30.
default
{
    state_entry()
    {
        list primes = [];
        integer n = 1;
        while (llGetListLength(primes) < 12)
        {
            ++n;
            if (n > 2 && n % 2 == 0) continue;                 // skip even numbers after 2
            boolean isPrime = true;
            integer d;
            for (d = 3; d * d <= n; d += 2)
            {
                if (n % d == 0)
                {
                    isPrime = false;
                    break;                                     // one divisor is enough
                }
            }
            if (isPrime) primes += n;
        }
        llOwnerSay("primes: " + llList2CSV(primes));

        // Collatz: the number below 30 whose chain is longest; the loop stops itself with break
        integer bestStart = 1;
        integer bestLength = 1;
        integer start;
        for (start = 2; start < 30; ++start)
        {
            integer x = start;
            integer steps = 1;
            while (TRUE)
            {
                if (x == 1) break;
                x = x % 2 == 0 ? x / 2 : 3 * x + 1;
                ++steps;
            }
            if (steps > bestLength)
            {
                bestLength = steps;
                bestStart = start;
            }
        }
        llOwnerSay("longest Collatz chain below 30: start " + (string)bestStart + ", " + (string)bestLength + " numbers");
    }
}
```

**Output** (run 2026-10-17):

```text
primes: 2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37
longest Collatz chain below 30: start 27, 112 numbers
```

## 21. `foreach (type x in list)`

Totals per category from CSV rows: the loop variable is a fresh string each turn, and the totals are kept in a strided list.

```lsl
// c21_foreach.lsl - foreach (type x in list): totals per category from a list of CSV rows.
default
{
    state_entry()
    {
        list rows = [
            "apple,fruit,3,1.20",
            "pear,fruit,2,1.50",
            "carrot,veg,5,0.40",
            "leek,veg,1,0.90",
            "fig,fruit,4,2.00",
            "oats,grain,2,3.10"
        ];

        list totals = [];                                      // category, amount, category, amount, ...
        float grand = 0.0;
        integer lines = 0;

        foreach (string row in rows)
        {
            list c = llCSV2List(row);
            string category = llList2String(c, 1);
            float amount = llList2Integer(c, 2) * llList2Float(c, 3);

            integer at = llListFindList(totals, [category]);
            if (at < 0)
                totals += [category, amount];
            else
                totals = llListReplaceList(totals, [llList2Float(totals, at + 1) + amount], at + 1, at + 1);

            grand += amount;
            ++lines;
        }

        integer i;
        for (i = 0; i < llGetListLength(totals); i += 2)
            llOwnerSay(llList2String(totals, i) + ": " + llGetSubString((string)llList2Float(totals, i + 1), 0, 4));
        llOwnerSay((string)lines + " lines, grand total " + llGetSubString((string)grand, 0, 4));
    }
}
```

**Output** (run 2026-10-17):

```text
fruit: 14.60
veg: 2.900
grain: 6.200
6 lines, grand total 23.70
```

## 22. List destructuring `[type a, type b] = list;`

Typed rows unpacked in one statement (text converted to `string`, `integer`, `float`), and a function that returns two numbers as a list, unpacked by the caller.

```lsl
// c22_destructuring.lsl - list destructuring: [type a, type b] = list;  Every variable needs its type, and values
// are converted to it. Parse typed rows, and return several results from one function.
list minMax(list values)
{
    float lo = llList2Float(values, 0);
    float hi = lo;
    integer i;
    for (i = 1; i < llGetListLength(values); ++i)
    {
        float v = llList2Float(values, i);
        if (v < lo) lo = v;
        if (v > hi) hi = v;
    }
    return [lo, hi];
}

default
{
    state_entry()
    {
        // the last column is 1 / 0: a text "TRUE" would convert to 0, as (integer)"TRUE" does in stock LSL
        list rows = ["apple,3,1.25,1", "pear,0,1.50,0", "fig,12,3.40,1"];
        list prices = [];
        integer inStock = 0;

        foreach (string row in rows)
        {
            [string name, integer qty, float price, integer fresh] = llCSV2List(row);   // text -> typed variables
            prices += price;
            if (qty > 0) ++inStock;
            llOwnerSay(f"{name}: {qty} pcs at {price}{fresh ? " (fresh)" : ""}{qty == 0 ? "  OUT OF STOCK" : ""}");
        }

        [float cheapest, float dearest] = minMax(prices);                              // two results from one call
        llOwnerSay(f"{inStock} of {llGetListLength(rows)} in stock, price range {cheapest} .. {dearest}");
    }
}
```

**Output** (run 2026-10-17):

```text
apple: 3 pcs at 1.250000 (fresh)
pear: 0 pcs at 1.500000  OUT OF STOCK
fig: 12 pcs at 3.400000 (fresh)
2 of 3 in stock, price range 1.250000 .. 3.400000
```

Every variable needs its type. Conversion follows stock LSL: the text `TRUE` becomes 0 as an integer, so the example stores 1 / 0.

## 23. Closures `(params) => { body }`

Callbacks written where they are started: a timer that ticks three times and stops itself, and a chat listener that receives the event parameters. Run with the message `hello there` on channel 0.

```lsl
// c23_closures.lsl - callback closures:  (params) => { body }  as the last argument of llSetTimerEvent, llSensor
// and llListen. The compiler turns the body into the timer() / sensor() / listen() handler for you.
// Only these three calls accept a closure. Run with a wait of about 2500 ms and the chat message  hello there.
integer ticks;

default
{
    state_entry()
    {
        // 1) a timer callback with a counter: three ticks, then stop
        llSetTimerEvent(0.4, () =>
        {
            ++ticks;
            llOwnerSay("tick " + (string)ticks);
            if (ticks == 3)
            {
                llSetTimerEvent(0.0);
                llOwnerSay("timer stopped");
            }
        });

        // 2) a listen callback: it receives the event's parameters
        llListen(0, "", NULL_KEY, "", (integer channel, string who, key id, string msg) =>
        {
            llOwnerSay("heard on channel " + (string)channel + ": " + msg + " (" + (string)llStringLength(msg) + " characters)");
        });
    }
}
```

**Output** (run 2026-10-17):

```text
tick 1
tick 2
tick 3
timer stopped
heard on channel 0: hello there (11 characters)
```

Closures are **not general lambdas**: only `llSetTimerEvent(seconds, closure)`, `llSensor(..., closure)` and `llListen(..., closure)` accept one, and the compiler turns the body into the `timer()`, `sensor()` or `listen()` handler. The body works on globals.

---
Back to [[ZEXT Constructs - One Example Each]].
