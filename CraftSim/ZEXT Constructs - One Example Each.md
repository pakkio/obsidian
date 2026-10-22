---
tags:
  - craftsim
  - zext
  - zengine
  - examples
  - constructs
aliases:
  - ZEXT constructs
  - one example per construct
---

# ZEXT Constructs - One Example Each

← Back to [[CraftSim]] | Reference: [[ZEXT Language Extensions]] | Short tour: [[ZEXT Sugar and Async - Worked Examples]]

One realistic example for every ZEXT construct listed in [[ZEXT Language Extensions]], each a small program rather than a one-liner. **All 29 were run on the live simulator on 2026-10-17** with the MCP tool `zengine_run` (console: `zengine run <file> --wait <ms>`); the output under each is what it printed. The files are in the repo under `tasks/zext/constructs/` (`c01_...` to `c29_...`).

Along the way the examples exposed behaviour that the reference pages did not state, and one bug; those are in the comments after the outputs and in [[Known and Unresolved Problems]].

**Not covered**

- `#include` (needs a library script in a prim's inventory or an asset UUID, plus `[ZEngine] EnableInclude = true`; the throwaway prim of `zengine run` has neither) and `#STEP` / `#STOP` (an interactive debugger that opens a dialog and waits for the owner). Both are described in [[ZEXT Language Extensions]] section 3 and were not run here.
- The `->` bind inside a user function is shown (24). In an `if` / `else`, a loop or a `switch` it runs, and (since 2026-10-22) `zengine desugar` lowers it too, by flattening the handler into a dispatch function: [[ZEXT Arrows - Async Bind, Fat Arrow and Timers]].

## Parts

This note was split because it had grown too long. Read the parts in order or jump to the one you need.

- [[ZEXT Constructs - One Example Each - Values and operators]]: booleans, decimals, ternary, `??`, pipeline, bitwise compound assignment.
- [[ZEXT Constructs - One Example Each - Strings, indexing and slices]]: indexing, slices, map literals, string literals, f-strings, `%` and `*`.
- [[ZEXT Constructs - One Example Each - State arguments and preprocessor]]: state parameters, `#define` family, `#TRON`/`#TROFF`.
- [[ZEXT Constructs - One Example Each - Control flow and destructuring]]: `switch`, `break`/`continue`, `foreach`, destructuring, closures.
- [[ZEXT Constructs - One Example Each - Async binds and timers]]: `->` binds, `await { }`, named and inline timers.

## Summary of what these examples taught

| Finding | Where |
| :--- | :--- |
| Negative **list** index returns an empty value (strings are fine); `zengine desugar` lowers it correctly | 7 |
| `decimal` cannot be mixed with float literals; equality with a float literal is false | 2 |
| Escapes still apply inside `"""..."""`; use `r"..."` for literal backslashes | 12 |
| `%` formats only `%d %i %s %f %%`; width and precision flags are left as typed | 15 |
| State parameters are visible in `state_entry` only; a state-to-itself transition does not re-enter | 16 |
| Closures only exist as the last argument of `llSetTimerEvent`, `llSensor`, `llListen` | 23 |
| A slice stop is inclusive: `[:2]` is three characters | 8, 10 |
| `key`, `list`, `vector`... cannot be variable names, and the error points elsewhere | [[ZEXT Sugar and Async - Worked Examples]] |
