# 4. Lifetimes as Regions

In [References & Borrowing](03-references-and-borrowing.md) you saw the borrow checker reject a reference that outlived its referent. That chapter waved its hands at *how* the checker knows. The machinery is **lifetimes**, and the single most important sentence in this chapter is this:

**A lifetime is a region — a set of program points over which a reference is valid — and a lifetime annotation `'a` is a generic parameter ranging over such regions. Annotations *describe* constraints between regions; they never *change* them.**

If you internalise that, the rest is bookkeeping. The thing that trips up newcomers is treating `'a` as a runtime quantity ("how long does this thing live?") rather than what it is: a *type-level* variable that the compiler solves for, exactly like a type parameter `T`. You have met this idea before — it just had a different name in a different course.

> **🎓 Tripos link →** This is **region-based memory management** from *Semantics of Programming Languages*. A lifetime `'a` is a region; `&'a T` is the type of a reference into region `'a`. The borrow checker is a **constraint solver**: it collects outlives-constraints (`'a: 'b`, read "`'a` outlives `'b`") from the program text, then checks satisfiability. Tofte–Talpin region inference let you *allocate* into regions; Rust uses the same algebra purely to *prove safety*, with no runtime region at all. The "lifetime" in the type is the static region; nothing is stamped into the binary.

## Lifetimes do not control dropping

Kill this myth now, because it is the commonest wrong mental model. A value is dropped when its **owner** goes out of scope — that is determined by ownership and lexical scope ([Ownership & Moves](02-ownership-and-moves.md)), full stop. Lifetimes are about how long a *reference* may be *used*, and the checker's only job is to prove that every use of a reference happens within the region where its referent is alive. The annotation `'a` does not extend, shorten, or otherwise touch when anything is freed.

Concretely, since the 2018 edition Rust uses **non-lexical lifetimes (NLL)**: a borrow's region ends at its *last use*, not at the closing brace.

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];     // shared borrow begins
println!("{first}");   // ... and ends here, at its last use
v.push(4);             // mutable borrow is fine: the shared borrow's region is over
```

The `&v[0]` still *exists* lexically until the block ends, but its lifetime — the region the checker reasons about — is the slice `[let first ..= println!]`. Drop happens later and is unaffected.

> **🦀 From your toolbox →** In C++, a reference or raw pointer has no statically tracked validity region; you reason about lifetimes in your head and `-fsanitize=address` tells you at runtime when you got it wrong. RAII governs *destruction* (like Rust's drop), but the validity of a `T&` is not part of its type. Rust splits these: ownership ≈ RAII destruction order; lifetimes ≈ a separate, purely static proof that no reference is read after its referent dies. The analogy breaks down precisely where C++ is silent: `&'a T` carries `'a` in the type, so the *signature* of a function is part of the contract a caller must satisfy.

## Why function boundaries need annotations

Inside a single function body the compiler sees every borrow and every drop, so it infers all regions itself — you essentially never write a lifetime inside a body. The trouble starts at **function boundaries**, where the body is opaque to callers and callers are opaque to the body.

Consider returning the longer of two string slices:

```rust
fn pick(a: &str, b: &str) -> &str {
    if a.len() >= b.len() { a } else { b }
}
```

This does not compile:

```text
error[E0106]: missing lifetime specifier
 --> src/lib.rs:1:30
  |
1 | fn pick(a: &str, b: &str) -> &str {
  |            ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the
          signature does not say whether it is borrowed from `a` or `b`
```

The error is exactly right. At compile time the body is summarised by its *signature alone*. The returned reference borrows either `a` or `b` — the compiler can't tell which without running the `if`, and neither can it, in general. So the caller has no way to know how long the result is valid. The fix is to state the relationship in the type:

```rust
fn pick<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
```

Read this as a quantified statement: *for all regions `'a`, given two `&str` both valid for `'a`, I return a `&str` valid for `'a`.* The caller, at each call site, instantiates `'a` with a concrete region — and the compiler picks the **largest** region that satisfies all the input constraints, which is the *intersection* of the two inputs' regions, i.e. the shorter one. The result therefore borrows for "as long as both inputs are alive", which is the tightest sound guarantee.

> **🎓 Tripos link →** From *Logic and Proof* / Curry–Howard: the signature `fn pick<'a>(...) -> &'a str` is a ∀-quantified proposition, `∀'a. (&'a str) → (&'a str) → &'a str`. The function body is its proof term. A call site is ∀-elimination: substitute a concrete region for `'a`. The borrow checker is the proof checker. "Missing lifetime specifier" means the proposition you wrote is too weak to be proven — you must strengthen the type until the body type-checks against it.

Crucially, the annotation does **not** force the caller's two slices to have equal lifetime. Subtyping (below) lets a longer-lived slice be *coerced* to a shorter region so both can unify at `'a`. The signature is a constraint, not an equation.

> **⚙️ Under the hood →** Lifetimes are fully erased after borrow checking. `pick::<'x>` and `pick::<'y>` are the *same* machine code — there is no monomorphisation over lifetimes (unlike type parameters, which *do* get monomorphised; see [Generics & Traits](08-generics-and-traits.md)). A `&'a T` and a `&'b T` have identical layout: one pointer. The entire apparatus exists at the `rustc` MIR level and evaporates before codegen. This is why the chapter's recurring refrain — "no runtime cost" — is literally true.

## Annotations are constraints, demonstrated

The signature is a *contract the compiler enforces at the call site*, not a claim it trusts blindly. Watch it bite:

```rust
fn main() {
    let long = String::from("the quick brown fox");
    let result;
    {
        let short = String::from("dog");
        result = pick(long.as_str(), short.as_str());
    }                         // `short` dropped here
    println!("{result}");     // but `result` may borrow `short`!
}
```

```text
error[E0597]: `short` does not live long enough
  |
  |         result = pick(long.as_str(), short.as_str());
  |                                      ^^^^^ borrowed value does not live long enough
  |     }
  |     - `short` dropped here while still borrowed
  |     println!("{result}");
  |               ------ borrow later used here
```

*You* can see `long` is the longer string, so `result` "really" borrows `long`. The compiler cannot — it reasons from the *signature*, which says the output borrows for the intersection of both inputs' regions. `short`'s region ends at the inner brace, so `result`'s region ends there too, and using it afterwards is rejected. This is the price and the point: the analysis is **local** to the signature, so a bug in `pick` can never silently corrupt a caller, and a change to `pick`'s body that keeps the same signature can never break callers.

## When the relationship is asymmetric

You only relate the lifetimes that are actually related. If `pick` always returned the first argument, the second's region is irrelevant to the output:

```rust
fn first<'a>(a: &'a str, _b: &str) -> &'a str {
    a
}
```

`_b` gets its own (elided, anonymous) lifetime with no link to `'a` or the return. Tying everything to one `'a` when it isn't warranted is a common over-constraint that needlessly rejects valid callers.

And you can never *manufacture* a region with an annotation:

```rust
fn dangling<'a>() -> &'a str {
    let local = String::from("born here, dies here");
    local.as_str()   // borrows a local
}
```

```text
error[E0515]: cannot return value referencing local variable `local`
  |
  |     local.as_str()
  |     -----^^^^^^^^^
  |     returns a value referencing data owned by the current function
```

`'a` is chosen by the *caller*, so it denotes a region in the caller's scope — necessarily larger than the function body. `local` dies at the end of the body. No substitution for `'a` can make this sound, and the annotation has no power to keep `local` alive. The fix is to return owned data (`String`), transferring ownership to the caller (see the owned/borrowed split in [Slices & the Owned/Borrowed Duality](05-slices-and-duality.md)).

## Lifetime elision: deriving the three rules

You wrote `&str -> &str` functions in earlier chapters with no annotations. That is **lifetime elision**: a small, deterministic decision procedure baked into the compiler that fills in the *common* cases. It is sugar, not inference — if the rules don't fully determine the lifetimes, the compiler errors rather than guessing. The rules are worth deriving because knowing exactly when they *stop* is what lets you predict the dreaded `E0106`.

Vocabulary: lifetimes on parameters are **input lifetimes**; lifetimes on the return type are **output lifetimes**. The three rules, applied in order:

1. **Each elided input reference gets its own fresh lifetime.** `fn f(x: &T, y: &U)` becomes `fn f<'a, 'b>(x: &'a T, y: &'b U)`.
2. **If there is exactly one input lifetime, it is assigned to every output lifetime.** `fn f<'a>(x: &'a T) -> &U` becomes `fn f<'a>(x: &'a T) -> &'a U`.
3. **If one input is `&self` or `&mut self`, the lifetime of `self` is assigned to every output lifetime.** (Rule 3 only fires inside methods.)

After applying these, if any output lifetime is still undetermined, elision fails and you must annotate.

Let's run the procedure as the compiler does. `fn head(s: &str) -> &str`:

- Rule 1: `fn head<'a>(s: &'a str) -> &str`.
- Rule 2: one input lifetime, so the output takes it: `fn head<'a>(s: &'a str) -> &'a str`. Done — no annotation needed.

Now `fn pick(a: &str, b: &str) -> &str`:

- Rule 1: `fn pick<'a, 'b>(a: &'a str, b: &'b str) -> &str`.
- Rule 2: more than one input lifetime → does not apply.
- Rule 3: not a method, no `self` → does not apply.

The output lifetime is undetermined, so elision fails — and that is *exactly* the `E0106` we hit earlier. The rules didn't guess; they gave up and demanded an annotation. That is the system working as designed.

> **⚠️ Pitfall →** Rule 2's "exactly one input lifetime" counts lifetime *variables*, not `&` characters. `fn f(x: &(i32, &str)) -> &str` has two input lifetimes (the outer `&` and the inner `&str`), so Rule 2 does *not* fire and you get `E0106`. Conversely `fn g(x: &str, n: usize) -> &str` has exactly one input *lifetime* (`usize` carries none), so Rule 2 *does* fire and `g` needs no annotation. Count regions, not ampersands.

> **🦀 From your toolbox →** Elision is the closest Rust analogue to OCaml's Hindley–Milner type inference you saw in *Foundations* — both let you omit annotations the compiler can recover. The difference is sharp: HM is *principal* and *complete* (it finds the most general type whenever one exists). Elision is deliberately *incomplete* — three syntactic patterns, no backtracking, no unification across the body. Rust chose a dumb-but-predictable procedure so that the *signature* alone is the contract and error messages point at the signature, not at some distant body expression.

## Structs that hold references

Every struct field that is a reference forces a lifetime parameter on the struct, for the same reason functions need one: the type must record how long the borrowed data must outlive the struct.

```rust
struct Parser<'src> {
    input: &'src str,
    pos: usize,
}
```

`Parser<'src>` reads as "a parser borrowing source text for region `'src`". The annotation creates an **outlives obligation**: any `Parser<'src>` instance must not outlive `'src`. The compiler enforces it:

```rust
fn main() {
    let p;
    {
        let text = String::from("1 + 2");
        p = Parser { input: text.as_str(), pos: 0 }; // borrows `text`
    } // `text` dropped — but `p` (region containing `input`) is used below
    println!("{}", p.pos);
}
```

```text
error[E0597]: `text` does not live long enough
```

The struct can't outlive what it borrows. This is the structural rule that makes self-referential and dangling-pointer-holding structs *unrepresentable* by construction. (If you genuinely need a struct that owns data and references it, that requires `Pin`/raw pointers or owning the data and storing indices — covered under [Smart Pointers & Interior Mutability](15-smart-pointers.md). The everyday answer is: store an owned `String`, or store an index, not a borrow.)

Methods on such a struct declare the lifetime after `impl`, because it's part of the type:

```rust
impl<'src> Parser<'src> {
    fn remaining(&self) -> &'src str {       // ties to the *source*, not to &self
        &self.input[self.pos..]
    }

    fn announce(&self, msg: &str) -> &str {  // elided; Rule 3 ties output to &self
        println!("note: {msg}");
        &self.input[..0]
    }
}
```

In `remaining`, we *deliberately* annotate `'src` to return a slice tied to the borrowed source, not to the borrow of `self` — so the returned slice can outlive a temporary `&self`. In `announce`, elision Rule 3 fires: two inputs (`&self`, `msg`), so the output borrows from `self`. Note `announce`'s elided output lifetime is `&self`'s, which is shorter than `'src` — a meaningful difference if the caller drops the `Parser` but wants to keep the slice. Choose annotations to express the *weakest* guarantee that is still true.

## `'static`: two meanings, do not conflate them

`'static` is the longest possible region — the whole program run. It appears in two syntactically similar but semantically distinct positions.

**1. As a reference lifetime, `&'static T`:** the referent lives for the entire program. String literals are the canonical case — their bytes are baked into the binary's read-only data:

```rust
let s: &'static str = "compiled into the binary";
```

**2. As a trait bound, `T: 'static`:** this means "`T` contains no references with a lifetime shorter than `'static`" — i.e. `T` is either fully owned or only borrows `'static` data, so a value of type `T` may be *held indefinitely*. This is the bound you'll see on `thread::spawn` and on `Box<dyn Trait>` ([Fearless Concurrency](13-fearless-concurrency.md), [Trait Objects](09-trait-objects-and-oop.md)).

The trap: `T: 'static` does **not** mean "lives forever". An owned `String` satisfies `T: 'static` — it borrows nothing, so it can be kept as long as you like — even though you drop it after one line:

```rust
fn keep<T: std::fmt::Debug + 'static>(x: T) -> Box<dyn std::fmt::Debug> {
    Box::new(x)
}

let owned = String::from("dropped soon, but still `: 'static`");
let _ = keep(owned);          // OK: String owns its data, no borrows

let s = String::from("hi");
let r: &str = &s;
let _ = keep(r);              // ERROR: r borrows s for less than 'static
```

```text
error[E0597]: `s` does not live long enough
  |     let _ = keep(r);
  |             ------- argument requires that `s` is borrowed for `'static`
```

`String: 'static` (owns everything); `&'a str: 'static` only if `'a` *is* `'static`. The bound restricts *borrowing*, not *duration of existence*.

> **⚠️ Pitfall →** When a borrow-related error suggests "consider adding `'static`", do **not** reflexively add it. It is almost always the wrong fix — it usually means you tried to return or store a reference to something that dies too soon. Slapping `'static` on the signature just moves the rejection to the call site (or fails outright, as above). The real fix is to return owned data, restructure ownership, or shorten the borrow. `'static` is a strong claim; make it only when the data genuinely lives that long.

## Subtyping and variance (the precise version)

You know subtyping from *Further Java* and *Semantics*. Lifetimes induce a subtyping relation, and getting its **variance** right is what separates a working mental model from one that mysteriously rejects code.

The base relation: if region `'long` ⊇ region `'short` (the longer region contains the shorter), then `&'long T` is a **subtype** of `&'short T`. Written `'long: 'short` ("`'long` outlives `'short`"). A reference valid for a *bigger* region can be used wherever one valid for a *smaller* region is expected — you can always forget that you had it for longer. This is why a `&'static str` slots into a parameter of any lifetime: `'static` outlives everything.

```rust
fn use_for<'a>(_: &'a str) {}
let forever: &'static str = "x";
use_for(forever);   // 'static coerces down to 'a — perfectly fine
```

Now the part people get wrong:

- `&'a T` is **covariant** in `'a` *and* covariant in `T`. Longer-lived is usable as shorter-lived, and `&&'long U` is usable as `&&'short U`. Reading deeper doesn't let you violate anything.
- `&'a mut T` is **covariant in `'a`** but **invariant in `T`**. You can shorten the *outer* borrow's region, but you may **not** vary the referent type's lifetime at all — neither up nor down.

Why the asymmetry? Through a `&mut T` you can both *read* and *write*. If `&'a mut T` were covariant in `T`, you could take a `&mut &'long str`, treat it as `&mut &'short str`, and *write* a `'short` reference into a slot the original owner believes holds a `'long` one — then read it back after `'short` has expired. Dangling reference, statically. So mutability forces invariance. The compiler tells you this verbatim:

```rust
fn shorten<'a, 'b: 'a>(r: &'a mut &'b str) -> &'a mut &'a str { r }
```

```text
error: lifetime may not live long enough
  = note: requirement occurs because of a mutable reference to `&str`
  = note: mutable references are invariant over their type parameter
```

> **🎓 Tripos link →** This is exactly the *Further Java* array-covariance hole, but caught statically instead of at runtime. Java made `String[] <: Object[]` (covariant arrays), so writing an `Integer` into a `String[]`-aliased-as-`Object[]` throws `ArrayStoreException` *at runtime*. Same disease — covariance plus mutation is unsound — same cure shape, different time: Rust makes the mutable referent **invariant** so the violating coercion is a *type error*. From *Semantics*: read-only positions may be covariant, write positions must be invariant; read-write must be invariant. Rust applies this rule to lifetimes mechanically.

## Combining lifetimes with generics and bounds

Lifetimes, type parameters, and trait bounds all coexist in one `<...>` list because lifetimes *are* a kind of generic parameter (by convention they come first):

```rust
use std::fmt::Display;

fn pick_and_log<'a, T: Display>(a: &'a str, b: &'a str, tag: T) -> &'a str {
    println!("[{tag}] choosing");
    if a.len() >= b.len() { a } else { b }
}
```

The **outlives bound** `T: 'a` ("every reference inside `T` is valid for at least `'a`") is the tool for relating a type parameter to a region — you need it when a generic struct stores a `&'a` to a `T`, so the compiler can prove the borrow won't outlive the data:

```rust
struct Ref<'a, T: 'a> {     // T: 'a is in fact inferred here, but this is its meaning
    value: &'a T,
}
```

Modern Rust infers `T: 'a` in struct definitions automatically (it's implied by the `&'a T` field), so you rarely write it; you *will* write it explicitly when a bound isn't otherwise derivable, e.g. on trait objects like `Box<dyn Trait + 'a>`.

## Mental-model recap

- A lifetime is a **region** (a set of program points); `'a` is a **generic parameter** ranging over regions, solved by a constraint checker. It is fully erased before codegen — **zero runtime cost, no monomorphisation over lifetimes.**
- Annotations **describe** outlives-relationships between regions; they never change when anything is dropped. Drop is governed by *ownership and scope*; lifetimes only constrain how long a *reference* may be *used*.
- Function/struct boundaries need annotations because the analysis is **local to the signature**: the signature is the entire contract between body and caller. Elision (3 deterministic rules) fills the common cases and *errors rather than guesses* when it can't.
- `'static` has two meanings: `&'static T` (referent lives the whole run) vs `T: 'static` (the type holds no shorter-than-`'static` borrows, so an *owned* value qualifies). "Add `'static`" is usually the wrong fix for a borrow error.
- `&'a T` is covariant; `&'a mut T` is **invariant in the referent type** — the same covariance-plus-mutation unsoundness Java arrays hit at runtime, caught here statically.

## Exercises

1. Write `fn longest_prefix<'a>(a: &'a str, b: &str) -> &'a str` that returns the first `a.len().min(b.len())` bytes of `a`. Then *try* to also relate `b`'s lifetime to the output and observe how over-constraining the signature rejects callers that drop `b` early. Explain which signature is "weakest-but-sound".

2. Run elision by hand on each, predict whether it compiles, then check: (a) `fn f(x: &str) -> &str`; (b) `fn f(x: &str, y: &str) -> &str`; (c) `fn f(x: &(u8, &str)) -> &str`; (d) `fn f(&self, y: &str) -> &str` inside an `impl Foo<'a>`. State which rule fires (or fails) for each.

3. Define `struct Cursor<'a> { data: &'a [i32], at: usize }` with a method `fn next(&mut self) -> Option<&'a i32>`. Make `next` return a reference tied to the underlying `data` (lifetime `'a`), *not* to `&mut self`. Why does eliding the lifetime here produce the *wrong* (too-short) one, and what breaks at the call site if you let elision win?

4. (★) The compiler says `&'a mut T` is invariant in `T`. Construct the smallest function signature you can that the compiler rejects *only* because of this invariance, and write a one-sentence argument for the concrete dangling-pointer bug that would result if the coercion were allowed. (Hint: a `&mut &str` and two distinct lifetimes.)

5. (★) Take the dangling `fn dangling<'a>() -> &'a str` from the chapter. Explain, in terms of *who chooses* `'a`, why **no** body returning a borrow of a local can ever satisfy *any* caller-chosen `'a`. Then write a version that compiles by changing the return type, and say which course concept (ownership transfer) makes the fix sound.

6. Explain why `let v = vec![1,2,3]; let r = &v[0]; drop(v); println!("{r}");` fails but `let r = &v[0]; println!("{r}"); drop(v);` succeeds, in terms of NLL regions rather than lexical scope. What is the *region* of the borrow `r` in each case?
