---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
aliases:
  - classes
  - generics
  - templates
  - "<T>"
---

# ZEXT classes and `<T>` generic templates

← Back to [[ZEXT Syntax Sugar Index]] | Engine: [[ZEngine]] | Lists: [[ZEXT For Comprehensions (Scala)]] | Added 2026-11-20

> **Since 2026-11-21:** inheritance, traits, abstract classes, `super`, class parameters, case classes, implicits and extension methods: see [[ZEXT Scala-style Classes - Inheritance, Case Classes, Implicits]]. Some limits listed below were lifted then (marked).

```lsl
class Point {
    float x;
    float y = 1.0;                                   // field initialiser
    float len() { return llSqrt(x * x + y * y); }    // method
    move(float dx) { x += dx; }                      // a method that changes its object
    Point scaled(float k) { return Point(x * k, y * k); }
}

class Stack<T> {                                     // generic class
    list items;
    push(T v) { items += [v]; }
    T pop() { T v = items[-1]; items = llDeleteSubList(items, -1, -1); return v; }
    integer size() { return llGetListLength(items); }
}

T maxOf<T>(T a, T b) { if (a > b) return a; return b; }   // generic function

Point mid(Point a, Point b) { return Point((a.x + b.x) / 2, (a.y + b.y) / 2); }

default { touch_start(integer n) {
    Point p = Point(3.0, 4.0);          // no constructor written: one argument per field, in order
    p.move(1.0);
    llOwnerSay((string)p.len() + " " + (string)mid(p, Point(0.0, 0.0)).x);
    Stack<string> s = Stack<string>();
    s.push("a"); s.push("b");
    llOwnerSay(s.pop() + (string)maxOf<integer>(3, 7));
} }
```

## Classes

| Member | Form |
| :--- | :--- |
| field | `type name;` or `type name = init;` (any LSL type, including `list`, or another class) |
| constructor | `ClassName(params) { ... }`, at most one. Without one, `ClassName(f1, f2, ...)` takes every field in order and `ClassName()` gives the initialisers/defaults |
| method | `type name(params) { ... }` or `name(params) { ... }` (no result); a class return type is allowed |

Inside a method or constructor, fields are named directly (`x`) or as `this.x`; a local or parameter with the same name hides the field. Methods can call each other directly (`push(v)`) or as `this.push(v)`.

A declared object with no initialiser (`Point p;`) holds the field initialisers. Objects can be locals, globals, parameters, return values and fields of other objects (`line.b.move(1.0)`).

### Value semantics

**An object is a value, like a vector**: `Point b = a;` copies, and changing `b` leaves `a` alone. Passing an object to a function passes a copy; a method that changes its object changes the variable it was called on. There are no references, no sharing and no `null`.

Why: underneath, an object is one plain variable per field (`p` is `p__x` and `p__y`; nested objects `line__a__x`). That is what makes `list` fields possible (LSL lists cannot hold lists), keeps field access as fast as a variable, lets objects be saved and restored with the script state like any other variable, and lets `zengine desugar` print stock LSL.

### What is refused (compile errors with a message)

- ~~putting an object in a list (`[p]`)~~ **allowed since 2026-11-21**: an object in a list adds its fields, and a `list<Point>` gives them back as objects - see [[ZEXT Scala-style Classes - Inheritance, Case Classes, Implicits#Lists of objects]];
- `a == b` / `a != b` on objects: compare fields (**allowed for case classes since 2026-11-21**);
- a global object with an initialiser computed at run time: assign it in `state_entry`. A global `Point g;` is fine, and constant field initialisers apply to it (**since 2026-11-21 a constructor with constant arguments is allowed**: `Point g = Point(1.0, 2.0);`);
- a class containing itself (`class A { A inner; }`);
- inside a `while` / `do` / `for (;;)` condition or update, or inside `&&` / `||` / `?:`: a call to a method that **changes its object**, or anything that **produces an object** (`f(p).x`, `p.scaled(2).len()`). Those are computed into a temporary in front of the statement, which a loop condition cannot have and a conditional operand would run unconditionally. Methods that only read their object (`s.size()`) are fine everywhere: `while (s.size() > 0) s.pop();` works;
- passing an object to an LSL/OSSL function (`llOwnerSay(p)`) (**since 2026-11-21 allowed when the class has `string toString()`**, which case classes get automatically).

~~No inheritance, interfaces or virtual methods.~~ Since 2026-11-21: `extends`, traits (`with`), abstract methods and overriding - see [[ZEXT Scala-style Classes - Inheritance, Case Classes, Implicits]].

## Generics: `<T>` templates

`class Name<T, U, ...> { ... }` and `[ret] name<T, ...>(...) { ... }` at the top level are **templates**, C++ style: each distinct use `Name<integer>` / `Name<Box<string>>` / `maxOf<float>(…)` produces a copy with the parameters replaced, named `Name__integer` (`Box__Box__string`, …). Type arguments are LSL types (not `decimal`) or class names, and are always written: `maxOf(3, 7)` is an error, `maxOf<integer>(3, 7)` is right. `>>` closing two levels (`Box<Box<integer>>`) works.

Because a copy is compiled for each type, an operation the type does not support shows up as an ordinary compile error in that copy, reported at the template's line (`maxOf<vector>` fails on `>`). A generic function cannot (yet) return a generic type (`Box<T> wrap<T>(T v)`); a method of a generic class can.

## How it is lowered

1. **Templates** (`ZTemplateExpander.cs`), on the token stream before parsing: templates are lifted out, every use is replaced by the mangled name, and each instantiation is appended to the stream as an ordinary class or function. Nothing after this sees `<T>`.
2. **Classes** (`ZClassLowering.cs`), right after parsing:
   - each object variable, parameter and global becomes one variable per field;
   - a method becomes `Class__m(this__x, this__y, ..., params)`; a constructor `Class__new(...)`; the initialisers `Class__defaults()`;
   - an object result travels through the globals `__zr_Class__f`, and a method that changes its object (found by a fixed point over the methods: it assigns a field, or calls such a method on itself or on one of its object fields) copies its fields to `__zt_Class__f` on return, which the call site copies back.

```lsl
p.move(1.0);                 →  Point__move(p__x, p__y, 1.0); p__x = __zt_Point__x; p__y = __zt_Point__y;
float l = p.len();           →  float l = Point__len(p__x, p__y);
Point m = mid(a, b);         →  mid(a__x, a__y, b__x, b__y); m__x = __zr_Point__x; m__y = __zr_Point__y;
integer v = c.next() + 1;    →  integer __zv0 = Counter__next(c__n, c__step); c__n = __zt_Counter__n; ...; integer v = __zv0 + 1;
```

Type inference, the inliner, codegen, `zengine desugar` and state persistence all see only this. Tests: `ZClassTest` (11 at first, 21 with the 2026-11-21 features), `ZDesugarTest.TestClassesAndGenericsAreEquivalent`.
