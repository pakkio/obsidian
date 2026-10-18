---
tags:
  - craftsim
  - zext
  - zengine
  - examples
  - constructs
---

# ZEXT Constructs - One Example Each - State arguments and preprocessor

> Part of [[ZEXT Constructs - One Example Each]]. Sections: state parameters, `#define` family, `#TRON`/`#TROFF`.

## 16. State parameters and state arguments

A retry machine with exponential back-off: each state carries its own attempt number and delay, handed over with `state waiting(n + 1, delay * 2.0)`.

```lsl
// c16_state_params.lsl - state parameters and state arguments: a retry machine with exponential back-off.
// Parameters are in scope in state_entry ONLY, so each state does its work there; other events see globals.
// Going from a state to the SAME state does not re-enter it (as in stock LSL), hence the two alternating states.
// Run it with a wait of about 3500 ms.
integer gTries;
float gDelay;

default
{
    state_entry()
    {
        llOwnerSay("start");
        state waiting(1, 0.2);                                  // state arguments
    }
}

state waiting(integer n, float delay)                          // state parameters
{
    state_entry()
    {
        gTries = n;
        gDelay = delay;
        llOwnerSay(f"attempt {n}: waiting {delay} s");
        llSetTimerEvent(delay);
    }

    timer()
    {
        llSetTimerEvent(0.0);
        state checking(gTries, gDelay);
    }
}

state checking(integer n, float delay)
{
    state_entry()
    {
        if (n >= 3) state giveup(n, delay);
        llOwnerSay(f"attempt {n} failed, trying again with {delay * 2.0} s");
        state waiting(n + 1, delay * 2.0);
    }
}

state giveup(integer total, float lastDelay)
{
    state_entry()
    {
        llOwnerSay(f"giving up after {total} attempts, last delay {lastDelay} s");
    }
}
```

**Output** (run 2026-10-17):

```text
start
attempt 1: waiting 0.200000 s
attempt 1 failed, trying again with 0.400000 s
attempt 2: waiting 0.400000 s
attempt 2 failed, trying again with 0.800000 s
attempt 3: waiting 0.800000 s
giving up after 3 attempts, last delay 0.800000 s
```

Two behaviours found the hard way: the parameters are in scope in **`state_entry` only** (using `n` in `timer()` is "undefined variable"), so other events read globals; and a transition from a state **to itself** with new arguments did not re-enter it, which is why the machine alternates between `waiting` and `checking`.

## 17. Preprocessor: `#define`, `#ifdef`, `#ifndef`, `#if`, `#else`, `#undef`, `#x`, `a##b`

One source, a debug and a release flavour: object-like and function-like macros, a `LOG` macro that exists only while `DEBUG` is defined, a version switch with `#if`, stringizing (`STR(MAX_TRIES)` gives `MAX_TRIES`) and token pasting (`JOIN(speed, Limit)` declares `speedLimit`).

```lsl
// c17_preprocessor.lsl - #define (object-like and function-like), #ifdef / #ifndef / #if / #else / #undef,
// stringizing (#x) and token pasting (a##b). A debug build and a release build from one source.
#define MAX_TRIES 3
#define CLAMP(x, lo, hi) ((x) < (lo) ? (lo) : ((x) > (hi) ? (hi) : (x)))
#define SQUARE(x) ((x) * (x))
#define STR(x) #x
#define JOIN(a, b) a##b

#define DEBUG 1
#ifdef DEBUG
  #define LOG(msg) llOwnerSay("[debug] " + (msg))
#else
  #define LOG(msg)
#endif

#ifndef RELEASE
string BUILD = "development build";
#else
string BUILD = "release build";
#endif

#if MAX_TRIES >= 3
integer GENEROUS = TRUE;
#else
integer GENEROUS = FALSE;
#endif

integer JOIN(speed, Limit) = 40;                 // declares  speedLimit

default
{
    state_entry()
    {
        llOwnerSay(BUILD + ", generous retries: " + (string)GENEROUS);
        LOG("this line exists only while DEBUG is defined");

        integer reading = 57;
        llOwnerSay("clamped " + (string)reading + " to " + (string)CLAMP(reading, 0, speedLimit)
                   + ", square of 12 is " + (string)SQUARE(12) + ", square of 3+1 is " + (string)SQUARE(3 + 1));
        llOwnerSay("the name of the macro argument is " + STR(MAX_TRIES) + " and its value is " + (string)MAX_TRIES);

        #undef DEBUG
        #undef LOG
        #define LOG(msg)
        LOG("this one is gone: DEBUG was undefined");
        llOwnerSay("done");
    }
}
```

**Output** (run 2026-10-17):

```text
development build, generous retries: 1
[debug] this line exists only while DEBUG is defined
clamped 57 to 40, square of 12 is 144, square of 3+1 is 16
the name of the macro argument is MAX_TRIES and its value is 3
done
```

The `SQUARE(3 + 1)` result is 16, not 10, because the macro parenthesises its argument. After `#undef DEBUG` / `#define LOG(msg)` the second `LOG` line produces nothing.

## 18. `#TRON` / `#TROFF`

Tracing a region of the code without editing it: every statement between the markers prints its line, its text and the variables, and each handler prints its entry.

```lsl
// c18_tron.lsl - #TRON / #TROFF: trace everything between the two lines without touching the code.
// Each statement prints "script:line  <source text> [variables]" and each handler prints its entry.
integer gcd(integer a, integer b)
{
    while (b != 0)
    {
        integer t = b;
        b = a % b;
        a = t;
    }
    return a;
}

default
{
    state_entry()
    {
#TRON
        integer g = gcd(48, 18);
        llOwnerSay("gcd = " + (string)g);
#TROFF
        llOwnerSay("this line is not traced");
    }
}
```

**Output** (run 2026-10-17):

```text
c18_tron.lsl:16  EVENT state_entry()
c18_tron.lsl:19  integer g = gcd(48, 18);  [g=6]
gcd = 6
c18_tron.lsl:20  llOwnerSay("gcd = " + (string)g);
this line is not traced
```

---
Back to [[ZEXT Constructs - One Example Each]].
