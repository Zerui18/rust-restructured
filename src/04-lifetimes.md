# 4. Lifetimes as Regions

In [References & Borrowing](03-references-and-borrowing.md) you saw the borrow checker reject a reference that outlived its referent. That chapter waved its hands at *how* the checker knows. The machinery is **lifetimes**, and the single most important sentence in this chapter is this:

**A lifetime is a region — a set of program points over which a reference is valid — and a lifetime annotation `'a` is a generic parameter ranging over such regions. Annotations *describe* constraints between regions; they never *change* them.**

If you internalise that, the rest is bookkeeping. The thing that trips up newcomers is treating `'a` as a runtime quantity ("how long does this thing live?") rather than what it is: a *type-level* variable that the compiler solves for, exactly like a type parameter `T` (think of a Java or Swift generic `<T>`, but standing for a *span of code* instead of a type). You have met the underlying idea before — it just had a different name in a different course.

> **🎓 Tripos link →** From *Semantics of Programming Languages*, the intuition is: a lifetime `'a` names a *region* of the program, and `&'a T` is "a reference that's only valid inside region `'a`". The borrow checker works like a little constraint solver: it reads relationships off the program text — "this region must contain that one" — and checks they can all hold at once. There's no runtime machinery for any of this; the region lives entirely in the type and disappears before the binary is produced.

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

> **🦀 From your toolbox →** Think about how the languages you know decide *when memory is reclaimed* and *whether a reference is still valid*. Java's GC and Python's reference counting both reclaim objects at some indeterminate later point, and a reference is "valid" as long as something still points at the object — you never reason about validity statically. Swift's ARC is the same shape: the object lives while a strong reference exists. None of these track *where in the code* a reference may legally be used. Rust splits the two concerns cleanly: *ownership and scope* decide deterministically when a value is dropped (at a known point, no GC pause, no refcount), and *lifetimes* are a separate, purely compile-time proof that no reference is read after its referent dies. The analogy breaks down where the managed languages are silent: in Rust, `&'a T` carries `'a` *in the type*, so a function's *signature* is part of the contract every caller must satisfy — there is no runtime safety net catching you later.

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

Read this as a promise that holds *for any* region `'a`: *given two `&str` both valid for `'a`, I return a `&str` valid for `'a`.* The caller, at each call site, fills in `'a` with a concrete region — and the compiler picks the **largest** region that satisfies all the input constraints, which is the *overlap* of the two inputs' regions, i.e. the shorter one. The result therefore borrows for "as long as both inputs are alive", which is the tightest sound guarantee.

> **🔧 In practice →** This signature is exactly what you write when a function hands back a *view into* its inputs instead of allocating. Say you're tokenising and want the longest of two candidate matches without copying any bytes:
> ```rust
> fn longer_match<'a>(by_keyword: &'a str, by_symbol: &'a str) -> &'a str {
>     if by_keyword.len() >= by_symbol.len() { by_keyword } else { by_symbol }
> }
> ```
> The caller gets back a slice that still points into one of the original buffers — zero allocation, zero copy — and the `'a` is what lets the compiler guarantee the buffers outlive the slice. You reach for borrowed-output signatures like this all over parsers, config readers, and anything that slices a `&str`/`&[u8]` and returns part of it.

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

*You* can see `long` is the longer string, so `result` "really" borrows `long`. The compiler cannot — it reasons from the *signature*, which says the output borrows for the overlap of both inputs' regions. `short`'s region ends at the inner brace, so `result`'s region ends there too, and using it afterwards is rejected. This is the price and the point: the analysis is **local** to the signature, so a bug in `pick` can never silently corrupt a caller, and a change to `pick`'s body that keeps the same signature can never break callers.

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

> **🦀 From your toolbox →** Elision is the closest Rust analogue to the type inference you saw in *Foundations of Computer Science*, where OCaml lets you write `let f x = x + 1` and figures out the types for you — you omit annotations the compiler can recover. But the resemblance is shallow and worth pinning down. OCaml's inference is *thorough*: it explores the whole function body and finds the most general type whenever one exists. Elision is deliberately *not* thorough — it's just three syntactic patterns checked in order, with no looking inside the body and no backtracking. Rust chose a dumb-but-predictable procedure on purpose, so that the *signature* alone is the contract and error messages point at the signature rather than at some distant expression buried in the body.

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

> **🔧 In practice →** A borrowing struct is the right call whenever you build a short-lived "view" object over data someone else owns and you don't want to copy it. The classic case is a parser or scanner that walks over an input buffer:
> ```rust
> let source = std::fs::read_to_string("config.toml")?;
> let mut parser = Parser { input: &source, pos: 0 };
> // parser borrows `source`; as long as `parser` is alive, `source` must be too.
> ```
> Because `Parser<'src>` ties itself to `source`, the compiler will *stop you at compile time* from, say, returning the parser out of a function while letting `source` drop — exactly the use-after-free you'd otherwise debug at runtime in C. If that borrow obligation ever becomes inconvenient (you want the parser to outlive the input), that's the compiler telling you to store an owned `String` field instead.

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

> **🔧 In practice →** You'll meet `T: 'static` the first time you spawn a thread. `thread::spawn` requires its closure to be `'static` because the new thread might outlive the function that launched it — so the closure must not capture any reference that could go dangling:
> ```rust
> let data = vec![1, 2, 3];
> std::thread::spawn(move || {        // `move` makes the closure OWN `data`
>     println!("{:?}", data);         // now the closure is `'static`: it borrows nothing
> });
> ```
> Without `move` the closure would borrow `data`, the borrow would be shorter than `'static`, and you'd get the same "does not live long enough" error. The fix is almost never to add `'static` to a borrow — it's to *own* the data (here, `move` transfers `data` into the closure). That mirrors the pitfall below: `'static` is satisfied by owning, not by wishing.

> **⚠️ Pitfall →** When a borrow-related error suggests "consider adding `'static`", do **not** reflexively add it. It is almost always the wrong fix — it usually means you tried to return or store a reference to something that dies too soon. Slapping `'static` on the signature just moves the rejection to the call site (or fails outright, as above). The real fix is to return owned data, restructure ownership, or shorten the borrow. `'static` is a strong claim; make it only when the data genuinely lives that long.

## Subtyping and variance (the precise version)

You know subtyping from Java and Swift: a `String` can be used where an `Object` (Java) or a less-specific type is expected, because it's "at least as good". Lifetimes induce a similar subtyping relation of their own, and getting its **variance** right — *which direction* the substitution is allowed to go — is what separates a working mental model from one that mysteriously rejects code.

The base relation: if region `'long` ⊇ region `'short` (the longer region contains the shorter), then `&'long T` can be used where `&'short T` is expected. Written `'long: 'short` ("`'long` outlives `'short`"). A reference valid for a *bigger* region can be used wherever one valid for a *smaller* region is expected — you can always forget that you had it for longer. This is why a `&'static str` slots into a parameter of any lifetime: `'static` outlives everything.

```rust
fn use_for<'a>(_: &'a str) {}
let forever: &'static str = "x";
use_for(forever);   // 'static coerces down to 'a — perfectly fine
```

Now the part people get wrong:

- `&'a T` (a *shared, read-only* reference): a longer-lived one is freely usable where a shorter-lived one is wanted, and likewise `&&'long U` is usable as `&&'short U`. Reading deeper through layers of `&` never lets you violate anything.
- `&'a mut T` (a *mutable* reference): you can still shorten the *outer* borrow's region, but you may **not** vary the referent type's lifetime at all — neither lengthen nor shorten it.

Why the asymmetry? Through a `&mut T` you can both *read* and *write*. If a `&'a mut T` allowed you to swap a longer lifetime in `T` for a shorter one, you could take a `&mut &'long str`, treat it as `&mut &'short str`, and *write* a `'short` reference into a slot the original owner believes holds a `'long` one — then read it back after `'short` has expired. Dangling reference, statically. So mutability forbids that substitution. The compiler tells you this verbatim:

```rust
fn shorten<'a, 'b: 'a>(r: &'a mut &'b str) -> &'a mut &'a str { r }
```

```text
error: lifetime may not live long enough
  = note: requirement occurs because of a mutable reference to `&str`
  = note: mutable references are invariant over their type parameter
```

> **🎓 Tripos link →** This is the same hole that *Further Java* warns about with **array covariance**, but caught at compile time instead of at runtime. Java lets you treat a `String[]` as an `Object[]` (covariant arrays), so if you then write an `Integer` into it through the `Object[]` alias, you get an `ArrayStoreException` *while the program runs*. Same disease — letting you substitute freely *and* write through the result is unsound — and essentially the same cure, just applied earlier: Rust refuses the lifetime substitution on the *mutable* reference, so the violating coercion is a *compile error*, not a runtime crash. The rule of thumb is general: a position you only ever read from can be substituted freely; a position you can also write to cannot.

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
- `&'a T` (shared) substitutes freely from longer to shorter; `&'a mut T` (mutable) may **not** vary the referent type's lifetime at all — the same covariance-plus-mutation unsoundness Java arrays hit at runtime, caught here statically.

## Exercises

1. You have `fn pick<'a>(a: &'a str, b: &'a str) -> &'a str` from the chapter. A colleague proposes a "simpler" variant `fn pick2<'a, 'b>(a: &'a str, b: &'b str) -> &'a str` that always returns `a`. Which calls compile under `pick` but are *rejected* under a version that also tried to return `b` through `'a`? Sketch one call site where giving `b` its own independent `'b` matters, and state in one sentence the principle ("annotate the weakest true relationship").

2. Run elision by hand on each, predict whether it compiles *before* checking, and name which rule fires or fails: (a) `fn f(x: &str) -> &str`; (b) `fn f(x: &str, y: &str) -> &str`; (c) `fn f(x: &(u8, &str)) -> &str`; (d) `fn f(&self, y: &str) -> &str` inside an `impl Foo<'a>`. For the one that fails, write the annotation that would make it compile *and* express the weakest sound guarantee.

3. Adapt the `Parser<'src>` from the chapter: add a method `fn rest_after(&self, n: usize) -> &'src str` returning the source from byte `n` onward. Now *predict*: if you instead wrote the return type as plain `-> &str` and let elision choose, what lifetime would the output get, and give a concrete two-line caller that compiles with the `'src` version but is rejected by the elided one. (You don't have to run it — reason it out, then verify.)

4. Design decision: you're writing a function that returns "the first word of the input". You could return `&str` (borrowing the input) or `String` (owned). Give one calling scenario where the borrowing version is clearly better and one where it forces an awkward `'static` or a clone at the call site. Which would you make the default, and why?

5. (★) The chapter shows `&'a mut T` won't let you vary the referent's lifetime. Construct the *smallest* function signature you can that the compiler rejects **only** for this reason (hint: a `&mut &str` and two lifetimes where one outlives the other), then write one sentence describing the concrete dangling-pointer bug that would occur if Rust accepted it. Bonus: change one `&mut` to `&` and explain why it now compiles.

6. Explain, purely in terms of NLL *regions* (not lexical braces), why `let v = vec![1,2,3]; let r = &v[0]; drop(v); println!("{r}");` fails but reordering to `let r = &v[0]; println!("{r}"); drop(v);` succeeds. State the exact region of the borrow `r` in each case, then predict what happens if you add a *second* `println!("{r}")` after the `drop(v)` in the second version.
