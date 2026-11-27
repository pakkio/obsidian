---
tags:
  - craftsim
  - zengine
  - zext
  - lsl
  - syntax
  - scala
aliases:
  - inheritance
  - traits
  - case classes
  - implicits
  - extension methods
  - implicit class
---

# ZEXT Scala-style classes: inheritance, traits, case classes, implicits

← Back to [[ZEXT Syntax Sugar Index]] | Basics: [[ZEXT Classes and Generics]] | Engine: [[ZEngine]] | Added 2026-11-21 (commit `c7cc4a9395`; extension methods after it)

Everything here builds on the classes of [[ZEXT Classes and Generics]]: objects are still **values** (one variable per field, assignment copies), and all of it lowers to stock LSL (`zengine desugar` shows it). Classes that use none of it compile exactly as before.

```lsl
#indent
trait Named:
    string label = "?"
    string name():
        return "<" + label + ">"

trait Loud extends Named:
    string name():
        return llToUpper(super.name()) + "!"

abstract class Shape extends Named:
    float area()                              // no body: abstract
    string describe():
        return name() + " area " + (string)area()

class Circle(float r) extends Shape:          // class parameters
    float area():
        return PI * r * r

class Rect(float w, float h) extends Shape:
    float area():
        return w * h

class Square(float side) extends Rect(side, side) with Loud:
    string describe():
        return "square " + super.describe()

case class Point(float x, float y)

class Config(string level)
implicit Config cfg = Config("debug")

def log(string msg)(implicit Config c):
    print("[" + c.level + "] " + msg)

implicit Point toPoint(vector v):
    return Point(v.x, v.y)

implicit class RichStr(string s):
    integer words():
        return llGetListLength(llParseString2List(s, [" "], []))

default:
    state_entry():
        Shape s = Square(2.0)
        print(s.describe())                   // square <?>! area 4.000000
        Point p = <3.0, 4.0, 0.0>             // implicit conversion
        print(p, p == Point(3.0, 4.0))        // Point(3.0, 4.0) 1
        log("hello")                          // [debug] hello
        print("a b c".words())                // 3
```

## Inheritance and traits

| Form | Meaning |
| :--- | :--- |
| `class C extends B` | single superclass, any number of levels |
| `class C extends B with T1 with T2` | plus traits (mixins); only the first parent may be a class |
| `trait T [extends U with V]` | fields and methods, no constructor; can extend a class (as in Scala's stackable-trait pattern) |
| `abstract class A` | cannot be created; may have abstract methods |
| `float area();` / `float area()` in `#indent` | abstract method (no body) |
| `super.m(...)` | the next implementation after the current class/trait |

**Which method runs** is decided by the **Scala linearization** of the object's concrete class: `L(C extends B with T1 … Tn) = C, L(Tn) +> … +> L(T1) +> L(B)` (where `X +> Y` keeps the members of X not in Y, then Y). The first class or trait in it that has a body for the method wins. A `Shape` variable holding a `Square` runs `Square`'s methods.

**`super` is dynamic for traits**, as in Scala, so traits stack:

```lsl
abstract class IntQueue { integer get(); put(integer x); }
class BasicIntQueue extends IntQueue { list buf; integer get() { ... } put(integer x) { buf += [x]; } }
trait Doubling extends IntQueue { put(integer x) { super.put(2 * x); } }
trait Incrementing extends IntQueue { put(integer x) { super.put(x + 1); } }
class A extends BasicIntQueue with Doubling with Incrementing { }   // put(10) stores 22
class B extends BasicIntQueue with Incrementing with Doubling { }   // put(10) stores 21
```

Same results as Scala. A subclass object can be assigned to a base-typed variable or passed as a base-typed parameter, and keeps its changes (`Rect q = Square(2.0); q.grow(2.0); Shape s = q;`).

### Rules (compile errors with a message)

- a concrete class must implement every abstract method of its linearization, or be declared `abstract`;
- an abstract class or a trait cannot be created (`S s = S();`);
- an override keeps the parameters and the result type;
- **related classes need distinct field names** (they share one layout, see below);
- the classes of a linearization must be one chain (`class C extends A with T` where `trait T extends B` and B is unrelated to A is refused);
- unknown parents, cycles, `super` with no parent implementation.

Not available: downcasts and type tests (`isInstanceOf`), generic classes with `extends`. Objects in lists: see [Lists of objects](#lists-of-objects).

## Class parameters and case classes

```lsl
class Circle(float r) extends Shape("circle") { float area() { return PI * r * r; } }
class Config(string level, integer verbose)          // nothing else: no body needed
case class Point(float x, float y)
Point origin = Point(0.0, 0.0);                       // global from constants: allowed
```

- The parameters are the first **fields** and the **constructor** (a class with parameters cannot also write a constructor).
- `extends B(args)` fills B's parameters; B's own `extends A(...)` then fills A's, and so on up. If the superclass has parameters, a subclass must pass them. This is how a subclass sets up its parent.
- A **global object** can start from a constructor whose arguments are constants (also through parent arguments); anything computed still goes in `state_entry`.
- **`case class`** adds:
  - `toString()`, giving `Point(3.0, 4.0)` (floats without trailing zeros, lists as `[a, b]`, nested case objects printed recursively), unless the class writes its own;
  - `==` / `!=` comparing **every field** (and the concrete class), lists by content. Scala compares only the constructor parameters; here body fields count too.
- **Any class with `string toString()`** turns into that text where a string is wanted: `(string)p`, `print(p)`, `"p=" + p`, an LSL/OSSL argument (`llOwnerSay(p)`).

## Implicits

### Implicit values and parameters

```lsl
implicit Config cfg = Config("debug");                 // global or local
log(string msg)(implicit Config c) { ... }             // Scala's second parameter list
integer twice(implicit integer d) { return d * 2; }    // or inline: everything after 'implicit'
log("a");                    // uses cfg
implicit Config quiet = Config("quiet");
log("b");                    // uses quiet: the innermost scope wins
log("c", cfg);               // explicit still works
```

An omitted implicit argument is the one implicit value of that type in the **innermost scope that has any** (blocks, then function, then globals). None in scope, or two in the same scope, is a compile error naming the candidates. Inside the function, an implicit parameter is itself an implicit value. Works for class types and plain LSL types.

### Implicit conversions

```lsl
implicit Point toPoint(vector v) { return Point(v.x, v.y); }
implicit float toLen(Point p) { return llSqrt(p.x * p.x + p.y * p.y); }
float d = dist(<1.0, 2.0, 0.0>, <4.0, 6.0, 0.0>);   // vectors become Points
Point q = <6.0, 8.0, 0.0>;
float l = q;                                           // Point becomes float: 10.0
```

An `implicit` function with one parameter is applied wherever its result type is expected and a value of its parameter type is given: arguments, declarations, assignments, `return`, casts and `+`. The source type of a plain value is read from literals, declared variables, fields and function results; an integer may promote to a float conversion. Two conversions that both apply are an error. A conversion needs a **class on at least one side** (no scalar-to-scalar conversions).

### Implicit classes (extension methods)

```lsl
implicit class RichStr(string s) { integer words() { ... } string shout() { return llToUpper(s) + "!"; } }
implicit class RichInt(integer n) { integer squared() { return n * n; } }
implicit class PointOps(Point p) { float dot(Point o) { return p.x * o.x + p.y * o.y; } }

"a b c".words();   name.shout();   k.squared();   (3).squared();   a.dot(b);   "x".shout().words();
```

`x.m()` on a value, or on an object whose class has no `m`, becomes `W(x).m()` on a temporary wrapper, when exactly one implicit class with `m` takes x's type (an exact type beats an int→float promotion). The wrapper takes exactly one parameter and extends nothing. Write `(3).squared()`: `3.` on its own reads as a float. Changes a method makes to the wrapper do not reach the original value (as in Scala).

## Lists of objects

Added 2026-11-21. A list declared `list<Point>` holds objects:

```lsl
case class Point(float x, float y);
list<Point> corners = [Point(0, 0), Point(1, 0)];   // global: constant arguments
corners += Point(1, 1);                              // or [a, b]; implicit conversions apply
Point p = corners[0];          float x = corners[-1].x;
corners[0] = Point(9, 9);      corners[1].y += 0.5;
integer n = corners.size();    corners.remove(0);
for (c <- corners) llOwnerSay((string)c);          // Scala for gives objects
list<Point> right = for (c <- corners if c.x > 0) yield c;
list<Shape> shapes = [Sq(2), Circ(1)];              // a family: each keeps its class
float a = shapes[1].area();                          // dispatches on the stored kind
team.members[k].bump();                              // a mutating method writes the element back
```

Underneath it is a plain LSL list of the **fields, one object after another**: `corners` is `[0.0, 0.0, 1.0, 0.0, ...]`, and `corners[k].y` is `llList2Float(corners, k * 2 + 1)`. A family object takes its hidden `__kind` and every member's field, so a `list<Shape>` can mix circles and squares. `list<T>` works for locals, globals, parameters, results and class fields; `for (p <- ps)` knows the class from the declaration (or write `p: Point <- xs`).

- An object in **any** list literal adds its fields, so `llList2CSV([p])` shows them; only a `list<Point>` can give the objects back.
- What goes into a `list<Point>` is checked: plain values, or objects of an unrelated class, are compile errors.
- A class with a **list field** cannot be stored (LSL lists do not nest), and a typed list cannot be **sliced**.
- `remove(k)` is a statement; negative indices count from the end, as in LSL.
- **Cost.** Reads are O(1). Element writes (`ps[k] = ...`, `ps[k].f += ...`, a method that changes `ps[k]`) are **in place** when the variable is the list's only holder: ZEngine's copy-on-write lists (see `.claude/rules/zengine.md`) make `v = llListReplaceList(v, ...)` overwrite v's array instead of copying it, so 8000 writes on an 8000-item list take 3 ms instead of 111 ms. The first write after the list was copied (`qs = ps`, passed to a function, iterated with `for (p <- ps)`) copies once. Appends (`+=`) and `remove` still copy the list.

The chess solver in `tasks/chess_challenge.lsl` (a port of `pakkio/ChessChallenge`) keeps the board as a `list<Placed>` with `case class Placed(Piece piece, Slot at)`.

## How it is lowered (`ZClassLowering.cs`)

- **Families.** Classes and traits joined by `extends`/`with` form a family with **one layout**: a hidden `__kind` (the concrete class, 1..n; 0 = never assigned) followed by every member's own fields. So any family object fits any family variable, and `Shape s = Circle(...)` just copies fields. This is why related classes need distinct field names.
- **Dispatch.** A call whose possible concrete classes all resolve to one implementation calls it directly (`Rect__area`). Otherwise a generated dispatcher switches on `__kind`: `Shape__area__v(...)` for a virtual call, `Doubling__put__super(...)` for `super` inside a trait.
- **Per-family slots.** Object results and write-backs go through `__zr_<Family>__f` / `__zt_<Family>__f`, and "this method changes its object" is decided per family and method name, so an override that changes nothing still writes back when another override does.
- **Constructors.** `Circle__new(r)` sets `__kind`, runs every field initialiser of the linearization, assigns the class parameters, then the parents' parameters from `extends B(args)`.
- **Case classes.** `toString` is generated as source and parsed like user code (floats go through a generated `__zFloatStr`); `==` expands to a field-by-field comparison.
- **Implicits.** `FillImplicits` adds the omitted arguments at the call site; `Coerce` inserts conversion calls (or `toString`) where types meet; `WithExtension` rewrites extension calls onto the wrapper.

```lsl
Shape s = Circle(1.0);
llOwnerSay(s.describe());
// ↓ desugared
Circle__new(1.0);
s____kind = __zr_Named____kind; s__label = __zr_Named__label; s__r = __zr_Named__r; ...
llOwnerSay(Shape__describe__v(s____kind, s__label, s__r, ...));
```

Tests: `ZClassTest` (9 tests for these features, among the class tests); verified on `craft` with `zengine run`. Details for maintainers in `.claude/rules/zengine.md` and `tasks/sugar-todo.md`.
