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

> **🦀 From your toolbox →** Think OCaml `let x = 5` (a binding, not a mutable cell), but with an explicit escape hatch. C's `const int x` is the closest analogy for the *default*, except Rust's `mut` annotates the *binding*, not the type — there is no `const`-poisoning of pointer types the way `const char *` differs from `char *`. The analogy to OCaml's `ref` breaks down too: `mut` does not introduce a level of indirection the way `ref`/`:=` does; the variable *is* the storage and you assign through its name.

## Shadowing — and why it is not `mut`

You may bind the same name twice. The second `let` does not mutate the first binding; it creates a brand-new binding that *shadows* the old one for the rest of the scope:

```rust
let count = "  42  ";
let count = count.trim();        // &str, whitespace stripped
let count: u32 = count.parse().unwrap(); // now a u32 — different type!
```

This is exactly OCaml's shadowing, and it is categorically different from `mut`:

- Shadowing introduces a *new variable* (possibly a new type, as above: `&str` then `u32`). `mut` keeps the same variable and forbids changing its type.
- Shadowing requires the `let` keyword every time, so an accidental re-assignment without `let` is still caught.
- After a chain of shadowing transformations, the final binding can be immutable. You get the ergonomics of "transform in place" without giving up the guarantee that the value won't change *after* you're done with it.

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

## Type inference: local Hindley–Milner, not global

Inside a function body Rust infers types the way you expect from OCaml — you rarely annotate locals:

```rust
let v = vec![1, 2, 3];   // Vec<i32>
let total: i64 = v.iter().sum(); // annotation steers `sum`'s output type
```

But the inference is deliberately *local*. **Function signatures are never inferred** — every parameter and the return type must be annotated:

```rust
fn add(x: i32, y: i32) -> i32 { x + y }
```

This is a hard rule, and it is a design choice, not a limitation of the algorithm. OCaml will happily infer the type of `let add x y = x + y` across the whole module. Rust refuses at the function boundary so that (a) type errors stay local and produce comprehensible messages instead of cascading across files, and (b) the public API of a function is documented and stable regardless of its body. The cost is a few annotations; the payoff is that inference within a body can be aggressive precisely because the boundaries are pinned.

> **🎓 Tripos link →** *Foundations of Computer Science.* This is Hindley–Milner with the let-generalisation scoped to the function body and the principal type at each `fn` boundary fixed by annotation rather than inferred. Unlike the global unification you saw in OCaml, Rust solves a constraint set per function, seeded by the signature. The inference is also influenced *backwards* by later use (note how `let guess: u32 = "42".parse().unwrap();` lets `parse` pick its return type from the annotation) — that's unification flowing both directions, not C-style forward-only declaration.

## Scalar types: sized, explicit, non-coercing

| Category | Rust | C analogue | Note for you |
|---|---|---|---|
| Signed int | `i8 i16 i32 i64 i128 isize` | `int8_t … intptr_t` | `i32` is the inference default |
| Unsigned int | `u8 u16 u32 u64 u128 usize` | `uint8_t … size_t` | `usize` is the index/length type |
| Float | `f32 f64` | `float double` | `f64` is the default; IEEE-754 |
| Boolean | `bool` (1 byte) | `_Bool` | only `true`/`false`, no int coercion |
| Character | `char` (4 bytes) | — | a Unicode scalar, not a C `char` |

Three things that will bite C and OCaml muscle memory:

**Sizes are in the name.** There is no implementation-defined `int`. `i32` is 32 bits everywhere. `isize`/`usize` are pointer-width (the moral equivalent of `ptrdiff_t`/`size_t`), and indexing a collection yields a `usize` — using a different width as an index is a type error, not a silent conversion.

**No implicit numeric coercion. None.** This is the single biggest break from C's arithmetic. `let x: i64 = some_i32;` does not compile; you must write `some_i32 as i64` or `i64::from(some_i32)`. C's integer-promotion and the entire usual-arithmetic-conversions ritual simply do not exist. Mixing an `i32` and a `u8` in `+` is a compile error. The reason is the invariant again: silent widening/narrowing is a classic source of bugs (sign flips, truncation), so Rust forces the conversion to be written, where it is auditable.

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

**Tuples** are anonymous, fixed-arity, heterogeneous products — straight out of OCaml:

```rust
let t: (i32, f64, u8) = (500, 6.4, 1);
let (x, y, z) = t;        // destructuring let
let first = t.0;          // positional access via .N
```

The empty tuple `()` is the **unit type**, and it matters enormously — it is Rust's "no meaningful value," the type of an expression evaluated for effect. (OCaml's `unit`, exactly.) We'll lean on it the moment we discuss expressions below.

**Arrays** `[T; N]` are fixed-length, stack-allocated, homogeneous — like C arrays but with the length as part of the type:

```rust
let xs: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0u8; 16];    // [value; count]: sixteen zero bytes
```

The crucial delta from C: **indexing is bounds-checked.** `xs[10]` does not read adjacent memory; it *panics* with `index out of bounds: the len is 5 but the index is 10`. When the index is a compile-time constant the check is elided; when it comes from runtime data the check happens at runtime. This is the array-out-of-bounds half of "memory safety" made concrete — Rust traps instead of letting you stroll off the end the way C does. The compiler cannot know a user-supplied index in advance, so it inserts a guard; safety beats the saved comparison.

For a growable, heap-backed sequence you want `Vec<T>`, and for a borrowed *view* into either an array or a `Vec` you want a *slice* `&[T]` — these, and why `[T; N]` / `Vec<T>` / `&[T]` form a family, are the subject of [slices and the owned/borrowed duality](05-slices-and-duality.md).

> **🦀 From your toolbox →** `[T; N]` is C's `T arr[N]` with the length promoted into the type (so `[i32; 4]` and `[i32; 5]` are *distinct types* and a function taking one rejects the other — unlike C, where arrays decay to bare pointers and lose their length at the first opportunity). The decay-to-pointer move has no equivalent; Rust keeps the length, which is exactly what makes bounds checks and the `for x in arr` loop below possible.

## The big one: Rust is expression-oriented

If you internalise one thing from this chapter, make it this. Like OCaml and unlike C/Java, **almost everything in Rust is an expression that evaluates to a value.** Blocks, `if`, `match`, and `loop` all produce values. The distinction Rust draws is:

- **Statements** perform an action and evaluate to nothing (`let y = 6;` is a statement; a `fn` definition is a statement).
- **Expressions** evaluate to a value (`5 + 6`, a function call, a macro call, a block `{ … }`).

A `let` binding is a *statement*, and it does **not** itself evaluate to a value — so unlike C, you cannot write `x = y = 6`:

```rust
// let x = (let y = 6);  // error: expected expression, found `let` statement
```

This is the opposite of C, where assignment is an expression returning the assigned value (the source of `if (x = 0)` bugs). Rust amputates that footgun by making `let` a statement.

### Blocks evaluate to their trailing expression

A `{ … }` block is an expression. Its value is the last expression inside it — **written without a trailing semicolon**:

```rust
let y = {
    let a = 3;
    a + 1          // no semicolon: this is the block's value
};                 // y == 4
```

The semicolon is the pivotal piece of punctuation in Rust, and it does the opposite of what it does in C. A trailing semicolon **discards** the value and turns the expression into a statement whose value is `()`. Drop the semicolon and the expression's value becomes the block's value. This rule unifies `let x = { … }` with function return below.

> **🎓 Tripos link →** *Semantics of Programming Languages.* In big-step operational semantics, a Rust block is `⟨e₁; e₂; …; eₙ⟩` where the value of the whole is the value of `eₙ`. A trailing semicolon is the rule `e ; ⤳ ()` — it sequences `e` for its side effects and yields unit. This is precisely OCaml's `e1; e2` sequencing, where `e1 : unit`, generalised: in Rust *every* non-final statement is implicitly forced to type-check as a statement, and the type of the block is the type of its tail expression (or `()` if there is none).

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

**No function overloading.** Unlike C++ and Java, you cannot define two functions with the same name and different parameter types. Names are unique within a scope; the polymorphism you'd reach for overloading to express is done with *generics* and *traits* instead (see [generics and traits](08-generics-and-traits.md)). And there is no `void f(int) {}` vs `void f(float) {}` — one name, one function.

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

This is the idiomatic "retry until success, then hand the result back" construct — there is no equivalent in C, where `break` is value-less. (`while` and `for` do *not* yield values this way; their type is always `()`, because they may run zero times and there'd be no value to produce.)

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

> **🎓 Tripos link →** *Foundations of Computer Science.* This is OCaml's `match … with` — including the exhaustiveness analysis. The C `switch` is the false friend: it has fall-through, no exhaustiveness guarantee, and operates only on integers/enums. Rust's `match` is a total function from the matched value's type to the result type, and the compiler discharges the proof of totality (Curry–Howard: the wildcard `_` is the case that makes the function total). Forget a variant of an `enum` and you get a compile error, not a silent default.

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

1. Predict the compiler's verdict, then check: `let mut s = "hi"; s = s.len();`. What error code, and why is `mut` the wrong tool? Rewrite it so it compiles.
2. Write `fn abs_diff(a: i32, b: i32) -> i32` *without* using `return` and without an explicit `if … return`. Now add a single semicolon somewhere in the body that makes it fail to compile, and state the exact error message you expect.
3. (★) Without running it, give the value and type of each `let` below, or mark it a compile error and name the cause:
   ```rust
   let a = { let x = 2; x * x };
   let b = if a > 3 { a } else { "small" };
   let c = loop { break a + 1; };
   let d = { 7; };
   ```
4. `let big: u8 = 200; let sum = big + big;` — describe what happens in a debug build vs a `--release` build, and rewrite the line three ways using the explicit overflow-method families so that (a) it clamps, (b) it signals failure, (c) it wraps deliberately.
5. Rewrite this index loop as an idiomatic `for`, and explain in one sentence what runtime work the compiler is now free to omit:
   ```rust
   let v = [3, 1, 4, 1, 5];
   let mut i = 0;
   while i < v.len() { println!("{}", v[i]); i += 1; }
   ```
6. (★) Using a labeled `loop`, write a search over a 2-D array `[[i32; 4]; 4]` that, on finding the first element greater than 10, breaks out of *both* loops and binds that element to a `let` outside the loops. (Hint: combine a loop label with `break`'s value-carrying form. Why can't you do this with `for`?)
