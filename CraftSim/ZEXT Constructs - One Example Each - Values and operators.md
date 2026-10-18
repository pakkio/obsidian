---
tags:
  - craftsim
  - zext
  - zengine
  - examples
  - constructs
---

# ZEXT Constructs - One Example Each - Values and operators

> Part of [[ZEXT Constructs - One Example Each]]. Sections: booleans, decimals, ternary, `??`, pipeline, bitwise compound assignment.

## 1. `boolean` and lowercase `true` / `false`

A door-access rule set. `boolean` is an integer underneath, so it can sit in lists and be passed around; the lowercase literals are accepted only with ZEXT on. An f-string prints a boolean as `TRUE` / `FALSE`.

```lsl
// c01_boolean.lsl - the boolean type and lowercase true / false: door access rules.
// A boolean is an integer underneath; a ZEXT f-string prints it as TRUE / FALSE.
boolean isStaff(string role)
{
    return role == "staff" || role == "admin";
}

boolean isOpenHour(integer hour)
{
    return hour >= 8 && hour < 20;
}

// During a lockdown only admins get in; staff always do; guests need a key and opening hours.
boolean canEnter(string role, integer hour, boolean hasKey, boolean lockdown)
{
    if (lockdown) return role == "admin";
    if (isStaff(role)) return true;
    return hasKey && isOpenHour(hour);
}

string verdict(boolean ok)
{
    return ok ? "ENTER" : "denied";
}

default
{
    state_entry()
    {
        list roles = ["guest", "guest", "guest", "staff", "admin"];
        list hours = [9, 22, 9, 3, 3];
        list keys  = [true, true, false, false, false];

        integer allowed = 0;
        integer i;
        for (i = 0; i < llGetListLength(roles); ++i)
        {
            string role = llList2String(roles, i);
            boolean hasKey = llList2Integer(keys, i);
            boolean ok = canEnter(role, llList2Integer(hours, i), hasKey, false);
            if (ok) ++allowed;
            llOwnerSay(f"{role} at {llList2Integer(hours, i)}h, key {hasKey}: {verdict(ok)}");
        }
        llOwnerSay(f"{allowed} of {llGetListLength(roles)} may enter normally");

        boolean lockdown = true;
        llOwnerSay("lockdown: staff " + verdict(canEnter("staff", 12, false, lockdown))
                   + ", admin " + verdict(canEnter("admin", 12, false, lockdown)));
    }
}
```

**Output** (run 2026-10-17):

```text
guest at 9h, key TRUE: ENTER
guest at 22h, key TRUE: denied
guest at 9h, key FALSE: denied
staff at 3h, key FALSE: ENTER
admin at 3h, key FALSE: ENTER
3 of 5 may enter normally
lockdown: staff denied, admin ENTER
```

## 2. `decimal(precision, scale)`

An invoice with 22 % VAT, and ten additions of 0.10. `decimal(12,2)` keeps exactly two decimals, so sums and shares are exact and round to the scale.

```lsl
// c02_decimal.lsl - decimal(precision, scale): exact money arithmetic where float drifts.
decimal(12,2) lineTotal(decimal(12,2) price, integer qty)
{
    return price * qty;
}

default
{
    state_entry()
    {
        // an invoice: three lines, 22 % VAT
        decimal(12,2) subtotal = 0.00;
        subtotal += lineTotal(19.99, 3);
        subtotal += lineTotal(4.50, 2);
        subtotal += lineTotal(0.10, 7);
        decimal(4,2) rate = 0.22;                       // a float literal cannot be mixed into decimal arithmetic:
        decimal(12,2) vat = subtotal * rate;           // keep the rate in a decimal of its own
        decimal(12,2) total = subtotal + vat;
        llOwnerSay("subtotal " + (string)subtotal + "  vat " + (string)vat + "  total " + (string)total);

        // ten times 0.10: float is off by a hair, decimal(10,2) is exact
        float f = 0.0;
        decimal(10,2) d = 0.00;
        integer i;
        for (i = 0; i < 10; ++i)
        {
            f += 0.1;
            d += 0.10;
        }
        llOwnerSay("float   10 x 0.1  = " + (string)f + "  equals 1.0: " + (string)(f == 1.0));
        llOwnerSay("decimal 10 x 0.10 = " + (string)d + "  prints as 1.00: " + (string)((string)d == "1.00"));

        // division rounds to the scale
        decimal(12,2) share = total / 7;
        llOwnerSay("total split 7 ways: " + (string)share + " each");
    }
}
```

**Output** (run 2026-10-17):

```text
subtotal 69.67  vat 15.33  total 85.00
float   10 x 0.1  = 1.000000  equals 1.0: 0
decimal 10 x 0.10 = 1.00  prints as 1.00: 1
total split 7 ways: 12.14 each
```

Two limits found while writing it: a **float literal cannot be mixed into decimal arithmetic** (`subtotal * 0.22` is a compile error, "use an explicit (decimal(p,s)) cast"; keep the rate in a `decimal(4,2)` of its own), and comparing a decimal with a float literal (`c == 0.30`) was false even for `0.10 + 0.20`; compare the printed text or another decimal.

## 3. Ternary `c ? a : b`

Shipping cost by weight band written as one chained ternary that reads like an `if` / `else if` ladder, a second ternary for the express surcharge, and a `clamp` made of two nested ternaries.

```lsl
// c03_ternary.lsl - the ternary operator: shipping price by weight band, nested and chained.
float shipping(float kg, boolean express)
{
    // chained ternaries read top to bottom like an if / else if ladder
    float base = kg <= 1.0  ? 4.5
               : kg <= 5.0  ? 7.9
               : kg <= 20.0 ? 14.0
               :              14.0 + (kg - 20.0) * 0.8;
    return express ? base * 1.6 : base;
}

integer clamp(integer x, integer lo, integer hi)
{
    return x < lo ? lo : x > hi ? hi : x;
}

default
{
    state_entry()
    {
        list weights = [0.4, 3.0, 12.0, 35.0];
        integer i;
        for (i = 0; i < llGetListLength(weights); ++i)
        {
            float kg = llList2Float(weights, i);
            llOwnerSay(f"{kg} kg: standard {llGetSubString((string)shipping(kg, false), 0, 4)}"
                       f"  express {llGetSubString((string)shipping(kg, true), 0, 4)}");
        }
        llOwnerSay("clamp: " + (string)clamp(-5, 0, 10) + " " + (string)clamp(7, 0, 10) + " " + (string)clamp(99, 0, 10));
    }
}
```

**Output** (run 2026-10-17):

```text
0.400000 kg: standard 4.500  express 7.200
3.000000 kg: standard 7.900  express 12.64
12.000000 kg: standard 14.00  express 22.40
35.000000 kg: standard 26.00  express 41.60
clamp: 0 7 10
```

## 4. Null-coalescing `a ?? b`

Layered configuration: a user setting, else the group setting, else the built-in default, chained with `??`. "Missing" means a blank string, `NULL_KEY` or the value `llJsonGetValue` returns for an absent key, and the test is made **before** the cast, so `(integer)llJsonGetValue(...) ?? 10` works.

```lsl
// c04_coalesce.lsl - the ?? operator: layered configuration (user > group > built-in defaults).
// "Missing" means a blank string, NULL_KEY or llJsonGetValue's JSON_INVALID. The test runs on the value
// BEFORE a cast, so  (integer)llJsonGetValue(...) ?? fallback  works even though the cast would give 0.
string DEFAULTS = """{"speed": 5, "radius": 20, "color": "white", "title": "Lamp"}""";
string GROUP    = """{"speed": 8, "color": ""}""";
string USER     = """{"radius": 50}""";

default
{
    state_entry()
    {
        integer speed  = (integer)llJsonGetValue(USER, ["speed"])
                      ?? (integer)llJsonGetValue(GROUP, ["speed"])
                      ?? (integer)llJsonGetValue(DEFAULTS, ["speed"]);
        integer radius = (integer)llJsonGetValue(USER, ["radius"])
                      ?? (integer)llJsonGetValue(GROUP, ["radius"])
                      ?? (integer)llJsonGetValue(DEFAULTS, ["radius"]);
        // the group sets color to "" (blank = missing), so the default applies
        string color   = llJsonGetValue(USER, ["color"])
                      ?? llJsonGetValue(GROUP, ["color"])
                      ?? llJsonGetValue(DEFAULTS, ["color"]);
        string title   = llJsonGetValue(USER, ["title"]) ?? llJsonGetValue(GROUP, ["title"]) ?? "untitled";

        llOwnerSay(f"speed {speed} (from group), radius {radius} (from user), color {color} (default), title {title}");

        string nobody = "";
        llOwnerSay("blank string: " + (nobody ?? "guest") + " | NULL_KEY: " + ((string)(key)NULL_KEY ?? "no key"));
    }
}
```

**Output** (run 2026-10-17):

```text
speed 8 (from group), radius 50 (from user), color white (default), title untitled
blank string: guest | NULL_KEY: no key
```

The group sets `color` to `""`; a blank counts as missing, so the default `white` wins.

## 5. Pipeline `x |> f(args)`

A slug generator and a word counter. The left side becomes the **first** argument of the call, so the pipeline reads in the order things happen; the last line shows the same chain written inside out.

```lsl
// c05_pipeline.lsl - the |> operator: the left side becomes the FIRST argument of the call on the right.
// Reads left to right instead of inside out.
string slug(string title)
{
    return title
        |> llStringTrim(STRING_TRIM)
        |> llToLower()
        |> llParseString2List([" ", "_", "-", ","], [])
        |> llDumpList2String("-");
}

integer wordCount(string text)
{
    return text |> llParseString2List([" ", "\n"], []) |> llGetListLength();
}

default
{
    state_entry()
    {
        list titles = ["  Hello, Brave New World  ", "snake_case_to-slug", "One"];
        integer i;
        for (i = 0; i < llGetListLength(titles); ++i)
        {
            string t = llList2String(titles, i);
            llOwnerSay("[" + t + "] -> " + slug(t));
        }
        llOwnerSay("words: " + (string)wordCount("the quick brown fox jumps over the lazy dog"));
        // the same pipeline written inside out, for comparison
        llOwnerSay("inside out: " + llDumpList2String(llParseString2List(llToLower(llStringTrim("  A B  ", STRING_TRIM)), [" "], []), "-"));
    }
}
```

**Output** (run 2026-10-17):

```text
[  Hello, Brave New World  ] -> hello-brave-new-world
[snake_case_to-slug] -> snake-case-to-slug
[One] -> one
words: 9
inside out: a-b
```

## 6. Compound bitwise and shift assignment `|= &= ^= <<= >>=`

A permission mask (grant, toggle, revoke) and an RGB colour packed into one integer and unpacked again.

```lsl
// c06_compound_bits.lsl - compound bitwise and shift assignment: |= &= ^= <<= >>=
// A permission mask and an RGB colour packed into one integer.
integer READ  = 1;
integer WRITE = 2;
integer EXEC  = 4;
integer ADMIN = 8;

string show(integer p)
{
    string s = "";
    s += (p & READ)  ? "r" : "-";
    s += (p & WRITE) ? "w" : "-";
    s += (p & EXEC)  ? "x" : "-";
    s += (p & ADMIN) ? "A" : "-";
    return s + " (" + (string)p + ")";
}

integer pack(integer r, integer g, integer b)
{
    integer c = 0;
    c |= r << 16;
    c |= g << 8;
    c |= b;
    return c;
}

default
{
    state_entry()
    {
        integer p = 0;
        p |= READ | WRITE;      llOwnerSay("grant read+write    " + show(p));
        p ^= EXEC;              llOwnerSay("toggle exec on      " + show(p));
        p ^= EXEC;              llOwnerSay("toggle exec off     " + show(p));
        p &= ~WRITE;            llOwnerSay("revoke write        " + show(p));
        p |= ADMIN;             llOwnerSay("grant admin         " + show(p));

        integer c = pack(255, 128, 7);
        integer r = (c >> 16) & 255;
        integer g = (c >> 8) & 255;
        integer b = c & 255;
        llOwnerSay("packed " + (string)c + " -> r " + (string)r + " g " + (string)g + " b " + (string)b);

        integer x = 3;
        x <<= 4;                llOwnerSay("3 <<= 4 -> " + (string)x);
        x >>= 2;                llOwnerSay("then >>= 2 -> " + (string)x);
    }
}
```

**Output** (run 2026-10-17):

```text
grant read+write    rw-- (3)
toggle exec on      rwx- (7)
toggle exec off     rw-- (3)
revoke write        r--- (1)
grant admin         r--A (9)
packed 16744455 -> r 255 g 128 b 7
3 <<= 4 -> 48
then >>= 2 -> 12
```

---
Back to [[ZEXT Constructs - One Example Each]].
