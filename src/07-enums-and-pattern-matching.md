# 7. Enums & Pattern Matching

You already know this construct under another name. In OCaml you wrote `type shape = Circle of float | Rect of float * float`; in Foundations you destructured it with `match`. Rust's `enum` *is* the OCaml variant — a **sum type**, a tagged union — wearing C-family syntax. So the conceptual delta in this chapter is small but the engineering payoff is enormous: Rust takes the OCaml sum type, gives it a guaranteed memory layout, makes its constructors first-class functions, fuses it with the ownership system from [ownership](02-ownership-and-moves.md), and then uses *exhaustiveness checking* — a totality argument straight out of your Semantics course — to statically eliminate two of the most expensive bug classes in the C/Java world: the null dereference and the unhandled case.

If [structs](06-structs-and-methods.md) are Rust's product types (a value of `Point` is an `x` **and** a `y`), enums are its sum types (a value of `Shape` is a `Circle` **or** a `Rect`). Everything else in this chapter follows from taking that "or" seriously.

## Enums are sum types with a runtime tag

Here is the definition that anchors the whole chapter:

```rust
enum Shape {
    Dot,                       // unit-like: no payload
    Circle(f64),               // tuple-like: one anonymous field
    Rect { w: f64, h: f64 },   // struct-like: named fields
}
```

Three styles of variant, mirroring the three struct flavours from [structs](06-structs-and-methods.md): a *unit-like* variant carries no data, a *tuple-like* variant carries positional fields, and a *struct-like* variant carries named fields. Crucially, different variants of the **same** enum may carry **different** payloads. That is the property a struct cannot give you: a struct with a `kind` tag plus an optional payload field would have to allocate room for every possible payload at once and trust you to keep `kind` consistent with the payload. The enum makes the tag and the payload inseparable — the type system guarantees you can never read the `Rect` fields out of a value that is actually a `Circle`.

Construct values by naming the variant under the enum's namespace:

```rust
let a = Shape::Dot;
let b = Shape::Circle(2.0);
let c = Shape::Rect { w: 3.0, h: 4.0 };
```

A subtle, useful fact: the tuple-like constructor `Shape::Circle` is *itself a function* of type `fn(f64) -> Shape`. You can pass it to a higher-order function exactly as you would in OCaml:

```rust
let radii = [1.0, 2.0, 3.0];
let circles: Vec<Shape> = radii.into_iter().map(Shape::Circle).collect();
```

> **🦀 From your toolbox →** This is OCaml's `type` with `|` variants, almost beat for beat — and the `Shape::Circle`-as-function trick is exactly OCaml treating a constructor as a function in `List.map Circle xs`. The analogy breaks down against C and Java. A C `enum` is just a named integer; it carries no payload and offers no type safety. The faithful C analogue is a `struct { enum tag; union { ... }; }` discriminated union — but C will not stop you reading the wrong union arm, which is exactly the unsoundness Rust closes. Java has no sum types until sealed interfaces (Java 17+); before that you reached for the visitor pattern or a class hierarchy, both of which scatter one logical type across many files.

> **⚙️ Under the hood →** An enum is laid out as a *discriminant* (the tag, an integer wide enough to count the variants) followed by enough space for the largest variant's payload. `Shape` above is roughly `{ tag: u8, payload: [the bigger of f64 and (f64,f64)] }`, so `size_of::<Shape>()` is dominated by the `Rect` arm plus tag and alignment padding — every `Shape`, even a `Dot`, occupies that worst-case size. This is the same trade-off as a C tagged union, except rustc both computes the layout and inserts the tag checks the C programmer had to write by hand. The exact discriminant type and ordering are unspecified unless you pin them with `#[repr(...)]` (see [unsafe and FFI](16-unsafe-and-ffi.md)).

## `Option<T>`: the null pointer, made honest

Now the single most important enum in the language. It is defined in the standard library as nothing more exotic than:

```rust
enum Option<T> {
    None,
    Some(T),
}
```

A generic sum type (we cover generics properly in [generics and traits](08-generics-and-traits.md); read `<T>` here the way you read OCaml's `'a option`). `Option<T>` is so fundamental it is in the prelude — you write `Some`/`None` without qualification — and it is Rust's answer to what Tony Hoare called his "billion-dollar mistake": the null reference.

In C, every `T*` is implicitly nullable, and the compiler does nothing to make you check. In Java every reference is implicitly nullable, and the NPE is a *runtime* event. Rust's move is to make absence a *value of a distinct type*: a `String` is always a real string, and "maybe a string" is a different type, `Option<String>`, with a different shape. You cannot accidentally use an `Option<String>` where a `String` is wanted — the types don't unify — so the compiler forces you to handle the `None` case *before* you can touch the inner value. The billion-dollar mistake becomes a compile error.

```rust
fn first_word(s: &str) -> Option<&str> {
    s.split_whitespace().next()   // None if the string is all whitespace
}

// You cannot do `first_word(s).len()` — Option<&str> has no `.len()`.
// You must first establish that there *is* a word:
match first_word("  hello world") {
    Some(w) => println!("first word: {w}"),
    None    => println!("(empty)"),
}
```

`Result<T, E>` is the same idea applied to fallible operations: `enum Result<T, E> { Ok(T), Err(E) }`. We mention it here only so the shape is familiar; the full treatment, including the `?` operator, lives in [error handling](11-error-handling.md).

> **🎓 Tripos link →** In Semantics, a type system is *sound* when "well-typed programs do not go wrong" — the progress-and-preservation argument. `Option<T>` is exactly how Rust extends that guarantee to cover absence. "Dereferencing null" is a way for a program to go wrong; by reifying absence into the type, Rust pushes that failure mode out of the set of well-typed programs entirely. Java's `null` is the classic counterexample where the type system *fails* to be sound in this respect: `String s` claims to be a string but may be `null`, and the unsoundness is only caught dynamically.

## `match`: an expression, and a totality check

`match` compares a *scrutinee* against a sequence of *arms*, each arm being a pattern, an optional guard, and an expression. The first arm whose pattern matches wins, and its expression becomes the value of the whole `match`.

```rust
fn area(s: &Shape) -> f64 {
    match s {
        Shape::Dot            => 0.0,
        Shape::Circle(r)      => std::f64::consts::PI * r * r,
        Shape::Rect { w, h }  => w * h,
    }
}
```

Three things to internalise immediately, because they distinguish `match` from C's `switch`:

1. **It is an expression, not a statement.** `area` has no `return`; the `match` *is* the function body. This is the OCaml/expression-oriented model from Foundations, not the C statement model. Every arm must produce a value of the same type.

2. **No fall-through.** Unlike C's `switch`, arms do not fall through to the next; there is no `break`. Each arm is self-contained.

3. **It is exhaustive — and that is checked at compile time.** You must handle every possible value of the scrutinee. Omit the `Dot` arm and the program does not compile:

```text
error[E0004]: non-exhaustive patterns: `&Shape::Dot` not covered
```

This exhaustiveness check is the single most valuable property of `match`. It is a **totality** argument: the compiler proves your function is defined on every inhabitant of the input type. When you later add a `Triangle` variant to `Shape`, every `match` that has not been updated becomes a compile error pointing you at exactly the code that needs attention. In a Java `switch` or a chain of `instanceof`, that same change fails silently and you discover the gap in production.

> **🎓 Tripos link →** This is precisely OCaml's "this pattern-matching is not exhaustive" warning from Foundations, promoted from a warning to a hard error. Formally it is a coverage/totality check over the inhabitants of an algebraic data type — the compiler verifies that the patterns partition the value space. Tie it to Semantics: making the function *total* (defined everywhere on its domain) is what lets `match` be sound as an expression — there is no "stuck" configuration where no arm applies.

### Binding, catch-alls, and `_`

Arms can bind parts of the scrutinee to names (that is how `r`, `w`, and `h` above came into scope). To make a `match` exhaustive without enumerating everything, end with a catch-all:

```rust
fn classify(n: u8) -> &'static str {
    match n {
        0       => "zero",
        1 | 2   => "small",     // `|` is the pattern-or operator
        3..=9   => "single digit",
        other   => {            // binds the value, for use in the arm
            if other >= 100 { "big" } else { "medium" }
        }
    }
}
```

A bare identifier like `other` is an *irrefutable* pattern (defined below) that matches anything and binds it. If you need a catch-all but do not need the value, use `_`, the wildcard, which matches anything and binds *nothing*:

```rust
match config_flag {
    1 => enable(),
    _ => {}          // ignore everything else, bind nothing
}
```

The difference between `_` and `_name` matters for ownership: `_name` still *binds* (and therefore can move) the value, whereas `_` never binds. Matching `Some(_s)` against an `Option<String>` moves the `String`; matching `Some(_)` does not.

## All the pattern syntax, in one place

Everything above used patterns; now we catalogue the full grammar. Patterns describe the *shape* of data, and the same syntax works in `match`, `if let`, `while let`, `let`, function parameters, and `for`.

**Literals and ranges.** Match against concrete values, or inclusive ranges with `..=`:

```rust
match byte {
    b'a'..=b'z' => "lowercase",
    b'0'..=b'9' => "digit",
    0           => "nul",
    _           => "other",
}
```

Ranges in patterns are restricted to integers and `char`, because those are the only types for which rustc can decide at compile time whether a range is empty (and thus reason about exhaustiveness). Note the `..=` (inclusive); the half-open `..` is *not* a pattern range — it means something else, below.

**Alternation with `|`.** `1 | 2 | 3` matches any of the three. Combined with `@` and guards it composes (see below); note its precedence interacts with guards — `4 | 5 | 6 if cond` parses as `(4 | 5 | 6) if cond`, the guard applies to the whole alternation.

**Destructuring structs.** Bind fields by name, optionally renaming or matching literals:

```rust
struct Point { x: i32, y: i32 }

let Point { x, y } = p;                  // shorthand: binds x and y
let Point { x: a, y: b } = p;            // rename to a, b
match p {
    Point { x: 0, y }          => println!("on y-axis at {y}"),
    Point { x, y: 0 }          => println!("on x-axis at {x}"),
    Point { x, .. }            => println!("x is {x}, ignore the rest"),
}
```

**Destructuring enums.** The pattern mirrors the variant's declaration — that symmetry is the whole point:

```rust
match msg {
    Shape::Dot            => 0.0,
    Shape::Circle(r)      => r * r,       // tuple-like → positional
    Shape::Rect { w, h }  => w * h,       // struct-like → named
}
```

**Nested patterns.** Patterns compose to arbitrary depth, which is where they earn their keep against `if`/`else` ladders:

```rust
match point_in_option {
    Some(Point { x: 0, y: 0 }) => "origin",
    Some(Point { x: 0, .. })   => "on y-axis",
    Some(_)                    => "somewhere",
    None                       => "absent",
}
```

**Ignoring with `_` and `..`.** `_` ignores exactly one position; `..` ignores *all the remaining* positions of a tuple, array, or struct. `..` may appear at most once per pattern, because `(.., x, ..)` would be ambiguous about how many elements to skip — and rustc rejects it for that reason:

```rust
let (first, .., last) = (1, 2, 3, 4, 5);   // first = 1, last = 5
```

**`@` bindings.** Bind a value to a name *and* test it against a sub-pattern in one go. Without `@` you can test a range *or* capture the value, but not both:

```rust
match id {
    n @ 1..=9   => println!("single digit {n}"),  // tested AND bound
    n @ 10..=99 => println!("two digits {n}"),
    _           => println!("out of range"),
}
```

**Match guards.** An extra `if` condition after the pattern, able to use the pattern's bindings and to express things no pattern can (like relating two bindings, or comparing against an outer variable):

```rust
match opt {
    Some(x) if x % 2 == 0 => println!("even {x}"),
    Some(x)               => println!("odd {x}"),
    None                  => {}
}
```

Beware: a guard *defeats exhaustiveness analysis*. The compiler cannot reason about arbitrary boolean conditions, so `Some(x) if x % 2 == 0` is not treated as covering any `Some`; you still need a fallback arm. Use guards for refinement, not as your only handler of a variant.

### Shadowing inside patterns — the classic gotcha

Because each `match`/`if let`/`while let` arm opens a new scope, a *bare name* in a pattern introduces a fresh binding that **shadows** any outer variable of the same name. This bites people constantly:

```rust
let x = Some(5);
let y = 10;
match x {
    Some(50) => println!("fifty"),
    Some(y)  => println!("matched, y = {y}"),  // NEW y, shadows outer y; binds 5
    _        => println!("default"),
}
// outer y is still 10 here
```

If your intent was "match when the inner value *equals the outer* `y`", the pattern `Some(y)` does the opposite — it captures the inner value into a new `y`. The fix is a guard against a differently-named binding: `Some(n) if n == y`.

> **⚠️ Pitfall →** Forgetting that patterns bind rather than compare is the most common `match` bug for newcomers from C, where `case y:` would compare. There is no compiler *error* here — the code compiles and runs, it just always takes the `Some(_)` arm. The idiomatic fix is to bind to a fresh name and compare in a guard (`Some(n) if n == y`), or to inline the constant if it is one (`Some(10) => ...`).

## Refutability: which patterns go where

A pattern is **irrefutable** if it matches every possible value of its type (e.g. `x`, `(a, b)`, `Point { x, y }` — they cannot fail), and **refutable** if some value fails to match (e.g. `Some(x)`, `1..=5`, `Shape::Circle(r)`). This distinction is *the* rule that decides which construct a pattern is legal in:

- **Irrefutable only:** `let`, function parameters, `for` loop variables. These contexts have no "didn't match" branch, so a pattern that could fail makes no sense.
- **Refutable (and irrefutable, with a warning):** `if let`, `while let`, `let ... else`. These exist precisely to branch on success/failure, so an irrefutable pattern there is pointless and rustc warns.

This is why you cannot write `let Some(x) = maybe;`:

```text
error[E0005]: refutable pattern in local binding
  = note: `let` bindings require an "irrefutable pattern", like a `struct` or
          an `enum` with only one variant
help: you might want to use `let else` to handle the variant that isn't matched
```

The `let` form must always succeed because there is nowhere to put the failure. The compiler even points you at the fix: `let ... else`.

> **🎓 Tripos link →** Refutability is the static decision procedure behind exhaustiveness from your Semantics/Foundations work, viewed from the other side. An irrefutable pattern is one whose coverage is the entire type — a single arm that is *total* on its own. `let PATTERN = EXPR;` is sound only when `PATTERN` is total, because there is no second arm to handle the gap; the compiler discharges exactly that proof obligation before accepting the binding.

## The conditional family: `if let`, `let ... else`, `while let`, `matches!`

When you care about *one* pattern and want to ignore the rest, a full `match` with a `_ => {}` arm is noise. The conditional forms are sugar over that pattern.

**`if let`** runs a block only when one refutable pattern matches, optionally with `else`:

```rust
if let Some(w) = first_word(line) {
    println!("first word: {w}");
} else {
    println!("blank line");
}
```

You can chain `else if let` for unrelated conditions — more flexible than `match` (which tests one scrutinee) but, critically, *not* exhaustiveness-checked. Drop a case and the compiler stays silent. That is the price of the conciseness; pay it knowingly.

**`let ... else`** is the early-return idiom and, in my view, the most underused construct for newcomers. It binds in the *enclosing* scope on success and *diverges* (returns, breaks, panics) on failure — flattening the dreaded rightward-drift staircase:

```rust
fn config_port(raw: Option<&str>) -> u16 {
    // happy path stays at the top level; failure exits early
    let Some(text) = raw else {
        return 8080;            // the `else` block MUST diverge
    };
    let Ok(port) = text.trim().parse::<u16>() else {
        return 8080;
    };
    port                        // `text` and `port` are live here, unindented
}
```

The `else` block is mandatory and must not fall through to the code after it — it has to `return`, `break`, `continue`, or `panic!`. The bindings introduced by the pattern escape into the surrounding scope, which is exactly what makes it a "guard clause" in the classic sense. Compare this to the OCaml/C habit of nesting the success case ever deeper inside `match`/`if`; `let ... else` lets you assert preconditions linearly and keep the main logic at the left margin.

**`while let`** loops as long as the pattern keeps matching — the idiomatic way to drain something that yields `Some`/`Ok` until it stops:

```rust
let mut stack = vec![1, 2, 3];
while let Some(top) = stack.pop() {   // stops when pop() returns None
    println!("{top}");
}
```

**`matches!`** is a macro that reduces a `match`-against-one-pattern-with-an-optional-guard to a `bool`, ideal in `filter`/`assert` positions where you want a predicate, not control flow:

```rust
let is_vowel = |c: char| matches!(c, 'a' | 'e' | 'i' | 'o' | 'u');
assert!(matches!(opt, Some(n) if n > 3));
```

## Binding modes (match ergonomics) and `ref`

One wrinkle that interacts hard with [borrowing](03-references-and-borrowing.md): when you `match` on a *reference*, you usually want the bindings to be *references into* the original value, not *moves out of* it. Moving a field out of a borrowed value would invalidate the original — and the borrow checker forbids it.

Modern Rust handles this with **match ergonomics**: when the scrutinee is a reference and the pattern is not, bindings automatically take a borrow in the same mode (shared for `&T`, mutable for `&mut T`). This is why matching `&Option<String>` just works:

```rust
let name = Some(String::from("Ada"));
match &name {                          // scrutinee is &Option<String>
    Some(s) => println!("{} chars", s.len()),  // s is &String — borrowed, not moved
    None    => {}
}
println!("{name:?}");                  // name is still fully owned and usable
```

Had you matched `name` itself (not `&name`), the `Some(s)` arm would *move* the `String` out, and the final `println!` would fail to compile. The older, explicit way to force a borrow without ergonomics is the `ref` (and `ref mut`) keyword inside the pattern — `Some(ref s)` says "bind `s` as `&String`". You will see `ref` in older code and occasionally still need it when the source is owned but you want a borrow; reach for matching on `&value` first.

> **⚙️ Under the hood →** Match ergonomics is a purely compile-time elaboration: rustc tracks a "default binding mode" as it descends through reference layers in the pattern and rewrites bare bindings into `ref`/`ref mut` accordingly. No runtime cost, no extra indirection — the emitted code is identical to what you would get writing `ref` by hand. It exists so the common, safe case (borrow the inner data) is also the syntactically lightest, while the move-out case stays explicit and visible.

> **⚠️ Pitfall →** `match val { Some(s) => ..., None => ... }` where `val: Option<String>` and you *use `val` afterwards* gives `error[E0382]: borrow of partially moved value: val` (or "use of moved value"). The arm moved the `String` out. Fix: match on `&val` so ergonomics makes `s: &String`, or use `val.as_ref()` to turn `&Option<String>` into `Option<&String>` before matching. The mental rule: match the reference, not the value, whenever you still need the original.

## Memory layout: niche optimisation

A naive layout of `Option<T>` is "tag byte plus room for a `T`", which would make `Option<&i32>` larger than `&i32` — a real cost given how ubiquitous `Option` is. Rust avoids this with **niche optimisation**: if `T` already has a bit pattern (a *niche*) that can never occur as a valid value, rustc uses that forbidden pattern to encode `None`, with **zero** extra space.

The canonical niche is the null pointer. A `&T` or `Box<T>` is guaranteed non-null, so `None` is represented as all-zero bits and `Some(ptr)` is the pointer itself:

```rust
assert_eq!(size_of::<&i32>(),         size_of::<Option<&i32>>());   // both 8
assert_eq!(size_of::<Box<i32>>(),     size_of::<Option<Box<i32>>>()); // both 8
assert_eq!(size_of::<NonZeroU32>(),   size_of::<Option<NonZeroU32>>()); // both 4
```

So `Option<&T>` is genuinely a nullable pointer at the machine level — you get C's compact representation and Rust's safety, simultaneously, with the null check *enforced by the type*. Types without a spare niche still need a discriminant: `Option<i32>` is 8 bytes (4 for the `i32`, plus a tag word with alignment), because every 32-bit pattern is a valid `i32` and there is nothing to steal for `None`. (`bool` has a niche — only `0` and `1` are valid — so `Option<bool>` fits in one byte.)

> **🦀 From your toolbox →** This recovers, *safely*, the C idiom where you overload a sentinel value to mean "absent" — a `char *` of `NULL`, an index of `-1`, a `find` returning `npos`. In C you and every caller must remember the sentinel and check it by hand; nothing in the type records the convention. Niche-optimised `Option<&T>` gives you byte-for-byte the same representation, but the compiler *forces* the check and there is no way to forget which value is the sentinel. The analogy breaks down on safety, not on layout: the bits are identical, the guarantees are not.

## Mental-model recap

- An `enum` is a **sum type** (OCaml variant / tagged union), laid out as a discriminant plus space for the largest variant. The tag and payload are inseparable, so reading the wrong arm is impossible.
- `Option<T>` reifies absence as a *value of a distinct type*, turning the null dereference from a runtime crash into a compile error. `Result<T, E>` does the same for failure ([error handling](11-error-handling.md)).
- `match` is an **expression** and is **exhaustive** — a compile-time totality check that forces every case to be handled and flags every site when you add a variant.
- **Refutability** decides placement: irrefutable patterns only in `let`/params/`for`; refutable patterns drive `if let`/`while let`/`let ... else`. Use `let ... else` to keep the happy path unindented.
- Match the **reference** (`&val`), not the owned value, when you still need the original — match ergonomics then borrows the inner data instead of moving it.

## Exercises

1. Define `enum Json { Null, Bool(bool), Num(f64), Str(String), Arr(Vec<Json>), Obj(Vec<(String, Json)>) }` and write `fn depth(j: &Json) -> usize` returning the maximum nesting depth. Then add a `Json::Date(...)` variant and observe which `match` sites stop compiling — that is the totality check doing your refactoring for you.

2. Explain why `let Shape::Circle(r) = shape;` fails to compile, quote the error code, and give two distinct fixes (one using `match`, one using `let ... else`). Which one is appropriate when `shape` might legitimately be any variant at runtime?

3. The following compiles and runs but always prints `"other"`. Find the bug without running it, and fix it so it prints `"target"` when `code == limit`:
   ```rust
   let code = 7;
   let limit = 7;
   match code {
       limit => println!("target"),   // why does this not do what it looks like?
       _     => println!("other"),
   }
   ```
   (Bonus: the compiler emits a *warning* here too — what is it, and why?)

4. (★) Write `fn sum_tree(t: &Tree) -> i32` for `enum Tree { Leaf(i32), Node(Box<Tree>, Box<Tree>) }`. Do it by matching on `&t`. Then try writing it by matching on `t` by value and explain the exact borrow-checker error you get, in terms of moving out of a shared reference.

5. (★) Predict `size_of` for each, then verify with a program: `Option<i32>`, `Option<&i32>`, `Option<Box<i32>>`, `Option<Option<&i32>>`, and `Option<bool>`. Explain *each* result in terms of niche optimisation — in particular, why `Option<Option<&i32>>` does **not** stay pointer-sized but `Option<&i32>` does.

6. Refactor a function written as a deeply nested `if let { if let { ... } }` staircase (invent one that parses two `Option<&str>` fields and validates them) into a flat sequence of `let ... else` guard clauses. State precisely what each `else` block must do for the code to compile.
