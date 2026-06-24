# 1. The Language Delta: Rust as a diff from C and OCaml

You already know how to write a `for` loop. This chapter does not teach you what a variable is. Instead, it reads Rust's surface syntax as a *diff* against two languages you already hold in your head: C (the procedural, statically-typed, manual-memory machine model) and OCaml (the expression-oriented, type-inferred, immutable-by-default ML). Rust is, almost line for line, a deliberate merge of the two — ML's front end welded onto C's back end. Most of the "surprises" in this chapter are just one parent's habit colliding with the other's.

The deep machinery — ownership, borrowing, lifetimes — starts in [ownership](02-ownership-and-moves.md). Here we clear the syntactic ground so that nothing trips you later. The recurring theme, and the answer to almost every "why does Rust do *that*?": **so the compiler can prove memory- and data-race-safety statically, at zero runtime cost.** Hold that as your invariant; we will keep cashing it out.

## Bindings: `let`, immutable by default, and `mut`

A `let` binding looks like C's declaration but behaves like OCaml's `let`:

```rust
let x = 5;        // type inferred as i32
let y: f64 = 2.0; // optional annotation, ML-style: name : type
```

The first divergence from C: **bindings are immutable by default.** `x = 6;` is a compile error, not a warning.

```text
error[E0384]: cannot assign twice to immutable variable `x`
```

To opt into mutation you write `let mut x = 5;`. This is the inverse of C and Java, where everything is mutable unless you sprinkle `const`/`final`. The default is flipped on purpose: an immutable binding is a fact the compiler (and the reader) can rely on, and — looking ahead — immutability is the cheaper, contention-free half of the borrow rules in [references and borrowing](03-references-and-borrowing.md). `mut` is not just a lint; it is load-bearing type information.

> **🦀 From your toolbox →** Closest to Swift's `let` vs `var`: Swift's `let` is the immutable default and `var` is the opt-in to mutation — Rust just renames `var` to `let mut`. Java's `final` and Kotlin's `val` are the same idea, except Rust flips the default so that *not* writing `mut` is the common case (think "everything is `final` unless you say otherwise"). Python has no equivalent — every name is a rebindable reference — so the Rust discipline will feel stricter than `x = 5` in a script. Where the analogy breaks down: Rust's `mut` annotates the *binding*, not the value's type, and unlike Swift's `var` on a class reference, there's no hidden indirection — the variable *is* the storage, and you assign straight through its name.

## Shadowing — and why it is not `mut`

You may bind the same name twice. The second `let` does not mutate the first binding; it creates a brand-new binding that *shadows* the old one for the rest of the scope:

```rust
let count = "  42  ";
let count = count.trim();        // &str, whitespace stripped
let count: u32 = count.parse().unwrap(); // now a u32 — different type!
```

There is nothing like this in Java, Swift, or Python — re-using a name there always *reassigns* the same variable (and in Java/Swift the type is locked, so you couldn't go from `String` to `int` anyway). Rust's shadowing is a genuinely new binding, and it is categorically different from `mut`:

- Shadowing introduces a *new variable* (possibly a new type, as above: `&str` then `u32`). `mut` keeps the same variable and forbids changing its type.
- Shadowing requires the `let` keyword every time, so an accidental re-assignment without `let` is still caught.
- After a chain of shadowing transformations, the final binding can be immutable. You get the ergonomics of "transform in place" without giving up the guarantee that the value won't change *after* you're done with it.

> **🔧 In practice →** Shadowing shines when you parse-and-narrow an input. Picture a request handler that receives a port as text: you want a validated `u16` by the end, but the raw form is a `&str`. Instead of inventing `port_str`, `port_trimmed`, `port_num`, you keep one name and let its type sharpen as you go:
> ```rust
> let port = std::env::var("PORT").unwrap_or_else(|_| "8080".to_string());
> let port = port.trim();              // &str
> let port: u16 = port.parse().unwrap_or(8080); // now a validated u16
> // from here on `port` is the immutable, final value — no leftover string lying around
> ```
> The intermediate `&str` is gone, so nobody downstream can accidentally use the un-parsed version. That "the old shape is unreachable now" guarantee is exactly what `mut` can't give you.

A scope-local shadow reverts when the scope ends:

```rust
let n = 6;
{
    let n = n * 2;     // inner n == 12
    println!("{n}");   // 12
}
println!("{n}");       // 6 again — inner binding is gone
```

> **⚠️ Pitfall →** Coming from C/Java, the instinct is `let mut spaces = "   "; spaces = spaces.len();`. That is `error[E0308]: mismatched types — expected &str, found usize`, because `mut` cannot change the *type* of `spaces`. The fix is shadowing: `let spaces = spaces.len();`. The compiler is telling you these are conceptually two different values that happen to share a name.

## `const` and `static`

Two ways to name a compile-time-known constant, and they are not interchangeable:

```rust
const MAX_RETRIES: u32 = 3 * 5;       // inlined; no fixed address
static GREETING: &str = "hello";      // a single object with a 'static address
```

- `const` requires a type annotation, must be a *constant expression* (evaluated at compile time), and is conceptually *inlined* at each use site — like a C `#define` with real types and scoping, or C++ `constexpr`. It has no stable memory address.
- `static` is a single global with a fixed address that lives for the whole program (its lifetime is `'static`; see [lifetimes](04-lifetimes.md)). A `static mut` exists but is `unsafe` to touch, because a mutable global is a data race waiting to happen — see [unsafe](16-unsafe-and-ffi.md). Reach for `const` by default.

Convention: `SCREAMING_SNAKE_CASE` for both.

## Type inference: local, not global

Inside a function body Rust infers types the way you expect from Swift (`let v = [1, 2, 3]`) or modern Java's `var` — you rarely annotate locals:

```rust
let v = vec![1, 2, 3];   // Vec<i32>
let total: i64 = v.iter().sum(); // annotation steers `sum`'s output type
```

But the inference is deliberately *local*. **Function signatures are never inferred** — every parameter and the return type must be annotated:

```rust
fn add(x: i32, y: i32) -> i32 { x + y }
```

This is a hard rule, and it is a design choice, not a limitation of the algorithm. OCaml will happily infer the type of `let add x y = x + y` across the whole module. Rust deliberately stops at the function boundary so that (a) type errors stay local and produce comprehensible messages instead of cascading across files, and (b) the public API of a function is documented and stable regardless of its body — much like Swift and Java, where method signatures are always spelled out even though locals can be inferred. The cost is a few annotations; the payoff is that inference within a body can be aggressive precisely because the boundaries are pinned.

> **🔧 In practice →** The fact that inference flows *backwards* from how you use a value — not just forwards from where it's created — is what makes `.collect()` and `.parse()` pleasant. You write the target type once on the binding and the method figures out what to produce:
> ```rust
> let port: u16 = "8080".parse().unwrap();          // parse() reads u16 off the annotation
> let names: Vec<String> = rows.iter().map(|r| r.name.clone()).collect();
> ```
> Without the annotation on `port`, Rust can't tell *which* `parse` you meant (parse to `u16`? `i64`? `f32`?) and asks you to say. So the rule of thumb: annotate the binding, and let the call site infer the rest. This is the same "the variable's declared type steers the call" idea you've used with Swift's `let x: Int = ...` and Java's explicit target types.

## Scalar types: sized, explicit, non-coercing

| Category | Rust | C analogue | Note for you |
|---|---|---|---|
| Signed int | `i8 i16 i32 i64 i128 isize` | `int8_t … intptr_t` | `i32` is the inference default |
| Unsigned int | `u8 u16 u32 u64 u128 usize` | `uint8_t … size_t` | `usize` is the index/length type |
| Float | `f32 f64` | `float double` | `f64` is the default; IEEE-754 |
| Boolean | `bool` (1 byte) | `_Bool` | only `true`/`false`, no int coercion |
| Character | `char` (4 bytes) | — | a Unicode scalar, not a C `char` |

Three things that will bite muscle memory built on Java, Swift, or Python:

**Sizes are in the name.** There is no implementation-defined `int`. `i32` is 32 bits everywhere — closer to Java's guaranteed-width `int`/`long` than to C's wobbly `int`. `isize`/`usize` are pointer-width, and indexing a collection yields a `usize` — using a different width as an index is a type error, not a silent conversion. (Swift's `Int` and `Array` index type `Int` is the nearest cousin: an explicit width tied to the platform.)

**No implicit numeric coercion. None.** This is the single biggest surprise if you're used to Java, Swift, or Python. `let x: i64 = some_i32;` does not compile; you must write `some_i32 as i64` or `i64::from(some_i32)`. Java will silently widen an `int` to a `long` for you; Swift makes you write `Int64(x)` (so Swift will already feel familiar here); Python just promotes everything to arbitrary-precision integers. Rust takes the strict-Swift stance and applies it everywhere: mixing an `i32` and a `u8` in `+` is a compile error. The reason is the invariant again: silent widening/narrowing is a classic source of bugs (sign flips, truncation), so Rust forces the conversion to be written, where it is auditable.

```rust
let a: i32 = 1000;
let b: u8 = 5;
// let c = a + b;        // error[E0308]: mismatched types
let c = a + b as i32;    // explicit, fine
```

**`if` takes a `bool`, full stop.** There is no truthiness:

```rust
let n = 3;
// if n { ... }   // error[E0308]: expected `bool`, found integer
if n != 0 { /* ... */ }
```

Unlike C/JS/Python, `0`, `""`, and null-likes are not falsy. This kills a whole class of `if (ptr)` / `if (count)` ambiguities.

### Integer overflow: debug panics, release wraps

What does `255u8 + 1` do? The answer is *profiled*:

- **Debug builds**: overflow is checked and *panics* (a clean, defined abort — see [error handling](11-error-handling.md)). The bug surfaces during development.
- **Release builds** (`--release`): the check is omitted for speed, and arithmetic *wraps* using two's complement (`255u8 + 1 == 0`).

Crucially, **relying on the wrap is considered a bug.** If you actually want a specific behaviour, say so explicitly with the method families: `wrapping_add` (wrap in all modes), `checked_add` (returns `Option`, `None` on overflow), `overflowing_add` (returns `(value, bool)`), `saturating_add` (clamps to min/max).

```rust
let x: u8 = 250;
assert_eq!(x.wrapping_add(10), 4);        // 260 mod 256
assert_eq!(x.checked_add(10), None);      // signals overflow
assert_eq!(x.saturating_add(10), 255);    // clamps
```

> **⚙️ Under the hood →** In debug, rustc emits an LLVM checked-add intrinsic plus a branch to a panic. In release it emits a plain `add` and the overflow is wrapping by definition of two's complement on the hardware. This is *not* C, where signed overflow is undefined behaviour the optimiser may exploit. Rust's signed overflow is always *defined* (panic or wrap), never UB — there is no `-fwrapv` footgun, and the optimiser is never licensed to assume overflow can't happen.

### `char` is a Unicode scalar value

```rust
let a = 'z';
let zed: char = 'ℤ';
let cat = '😻';     // a single char
```

A `char` is **4 bytes** and holds exactly one Unicode scalar value (any code point in `U+0000..=U+D7FF` or `U+E000..=U+10FFFF`, i.e. everything except surrogate halves). This is nothing like C's `char` (a byte) or `wchar_t`. It is *not* a UTF-8 byte and *not* a UTF-8 grapheme either — string iteration and the byte-vs-char-vs-grapheme distinction is a genuine subtlety covered in [collections](10-collections.md). Single quotes are `char`; double quotes are `&str`.

## Compound types: tuples and fixed arrays

**Tuples** are anonymous, fixed-size, mixed-type groupings — the same idea as Swift's `(Int, Double, UInt8)` tuples or Python's `(500, 6.4, 1)`, but with the element types fixed at compile time:

```rust
let t: (i32, f64, u8) = (500, 6.4, 1);
let (x, y, z) = t;        // destructuring let — like Swift/Python tuple unpacking
let first = t.0;          // positional access via .N (Swift uses .0 too)
```

The empty tuple `()` is the **unit type**, and it matters enormously — it is Rust's "no meaningful value," the type of an expression evaluated only for its effect. Think Swift's `Void` (which is literally a typealias for the empty tuple `()`) or the implicit `None`-returning function in Python. We'll lean on it the moment we discuss expressions below.

**Arrays** `[T; N]` are fixed-length, stack-allocated, same-type-throughout — closest to Java's `int[]` or Swift's `Array`, except the length `N` is baked into the *type* and they can't grow:

```rust
let xs: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0u8; 16];    // [value; count]: sixteen zero bytes
```

The crucial point: **indexing is bounds-checked.** `xs[10]` does not read adjacent memory; it *panics* with `index out of bounds: the len is 5 but the index is 10`. This is the same safety net you get from Java's `ArrayIndexOutOfBoundsException`, Swift's array trap, or Python's `IndexError` — Rust just calls it a *panic*. When the index is a compile-time constant the check is elided; when it comes from runtime data the check happens at runtime. The difference from a hand-written C array, where reading past the end silently scribbles over (or reads) whatever memory is next door, is exactly the "memory safety" guarantee made concrete. The compiler cannot know a user-supplied index in advance, so it inserts a guard; safety beats the saved comparison.

For a growable, heap-backed sequence you want `Vec<T>`, and for a borrowed *view* into either an array or a `Vec` you want a *slice* `&[T]` — these, and why `[T; N]` / `Vec<T>` / `&[T]` form a family, are the subject of [slices and the owned/borrowed duality](05-slices-and-duality.md).

> **🦀 From your toolbox →** Unlike Java's `int[]` or Swift's `Array` — where the length is a runtime property of the object — Rust folds the length into the *type*. So `[i32; 4]` and `[i32; 5]` are genuinely *distinct types*, and a function taking `[i32; 4]` flatly rejects a `[i32; 5]`. (Java and Swift would accept any `int[]`/`[Int]` and discover a length mismatch only at runtime, if at all.) Where the analogy breaks down: in C, a bare array hands its length away the instant you pass it — it becomes just a pointer to the first element, and the size is lost. Rust never does that; it keeps the length attached, which is precisely what makes the bounds checks and the `for x in arr` loop below possible. If you've only met C's "arrays are really pointers" rule as a footgun, Rust's stance is "the length is part of who you are."

## The big one: Rust is expression-oriented

If you internalise one thing from this chapter, make it this. Unlike Java or Python — where `if`, loops, and blocks are *statements* that produce nothing — **almost everything in Rust is an expression that evaluates to a value.** This will feel most familiar from Swift, where `switch` and `if` can be used as expressions, and from OCaml, where it's the whole language. Blocks, `if`, `match`, and `loop` all produce values. The distinction Rust draws is:

- **Statements** perform an action and evaluate to nothing (`let y = 6;` is a statement; a `fn` definition is a statement).
- **Expressions** evaluate to a value (`5 + 6`, a function call, a macro call, a block `{ … }`).

A `let` binding is a *statement*, and it does **not** itself evaluate to a value — so unlike C, you cannot write `x = y = 6`:

```rust
// let x = (let y = 6);  // error: expected expression, found `let` statement
```

This is the opposite of Java and C, where assignment is itself an expression returning the assigned value — the source of the classic `if (x = 0)` typo (you meant `==`). Rust removes that footgun entirely by making `let` a statement, so `if x = 0` simply doesn't parse.

### Blocks evaluate to their trailing expression

A `{ … }` block is an expression. Its value is the last expression inside it — **written without a trailing semicolon**:

```rust
let y = {
    let a = 3;
    a + 1          // no semicolon: this is the block's value
};                 // y == 4
```

The semicolon is the pivotal piece of punctuation in Rust, and it does the opposite of what it does in C. A trailing semicolon **discards** the value and turns the expression into a statement whose value is `()`. Drop the semicolon and the expression's value becomes the block's value. This rule unifies `let x = { … }` with function return below.

> **🎓 Tripos link →** *Semantics of Programming Languages* — the intuition, no formalism. Read a block top to bottom: each line that ends in a semicolon is run for its effect and then thrown away (its value becomes `()`); the final line, if you leave the semicolon off, *is* the block's value. That's the whole rule. If you've used OCaml, it's the `e1; e2` sequencing you already know — run `e1` for effect, hand back `e2` — just generalised to a stack of statements followed by one tail expression. The mental model to carry: **semicolon = "and then discard"; no semicolon on the last line = "this is the answer."**

### Functions return their tail expression

There is no `return` keyword on the happy path. The body is a block; the function returns the block's trailing (semicolon-less) expression. The return type follows `->`:

```rust
fn five() -> i32 { 5 }          // returns 5

fn add_one(x: i32) -> i32 {
    x + 1                        // no semicolon ⇒ this is the return value
}
```

`return` exists, but it is for *early* exit only:

```rust
fn classify(n: i32) -> &'static str {
    if n < 0 { return "negative"; }
    "non-negative"
}
```

> **⚠️ Pitfall →** The number-one beginner error, and it hits OCaml veterans too: `fn add_one(x: i32) -> i32 { x + 1; }`. The stray semicolon turns `x + 1` into a statement, so the block yields `()`, and you get `error[E0308]: mismatched types — expected i32, found ()`, with the helpful note `implicitly returns () as its body has no tail or return expression`. The fix the compiler suggests is literally "remove this semicolon." When a function with a non-`()` return type "returns nothing," look for a rogue semicolon first.

A function with no `->` returns `()` implicitly — that is what `main` and `fn another()` do. This is why `()` is not a curiosity: it is the value of every "void" function and every semicolon-terminated block.

**No function overloading.** Unlike Java, Swift, or C++ — where you can write `f(int)` and `f(float)` as two functions sharing a name — Rust lets a name refer to exactly one function within a scope. The "same operation, many types" need that you'd solve with overloading (or with Java/Swift method overloads) is met instead by *generics* and *traits* (see [generics and traits](08-generics-and-traits.md)). One name, one function; the polymorphism moves into the type parameters.

## Control flow as expressions

### `if` is an expression

Because `if`/`else` produces a value, it replaces C's ternary `?:` (which Rust deliberately omits):

```rust
let parity = if n % 2 == 0 { "even" } else { "odd" };
```

Both arms must have the **same type** (`if cond { 5 } else { "six" }` is `error[E0308]: if and else have incompatible types`), because the binding needs one statically known type. An `if` without `else` has type `()`, so it can only be used in value position when its single arm is also `()`.

### `loop` is an expression that can yield a value

`loop { … }` is an unconditional infinite loop. Uniquely, `break` can carry a value *out* of it, making `loop` an expression:

```rust
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 { break counter * 2; }  // break WITH a value
};                                            // result == 20
```

This is the idiomatic "retry until success, then hand the result back" construct — there's no direct equivalent in Java, Swift, or Python, where `break` carries no value and you'd need a mutable variable declared *outside* the loop plus a flag. (`while` and `for` do *not* yield values this way; their type is always `()`, because they may run zero times and there'd be no value to produce.)

> **🔧 In practice →** This is the clean way to write a retry-with-backoff. Say you're polling a flaky service and want the response back once it succeeds, without a mutable `result` variable hanging around afterwards:
> ```rust
> let mut attempt = 0;
> let response = loop {
>     attempt += 1;
>     match try_fetch() {
>         Ok(body) => break body,                 // hand the success out of the loop
>         Err(_) if attempt < 5 => continue,      // try again
>         Err(e) => break_on_giving_up(e),        // (or break with a fallback)
>     }
> };
> // `response` is now a plain, immutable binding holding the value the loop produced
> ```
> Compare the Java/Swift shape: declare `String response = null;` before the loop, mutate it inside, and hope every path sets it. Rust's `break body` makes the loop itself the expression that produces `response`, so the binding can stay immutable and there's no "did I forget to assign it?" gap.

### `while` is the conditional loop

```rust
let mut n = 3;
while n != 0 {
    println!("{n}!");
    n -= 1;
}
```

Same as C's `while`, minus the truthiness — the condition must be `bool`.

### `for` iterates over iterators, not indices

This is a real break from the C `for (i = 0; i < n; i++)` idiom. Rust's `for` is the iterator-based `foreach` you know from OCaml/Java/Swift — there is no three-clause C form at all:

```rust
let xs = [10, 20, 30, 40, 50];
for x in xs {               // x takes each element by value
    println!("{x}");
}

for i in 0..5 { /* 0,1,2,3,4 */ }      // a Range is an iterator
for i in (1..4).rev() { /* 3,2,1 */ }  // ranges compose with iterator adapters
```

The reason this is the *idiomatic and faster* choice (not merely the safe one): iterating by index, `while i < xs.len() { xs[i] }`, forces a bounds check on every access, and a fencepost slip is a panic waiting to happen. The `for x in xs` form is driven by the iterator protocol, knows it stays in bounds, and the compiler routinely elides the checks entirely. Iterators are a centrepiece of idiomatic Rust and get their own treatment in [closures and iterators](12-closures-and-iterators.md).

### Labeled loops

`break`/`continue` target the innermost loop by default. To break out of an outer loop, label it with a leading-quote name:

```rust
'outer: for i in 0..5 {
    for j in 0..5 {
        if i * j > 6 { break 'outer; }   // exits BOTH loops
    }
}
```

Labels also combine with the value-carrying break: `break 'outer some_value`. (`return`, by contrast, always unwinds the whole *function*, not just a loop.)

## A 5-line taste of `match`

`if`/`else if` chains are the wrong tool once you're discriminating over several cases. Rust's `match` is a value-producing expression with **exhaustiveness checking** — the compiler rejects your code if you forget a case. A teaser; the full power (over enums, with binding and guards) is in [enums and pattern matching](07-enums-and-pattern-matching.md):

```rust
let label = match n {
    0 => "zero",
    1 | 2 => "small",
    3..=9 => "medium",
    _ => "large",        // the wildcard; required here for exhaustiveness
};
```

> **🎓 Tripos link →** *Foundations of Computer Science.* This is OCaml's `match … with`, exhaustiveness check and all — and it's the close cousin of Swift's `switch`, which also forces you to handle every case (or write `default`). The false friend is Java's / C's `switch`: it falls through unless you `break`, makes no promise that you covered every case, and works only on a handful of types. Rust's `match` insists that *some* arm fires for every possible value — the wildcard `_` is how you say "everything else." Forget a variant of an `enum` and you get a compile error, not a silent fall-through to nothing. The plain takeaway: the compiler turns "did I handle every case?" from a code-review worry into a guarantee.

## Comments and a note on `println!`

Line comments are `//`; block comments `/* … */` nest. Doc comments `///` and `//!` feed `rustdoc` and are covered in [testing and tooling](20-testing-and-tooling.md).

You have seen `println!("{x}")`. The trailing `!` means it is a **macro**, not a function — the formatting string is parsed and type-checked *at compile time*, which is why `{x}` can capture the variable `x` directly and why a wrong number of arguments is a compile error, not a runtime crash like C's `printf`. Macros get a full treatment in [macros](18-macros.md); for now, read `!` as "this is a macro."

## Mental-model recap

- **Bindings are immutable by default; `mut` is type-level information, not a hint.** Shadowing (`let` again) makes a *new* binding that may even change type — categorically different from `mut`, which keeps the same variable and the same type.
- **There is no implicit numeric coercion and no truthiness.** Conversions are written with `as`/`from`; `if` demands a `bool`. Integer overflow panics in debug and wraps (defined, never UB) in release — and relying on the wrap is a bug; say `wrapping_add`/`checked_add` if you mean it.
- **Rust is expression-oriented like OCaml: blocks, `if`, `match`, and `loop` all yield values.** The trailing-semicolon-or-not rule is the pivot: no semicolon ⇒ the expression is the block's/function's value; semicolon ⇒ discard it and yield `()`.
- **Functions return their tail expression; `return` is for early exit only. There is no overloading.** A non-`()` return type plus "returns nothing" almost always means a stray semicolon.
- **`for` iterates over iterators (and ranges), never C-style indices**, which is both safer (no fencepost panics) and faster (elided bounds checks). `loop` can carry a value out via `break value`; `while`/`for` cannot.

## Exercises

These go a little beyond what the chapter spelled out — most ask you to *predict* what the compiler will say, *adapt* an idea to a new shape, or *choose* between two tools and justify it.

1. **Which of these compile, and why?** For each line decide whether it's accepted or rejected, and name the rule in play. (Assume each is its own `main`.)
   ```rust
   let a = if true { 1 } else { 2 };          // (i)
   let b = if true { 1 };                      // (ii)
   let c: i64 = 5_i32;                          // (iii)
   let d = 5_i32 + 5_i64;                       // (iv)
   let e = if 1 { "yes" } else { "no" };        // (v)
   ```
   For the ones that fail, what's the one-line fix that keeps the obvious intent?

2. **`mut` vs shadowing — a design call.** You're cleaning a user-supplied string: trim it, then later you also need its length as a number. Sketch the bindings two ways — once with `let mut`, once with shadowing — and say which you'd ship and why. (Hint: think about what type the final binding has, and whether anything downstream could still reach the raw, un-trimmed string.)

3. **Predict the overflow behaviour.** Given `let big: u8 = 200;`, state what happens for each of the following in a *debug* build and in a `--release` build, and pick which one you'd actually write if the value is a running byte counter that should stop counting once it maxes out (and say why):
   ```rust
   let x = big + big;                 // (a)
   let y = big.wrapping_add(big);     // (b)
   let z = big.saturating_add(big);   // (c)
   let w = big.checked_add(big);      // (d)  — what's the type of `w`?
   ```

4. **Adapt the loop, then explain the speedup.** Rewrite the index loop below as an idiomatic `for`, then explain in one sentence what runtime work the `for` version lets the compiler skip that the indexed version forces it to do on *every* iteration.
   ```rust
   let v = [3, 1, 4, 1, 5];
   let mut i = 0;
   while i < v.len() { println!("{}", v[i]); i += 1; }
   ```

5. (★) **When does `loop`-with-value beat `while`?** You want to read lines until you hit one that parses as a positive integer, and bind *that integer* to an immutable variable for the rest of the function. Write it with a `loop` that uses `break value`. Then describe the shape you'd be stuck with if you tried the same thing with `while` (what extra variable would you need, and what guarantee would you lose?). Bonus: why can a `for` loop never stand in for this `break value` pattern?
