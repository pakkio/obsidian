---
tags:
  - craftsim
  - zext
  - zengine
  - examples
  - constructs
---

# ZEXT Constructs - One Example Each - Strings, indexing and slices

> Part of [[ZEXT Constructs - One Example Each]]. Sections: indexing, slices, map literals, string literals, f-strings, `%` and `*`.

## 7. Indexing `s[i]`, `s[-1]`, `list[i]`

A Caesar cipher, a palindrome test (`s[i]` against `s[-1 - i]`) and run-length encoding.

```lsl
// c07_index.lsl - s[i], s[-1], list[i]: a Caesar cipher, a palindrome test and run-length encoding.
// Negative indexes count from the end.
string ABC = "abcdefghijklmnopqrstuvwxyz";

string caesar(string text, integer shift)
{
    string out = "";
    integer i;
    for (i = 0; i < llStringLength(text); ++i)
    {
        string ch = text[i];
        integer p = llSubStringIndex(ABC, ch);
        out += p < 0 ? ch : ABC[(p + shift + 26) % 26];     // leave spaces and punctuation alone
    }
    return out;
}

boolean isPalindrome(string s)
{
    integer i;
    for (i = 0; i < llStringLength(s) / 2; ++i)
        if (s[i] != s[-1 - i]) return false;               // first against last, second against second-last...
    return true;
}

string rle(string s)
{
    string out = "";
    integer run = 1;
    integer i;
    for (i = 1; i <= llStringLength(s); ++i)
    {
        if (i < llStringLength(s) && s[i] == s[i - 1])
            ++run;
        else
        {
            out += (string)run + s[i - 1];
            run = 1;
        }
    }
    return out;
}

default
{
    state_entry()
    {
        string secret = caesar("meet me at noon", 3);
        llOwnerSay(secret + "  ->  " + caesar(secret, -3));
        llOwnerSay("level: " + (string)isPalindrome("level") + "  radar: " + (string)isPalindrome("radar")
                   + "  hello: " + (string)isPalindrome("hello"));
        llOwnerSay("rle: " + rle("aaabccddddd"));

        // list[i] works for a normal index. A NEGATIVE list index (scores[-1]) returns an empty value in
        // ZEngine 1.1.0 even though  zengine desugar  turns it into llList2Integer(scores, -1), which works;
        // use the length until that is fixed. (Negative STRING indexes, s[-1], are fine.)
        list scores = [40, 75, 91, 63];
        llOwnerSay("first " + (string)scores[0] + ", last " + (string)scores[llGetListLength(scores) - 1] + ", third " + (string)scores[2]);
    }
}
```

**Output** (run 2026-10-17):

```text
phhw ph dw qrrq  ->  meet me at noon
level: TRUE  radar: TRUE  hello: FALSE
rle: 3a1b2c5d
first 40, last 63, third 91
```

**Bug found:** a *negative list index* (`scores[-1]`) returns an empty value in ZEngine 1.1.0, while `zengine desugar` turns the same expression into `llList2Integer(scores, -1)`, which returns 63. Negative **string** indexes (`s[-1]`) work. The script uses `scores[llGetListLength(scores) - 1]` instead. Listed in [[Known and Unresolved Problems]].

## 8. Range slice `s[a:b]`

A timestamp parser (`"20261017T143005Z"` into a date and seconds into the day), a split at the last dot of a file name, and a word wrap. The stop index is **inclusive**: `ts[0:3]` is four characters.

```lsl
// c08_slice.lsl - s[a:b] range slices (the stop index is INCLUSIVE): a timestamp parser, file name parts, word wrap.
string two(integer n)
{
    return n < 10 ? "0" + (string)n : (string)n;
}

// "20261017T143005Z" -> a readable date and a seconds-since-midnight value
string describe(string ts)
{
    integer year  = (integer)ts[0:3];
    integer month = (integer)ts[4:5];
    integer day   = (integer)ts[6:7];
    integer hh    = (integer)ts[9:10];
    integer mm    = (integer)ts[11:12];
    integer ss    = (integer)ts[13:14];
    return (string)year + "-" + two(month) + "-" + two(day) + " " + two(hh) + ":" + two(mm) + ":" + two(ss)
           + " (" + (string)(hh * 3600 + mm * 60 + ss) + " s into the day)";
}

// split "report-final.v2.md" at the LAST dot
list nameAndExtension(string file)
{
    integer dot = -1;
    integer i;
    for (i = 0; i < llStringLength(file); ++i)
        if (file[i] == ".") dot = i;
    if (dot < 0) return [file, ""];
    return [file[:dot - 1], file[dot + 1:]];
}

default
{
    state_entry()
    {
        llOwnerSay(describe("20261017T143005Z"));

        list parts = nameAndExtension("report-final.v2.md");
        llOwnerSay("name [" + llList2String(parts, 0) + "] extension [" + llList2String(parts, 1) + "]");
        llOwnerSay("last three characters: " + "documentation"[-3:]);

        // word wrap at 18 columns using slices
        string text = "Slices make it easy to cut a long line into pieces of a fixed width";
        integer width = 18;
        integer start = 0;
        while (start < llStringLength(text))
        {
            integer stop = start + width - 1;
            if (stop >= llStringLength(text)) stop = llStringLength(text) - 1;
            llOwnerSay("|" + text[start:stop] + "|");
            start += width;
        }
    }
}
```

**Output** (run 2026-10-17):

```text
2026-10-17 14:30:05 (52205 s into the day)
name [report-final.v2] extension [md]
last three characters: ion
|Slices make it eas|
|y to cut a long li|
|ne into pieces of |
|a fixed width|
```

## 9. Map literal `["key" => value]`

A price list, a shopping cart priced against it, a merge with another map, a sort by key and a JSON export. A map is a strided list, so `llListFindList`, `llListSort` and `llList2Json` all work on it.

```lsl
// c09_map.lsl - map literals ["key" => value]: a price list, a shopping cart and a JSON export.
// A map is just a strided list, so every list function still works on it.
list PRICES = ["apple" => 1.2, "pear" => 1.5, "plum" => 2.0, "fig" => 3.4];

float priceOf(string item)
{
    integer at = llListFindList(PRICES, [item]);
    if (at < 0 || at % 2 == 1) return -1.0;            // not a key (or matched a value)
    return llList2Float(PRICES, at + 1);
}

default
{
    state_entry()
    {
        list cart = ["apple" => 3, "plum" => 2, "kiwi" => 1, "fig" => 1];
        float total = 0.0;
        integer i;
        for (i = 0; i < llGetListLength(cart); i += 2)
        {
            string item = llList2String(cart, i);
            integer qty = llList2Integer(cart, i + 1);
            float price = priceOf(item);
            if (price < 0.0)
                llOwnerSay(f"{item}: not on the price list");
            else
            {
                total += price * qty;
                llOwnerSay(f"{item}: {qty} x {price}");
            }
        }
        llOwnerSay("total " + llGetSubString((string)total, 0, 4));

        // add a key by concatenating another map, then sort by key
        list more = PRICES + ["kiwi" => 2.5];
        more = llListSort(more, 2, TRUE);
        llOwnerSay("sorted keys: " + llList2CSV(llList2ListStrided(more, 0, -1, 2)));

        // export as JSON text
        llOwnerSay(llList2Json(JSON_OBJECT, more));
    }
}
```

**Output** (run 2026-10-17):

```text
apple: 3 x 1.200000
plum: 2 x 2.000000
kiwi: not on the price list
fig: 1 x 3.400000
total 11.00
sorted keys: apple, fig, kiwi, pear, plum
{"apple":1.2,"fig":3.4,"kiwi":2.5,"pear":1.5,"plum":2}
```

## 10. Adjacent string literals

A multi-line help text written as five adjacent literals with no `+`; the script then parses its own text to list the options.

```lsl
// c10_adjacent.lsl - adjacent string literals are joined at compile time: no + and no \n gymnastics
// in a long help text. The script then reads its own text back to list the options.
string HELP = "usage: tool [options] file\n"
              "  -h, --help      show this text\n"
              "  -v, --verbose   print every step\n"
              "  -n, --dry-run   change nothing\n"
              "  -o, --output F  write the result to F\n";

default
{
    state_entry()
    {
        list lines = llParseString2List(HELP, ["\n"], []);
        llOwnerSay("lines: " + (string)llGetListLength(lines) + ", characters: " + (string)llStringLength(HELP));

        integer options = 0;
        string shorts = "";
        foreach (string line in lines)
        {
            if (line[:2] == "  -")                       // characters 0..2 (the stop index is inclusive): two spaces and a dash
            {
                ++options;
                shorts += line[3] + " ";
            }
        }
        llOwnerSay(f"{options} options, short forms: {shorts}");
        llOwnerSay(lines[0]);
    }
}
```

**Output** (run 2026-10-17):

```text
lines: 5, characters: 168
4 options, short forms: h v n o
usage: tool [options] file
```

The first version of this script checked `line[:3] == "  -"` and found 0 options: the stop is inclusive, so `[:3]` is **four** characters. `[:2]` is the three-character prefix.

## 11. Raw strings `r"..."`

A Windows path taken apart, and a regular-expression pattern kept exactly as typed. Backslashes are literal, so the raw string equals its doubly-escaped form.

```lsl
// c11_raw.lsl - raw strings r"...": backslashes are literal, so Windows paths and regular expressions read as written.
default
{
    state_entry()
    {
        string path = r"C:\Users\claud\notes\plan.v2.md";
        string escaped = "C:\\Users\\claud\\notes\\plan.v2.md";
        llOwnerSay("raw equals the escaped form: " + (string)(path == escaped) + " (" + (string)llStringLength(path) + " characters)");

        // take the path apart
        string backslash = llChar(92);                      // the separator, written without any escape
        list parts = llParseString2List(path, [backslash], []);
        string file = llList2String(parts, -1);
        llOwnerSay("drive " + llList2String(parts, 0) + ", " + (string)(llGetListLength(parts) - 2) + " folders, file " + file);
        llOwnerSay("rebuilt with slashes: " + llDumpList2String(parts, "/"));

        // a pattern a script would hand to a regular-expression engine, kept exactly as typed
        string pattern = r"^\d{3}-\d{4}\s*$";
        llOwnerSay("pattern " + pattern + " has " + (string)llStringLength(pattern) + " characters");
        llOwnerSay("a normal string would turn \\d into: " + "\d");
    }
}
```

**Output** (run 2026-10-17):

```text
raw equals the escaped form: 1 (31 characters)
drive C:, 3 folders, file plan.v2.md
rebuilt with slashes: C:/Users/claud/notes/plan.v2.md
pattern ^\d{3}-\d{4}\s*$ has 16 characters
a normal string would turn \d into: d
```

## 12. Triple-quoted strings `"""..."""`

A settings file embedded in the script: quotes inside need no escape and the text spans lines. The script skips comments and blank lines, splits each line at the first `=` and strips surrounding quotes.

```lsl
// c12_triple.lsl - triple-quoted strings: embedded quotes, several lines, no escapes.
// A small settings file is parsed: comments and blank lines are skipped, key = value is split at the first "=".
string SETTINGS = """
# region settings
name = "Pakkio Plaza"
max_agents = 40

; a different comment style
greeting = He said "welcome" and left
path = C:\temp\x
""";

string trim(string s)
{
    return llStringTrim(s, STRING_TRIM);
}

default
{
    state_entry()
    {
        integer seen = 0;
        foreach (string raw in llParseStringKeepNulls(SETTINGS, ["\n"], []))
        {
            string line = trim(raw);
            if (line == "" || line[0] == "#" || line[0] == ";") jump skip;

            integer eq = llSubStringIndex(line, "=");
            string name = trim(line[:eq - 1]);
            string value = trim(line[eq + 1:]);
            string dq = llChar(34);                         // the double-quote character
            if (value[0] == dq && value[-1] == dq)          // strip one pair of surrounding quotes
                value = value[1:llStringLength(value) - 2];
            ++seen;
            llOwnerSay(f"{name} = [{value}]");
            @skip;
        }
        llOwnerSay((string)seen + " settings read");
    }
}
```

**Output** (run 2026-10-17):

```text
name = [Pakkio Plaza]
max_agents = [40]
greeting = [He said "welcome" and left]
path = [C:<TAB>empx]
4 settings read
```

**Escapes still apply inside a triple-quoted string.** The line `path = C:\temp\x` came out as `C:` + a real tab + `empx`: `\t` became a tab and `\x` became `x`. For text that must stay literal use a raw string (11).

## 13. f-strings `f"... {expr} ..."`

An order table. Any expression goes inside the braces, including calls, arithmetic and even a call whose arguments contain quoted strings; `{{` and `}}` give literal braces.

```lsl
// c13_fstring.lsl - f"..." strings: any expression inside {}, including calls and arithmetic. {{ and }} are literal braces.
string money(float x)
{
    string s = (string)((float)((integer)(x * 100.0 + 0.5)) / 100.0);
    integer dot = llSubStringIndex(s, ".");
    return llGetSubString(s, 0, dot + 2);
}

default
{
    state_entry()
    {
        list orders = ["lamp", 2, 14.9, "desk", 1, 120.0, "cable", 5, 2.35];
        float grand = 0.0;
        integer i;
        llOwnerSay("ITEM      QTY   PRICE    LINE");
        for (i = 0; i < llGetListLength(orders); i += 3)
        {
            string name = llList2String(orders, i);
            integer qty = llList2Integer(orders, i + 1);
            float price = llList2Float(orders, i + 2);
            grand += qty * price;
            llOwnerSay(f"{name}{llGetSubString("          ", 0, 9 - llStringLength(name))}{qty}   {money(price)}   {money(qty * price)}");
        }
        llOwnerSay(f"total {money(grand)}, {llGetListLength(orders) / 3} lines, average {money(grand / (llGetListLength(orders) / 3))}");
        llOwnerSay(f"literal braces: {{qty}} and {{{llGetListLength(orders)}}}");
    }
}
```

**Output** (run 2026-10-17):

```text
ITEM      QTY   PRICE    LINE
lamp      2   14.90   29.80
desk      1   120.00   120.00
cable     5   2.35   11.75
total 161.55, 3 lines, average 53.85
literal braces: {qty} and {9}
```

## 14. Raw f-strings `rf"..."`

Output paths for a batch of exports: backslashes stay literal and the `{...}` parts are still filled in.

```lsl
// c14_rawf.lsl - rf"..." (raw f-string): backslashes stay literal AND {expressions} are filled in.
// Building Windows output paths for a batch of exports.
default
{
    state_entry()
    {
        string root = r"D:\exports";
        list regions = ["Nexus", "pakkio2", "PKC"];
        integer year = 2026;
        integer i;
        for (i = 0; i < llGetListLength(regions); ++i)
        {
            string region = llList2String(regions, i);
            string file = rf"{root}\{region}\{year}\objects-{i + 1}.csv";
            llOwnerSay(file);
        }
        // a plain f-string would read \n as a newline; rf keeps the two characters
        integer n = 3;
        llOwnerSay(rf"line break written as \n, count {n}");
    }
}
```

**Output** (run 2026-10-17):

```text
D:\exports\Nexus\2026\objects-1.csv
D:\exports\pakkio2\2026\objects-2.csv
D:\exports\PKC\2026\objects-3.csv
line break written as \n, count 3
```

## 15. String `%` formatting and `*` repetition

A text bar chart: `"#" * n` draws each bar, `"%s %s %d (%d%%)" % [...]` formats the line, and `" " * k` pads.

```lsl
// c15_percent_repeat.lsl - "fmt" % [values] and "text" * n: a text bar chart.
// % understands %d %i %s %f and %%; width and precision flags (%5d, %.2f, %x) are NOT interpreted and stay as typed.
string pad(string s, integer width)
{
    return s + (" " * (width - llStringLength(s)));       // a negative count gives an empty string
}

default
{
    state_entry()
    {
        list days   = ["mon", "tue", "wed", "thu", "fri"];
        list visits = [12, 31, 18, 45, 27];

        integer top = 0;
        integer total = 0;
        integer i;
        for (i = 0; i < llGetListLength(visits); ++i)
        {
            total += llList2Integer(visits, i);
            if (llList2Integer(visits, i) > top) top = llList2Integer(visits, i);
        }

        for (i = 0; i < llGetListLength(days); ++i)
        {
            integer v = llList2Integer(visits, i);
            integer bar = v * 20 / top;                          // longest bar is 20 characters
            integer pct = v * 100 / total;
            llOwnerSay("%s %s %d (%d%%)" % [pad(llList2String(days, i), 4), pad("#" * bar, 21), v, pct]);
        }
        llOwnerSay("%s: %d visits, busiest day %s with %f" % ["week", total, "thu", 45.0 / total * 100.0]);
        llOwnerSay("rule: " + ("-=" * 12));
    }
}
```

**Output** (run 2026-10-17):

```text
mon  #####                 12 (9%)
tue  #############         31 (23%)
wed  ########              18 (13%)
thu  ####################  45 (33%)
fri  ############          27 (20%)
week: 133 visits, busiest day thu with 33.834586
rule: -=-=-=-=-=-=-=-=-=-=-=-=
```

`%` understands `%d`, `%i`, `%s`, `%f` and `%%` **only**. Width, precision and other flags (`%5d`, `%.2f`, `%x`) are not interpreted: a format containing them comes back unchanged. A list shorter than the placeholders leaves blanks; extra values are ignored. Neither `%` nor `*` on strings can be lowered by `zengine desugar`.

---
Back to [[ZEXT Constructs - One Example Each]].
