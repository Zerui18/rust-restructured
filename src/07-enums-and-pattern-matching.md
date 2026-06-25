# 7. Enums & Pattern Matching

You already know this construct under several names. In Swift you wrote `enum Shape { case circle(Double); case rect(Double, Double) }` and switched over it with `switch`. In OCaml it was `type shape = Circle of float | Rect of float * float`. Rust's `enum` *is* that same idea — a value that is **one of several shapes**, each carrying its own payload, wearing C-family syntax. So the conceptual delta in this chapter is small but the engineering payoff is enormous: Rust takes the Swift/OCaml enum, gives it a guaranteed memory layout, makes its constructors first-class functions, fuses it with the ownership system from [ownership](02-ownership-and-moves.md), and then *checks at compile time that you handled every case* — statically eliminating two of the most expensive bug classes in the Java/C world: the null dereference and the unhandled case.

If [structs](06-structs-and-methods.md) bundle several fields together (a value of `Point` is an `x` **and** a `y`), enums express a choice (a value of `Shape` is a `Circle` **or** a `Rect`). Everything else in this chapter follows from taking that "or" seriously.

## Enums are a choice between shapes, with a runtime tag

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

A subtle, useful fact: the tuple-like constructor `Shape::Circle` is *itself a function* of type `fn(f64) -> Shape`. You can pass it to a higher-order function exactly as you would pass a method reference in Java or a function in Swift:

```rust
let radii = [1.0, 2.0, 3.0];
let circles: Vec<Shape> = radii.into_iter().map(Shape::Circle).collect();
```

> **🦀 From your toolbox →** This is Swift's `enum` with associated values, almost beat for beat: `Shape::Circle(2.0)` is Rust's `.circle(2.0)`, and switching over it is Swift's `switch shape { case .circle(let r): ... }`. The `Shape::Circle`-as-function trick mirrors how Swift lets you pass `.circle` where a function is expected. The analogy breaks down against Java and C. Java had no real sum types until sealed interfaces (Java 17+); before that you reached for an `abstract class` hierarchy or the visitor pattern, both of which scatter one logical type across many files and lose the compiler's "did you handle every case" check. A C `enum` is just a named integer — it carries no payload and offers no type safety; the faithful C analogue is a hand-written `struct` holding a tag plus a `union`, but C will not stop you from reading the wrong union arm, which is exactly the hole Rust closes.

> **⚙️ Under the hood →** An enum is laid out as a *discriminant* (the tag, an integer wide enough to count the variants) followed by enough space for the largest variant's payload. `Shape` above is roughly `{ tag: u8, payload: [the bigger of f64 and (f64,f64)] }`, so `size_of::<Shape>()` is dominated by the `Rect` arm plus tag and alignment padding — every `Shape`, even a `Dot`, occupies that worst-case size. This is the same trade-off as a C tagged union, except rustc both computes the layout and inserts the tag checks the C programmer had to write by hand. The exact discriminant type and ordering are unspecified unless you pin them with `#[repr(...)]` (see [unsafe and FFI](16-unsafe-and-ffi.md)).

## `Option<T>`: the null pointer, made honest

Now the single most important enum in the language. It is defined in the standard library as nothing more exotic than:

```rust
enum Option<T> {
    None,
    Some(T),
}
```

A generic enum (we cover generics properly in [generics and traits](08-generics-and-traits.md); read `<T>` here the way you read Swift's `Optional<T>` or Java's `Optional<T>`). `Option<T>` is so fundamental it is in the prelude — you write `Some`/`None` without qualification — and it is Rust's answer to what Tony Hoare called his "billion-dollar mistake": the null reference.

In Java every reference is implicitly nullable, and the NPE is a *runtime* event. In C, every `T*` is implicitly nullable too, and the compiler does nothing to make you check. Rust's move is the same one Swift made with `Optional`: make absence a *value of a distinct type*. A `String` is always a real string, and "maybe a string" is a different type, `Option<String>`, with a different shape. You cannot accidentally use an `Option<String>` where a `String` is wanted — the types don't unify — so the compiler forces you to handle the `None` case *before* you can touch the inner value. Where Swift gives you `if let`/`guard let` to unwrap an optional, Rust gives you `match`/`if let`/`let ... else` to do exactly the same job. The billion-dollar mistake becomes a compile error.

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

> **🔧 In practice →** You're parsing a config file where a `timeout` key is optional. You model the lookup as `config.get("timeout"): Option<&str>` and *cannot* forget that it might be missing, because the type won't let you call `.parse()` on it directly. The natural code reads almost like a sentence: `let timeout = config.get("timeout").and_then(|s| s.parse().ok()).unwrap_or(30);` — "look it up; if present, try to parse it; if any step failed, fall back to 30." This is the Swift `??` default-value habit, but the compiler *enforces* that you account for the missing case rather than trusting you to remember.

`Result<T, E>` is the same idea applied to fallible operations: `enum Result<T, E> { Ok(T), Err(E) }`. We mention it here only so the shape is familiar; the full treatment, including the `?` operator, lives in [error handling](11-error-handling.md).

## `match`: an expression that forces you to cover every case

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

Three things to internalise immediately, because they distinguish `match` from Java's `switch` and C's `switch`:

1. **It is an expression, not a statement.** `area` has no `return`; the `match` *is* the function body. Think of it like Swift's expression-style code or a ternary that scales — every arm must produce a value of the same type, and that value flows out as the result.

2. **No fall-through.** Unlike a classic C/Java `switch`, arms do not fall through to the next; there is no `break`. Each arm is self-contained.

3. **It is exhaustive — and that is checked at compile time.** You must handle every possible value of the scrutinee. Omit the `Dot` arm and the program does not compile:

```text
error[E0004]: non-exhaustive patterns: `&Shape::Dot` not covered
```

This exhaustiveness check is the single most valuable property of `match`: the compiler refuses to accept the function until it handles every shape the input could take. When you later add a `Triangle` variant to `Shape`, every `match` that has not been updated becomes a compile error pointing you at exactly the code that needs attention. In a Java `switch` over an enum, or a chain of `instanceof`, that same change compiles fine and you discover the gap in production — exactly the failure Swift's exhaustive `switch` also exists to prevent.

> **🔧 In practice →** You add a new payment method to an e-commerce backend — a `PaymentMethod::ApplePay` variant alongside `Card` and `BankTransfer`. Without exhaustive `match`, you'd grep for every place that handles payment methods and hope you found them all. With it, you just add the variant and recompile: every `match` that hasn't been taught about `ApplePay` lights up red — the refund calculator, the receipt formatter, the fraud check. The compiler hands you a to-do list of exactly the sites that need updating, and you can't ship until they're all handled.

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

A bare identifier like `other` matches anything and binds it. If you need a catch-all but do not need the value, use `_`, the wildcard, which matches anything and binds *nothing*:

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

Ranges in patterns are restricted to integers and `char`, because those are the only types for which rustc can decide at compile time whether a range is empty (and thus check that you've covered the whole space). Note the `..=` (inclusive); the half-open `..` is *not* a pattern range — it means something else, below.

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

Beware: a guard *defeats the exhaustiveness check*. The compiler cannot reason about arbitrary boolean conditions, so `Some(x) if x % 2 == 0` is not treated as covering any `Some`; you still need a fallback arm. Use guards for refinement, not as your only handler of a variant.

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

> **⚠️ Pitfall →** Forgetting that patterns bind rather than compare is the most common `match` bug for newcomers coming from Java/C, where `case y:` would compare. There is no compiler *error* here — the code compiles and runs, it just always takes the `Some(_)` arm. The idiomatic fix is to bind to a fresh name and compare in a guard (`Some(n) if n == y`), or to inline the constant if it is one (`Some(10) => ...`).

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

## The conditional family: `if let`, `let ... else`, `while let`, `matches!`

When you care about *one* pattern and want to ignore the rest, a full `match` with a `_ => {}` arm is noise. The conditional forms are sugar over that pattern.

**`if let`** runs a block only when one refutable pattern matches, optionally with `else`. This is the direct counterpart of Swift's `if let`:

```rust
if let Some(w) = first_word(line) {
    println!("first word: {w}");
} else {
    println!("blank line");
}
```

You can chain `else if let` for unrelated conditions — more flexible than `match` (which tests one scrutinee) but, critically, *not* exhaustiveness-checked. Drop a case and the compiler stays silent. That is the price of the conciseness; pay it knowingly.

**`let ... else`** is the early-return idiom — Rust's version of Swift's `guard let ... else { return }` — and, in my view, the most underused construct for newcomers. It binds in the *enclosing* scope on success and *diverges* (returns, breaks, panics) on failure — flattening the dreaded rightward-drift staircase:

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

The `else` block is mandatory and must not fall through to the code after it — it has to `return`, `break`, `continue`, or `panic!`. The bindings introduced by the pattern escape into the surrounding scope, which is exactly what makes it a "guard clause" in the classic sense — identical in spirit to Swift's `guard`. Compare this to the habit of nesting the success case ever deeper inside `if let`/`switch`; `let ... else` lets you assert preconditions linearly and keep the main logic at the left margin.

> **🔧 In practice →** You're in a web request handler that needs a logged-in user and a valid order id from the request. Instead of nesting two `if let`s and drifting rightward, you write the preconditions as flat guard clauses: `let Some(user) = session.user() else { return Response::unauthorized(); };` then `let Ok(order_id) = params.get("id").parse::<u64>() else { return Response::bad_request(); };`. Both `user` and `order_id` are now in scope for the rest of the handler, which stays at the left margin and reads top-to-bottom as "here are the things that must be true; now do the work."

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

> **🦀 From your toolbox →** This recovers, *safely*, the old habit of overloading a sentinel value to mean "absent": a pointer of `null`, an index of `-1`, a search returning some magic "not found" constant. In Java or C you and every caller must remember the sentinel and check it by hand; nothing in the type records the convention, which is how the NPE happens. Niche-optimised `Option<&T>` gives you byte-for-byte the same compact representation, but the compiler *forces* the check and there is no way to forget which value is the sentinel. The contrast with Swift is the useful one: Swift's `Optional` also makes absence type-safe, and Rust matches that safety while *also* guaranteeing the cheap pointer-sized layout. The analogy to old C/Java sentinels breaks down on safety, not on layout: the bits are identical, the guarantees are not.

## Mental-model recap

- An `enum` is a value that is **one of several shapes** (Swift enum with associated values / OCaml variant / tagged union), laid out as a discriminant plus space for the largest variant. The tag and payload are inseparable, so reading the wrong arm is impossible.
- `Option<T>` reifies absence as a *value of a distinct type*, turning the null dereference from a runtime crash into a compile error. `Result<T, E>` does the same for failure ([error handling](11-error-handling.md)).
- `match` is an **expression** and is **exhaustive** — a compile-time check that forces every case to be handled and flags every site when you add a variant.
- **Refutability** decides placement: irrefutable patterns only in `let`/params/`for`; refutable patterns drive `if let`/`while let`/`let ... else`. Use `let ... else` to keep the happy path unindented.
- Match the **reference** (`&val`), not the owned value, when you still need the original — match ergonomics then borrows the inner data instead of moving it.

## Exercises

1. **Adapt and observe.** Define `enum Json { Null, Bool(bool), Num(f64), Str(String), Arr(Vec<Json>), Obj(Vec<(String, Json)>) }` and write `fn depth(j: &Json) -> usize` returning the maximum nesting depth. Now add a `Json::Date(...)` variant *without* touching `depth`, and predict before compiling: which lines does rustc flag, and what error code? Was a `_ => ...` catch-all in your `depth` a good idea or a liability for this kind of refactor? Justify your choice.

2. **Which compiles, and why.** For each of the following, decide whether it compiles, and if not, name the failing rule (refutability or move-out):
   - `let Some(x) = compute();` where `compute() -> Option<i32>`
   - `let (a, b) = pair;` where `pair: (i32, String)` and you use `pair` afterwards
   - `for Shape::Circle(r) in shapes { ... }` where `shapes: Vec<Shape>`
   - `if let Point { x, y } = p { ... }` where `Point` has exactly those two fields
   Then state the one-line fix for each that does *not* compile.

3. **Predict the silent bug.** This compiles, runs, and always prints `"other"`. Without running it, explain why, then fix it so it prints `"target"` when `code == limit`:
   ```rust
   let code = 7;
   let limit = 7;
   match code {
       limit => println!("target"),
       _     => println!("other"),
   }
   ```
   The compiler also emits a *warning* here — predict what it is and why it appears.

4. **Design decision.** You're handling an `Option<Config>` in three different functions: (a) one that should fall back to a default if it's `None`, (b) one that should bail out early and return an error if it's `None`, and (c) one that needs to read a field whether or not it's present but must not consume the `Option`. For each, choose between `match`, `if let ... else`, `let ... else`, and matching on `&opt` — and say in one sentence why that form fits best.

5. (★) **Predict the sizes.** Predict `size_of` for each, then verify with a program: `Option<i32>`, `Option<&i32>`, `Option<Box<i32>>`, `Option<Option<&i32>>`, and `Option<bool>`. Explain *each* result in terms of niche optimisation — in particular, why `Option<Option<&i32>>` does **not** stay pointer-sized even though `Option<&i32>` does. (Hint: once the inner `Option` has claimed the null niche, what is left for the outer one to use?)
