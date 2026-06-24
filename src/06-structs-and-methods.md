# 6. Structs & Methods

A `struct` is Rust's **product type**: a labelled cartesian product of its field types, exactly the records you built in OCaml and the `struct`s you wrote in C. Nothing here will surprise you syntactically. What is worth your attention is the *seams* — the places where Rust's design diverges from C++ and Java in ways that fall directly out of [ownership](02-ownership-and-moves.md) and [borrowing](03-references-and-borrowing.md). There are three: structs own their fields (so construction and update *move*), behaviour lives in detached `impl` blocks rather than inside the type, and the method receiver `self` is just a borrow whose form (`&`, `&mut`, by-value) *is* the access discipline. Get those three and the rest is notation.

> **🎓 Tripos link →** From *Foundations of Computer Science*: a struct is the dual of an enum. Enums are sum types (`τ₁ + τ₂ + …`, one variant active); structs are product types (`τ₁ × τ₂ × …`, all fields present). Together with [enums](07-enums-and-pattern-matching.md) they are the algebraic data types you know — Rust just gives the product half named fields and the sum half [pattern matching](07-enums-and-pattern-matching.md). The cardinality is literally the product: a struct of a `bool` and a `u8` inhabits `2 × 256` values.

## The three shapes of struct

```rust
// 1. Named-field struct — the workhorse.
struct Session {
    user_id: u64,
    token: String,
    active: bool,
}

// 2. Tuple struct — fields positional, type still nominal.
struct Metres(f64);
struct Seconds(f64);

// 3. Unit struct — zero fields, zero size.
struct Imperial;
```

The named-field form is the OCaml record. The **tuple struct** is the one to dwell on: `Metres` and `Seconds` are *distinct types* despite identical representation, so the compiler rejects `fn fall(t: Seconds)` called with a `Metres`. This is the **newtype** idiom — nominal typing layered onto a structural payload, used constantly to give units, IDs, or wrapped invariants their own type identity (we revisit it under [advanced traits](17-advanced-traits-and-types.md), where it also sidesteps the orphan rule). The **unit struct** has no data; it exists to be a place to hang a trait `impl`, e.g. a marker or a zero-sized strategy object. It is genuinely zero bytes.

> **🦀 From your toolbox →** A tuple struct is C++'s `BOOST_STRONG_TYPEDEF` or Swift's single-case wrapper, but built into the language and free. Where the analogy to a C `struct` breaks: a Rust struct has *no defined field layout* by default — `repr(Rust)` lets the compiler reorder and pad fields freely for packing. You cannot `memcpy`-interpret it across an FFI boundary or assume offsets without `#[repr(C)]` (see [unsafe & FFI](16-unsafe-and-ffi.md)). There is also no implicit padding-to-alignment contract you can rely on.

## Construction, shorthand, and update

Construction names the type and supplies every field; order is irrelevant because fields are named.

```rust
fn open(user_id: u64, token: String) -> Session {
    Session {
        user_id,           // field init shorthand: `user_id: user_id`
        token,             // ditto
        active: true,
    }
}
```

**Field init shorthand** elides `field: field` when a local of the same name is in scope — pure ergonomics, no semantics. The interesting construct is **struct update syntax**:

```rust
let a = Session { user_id: 1, token: String::from("abc"), active: true };

let b = Session {
    active: false,
    ..a            // take user_id and token from `a`
};
```

`..a` fills the unspecified fields from `a`. The `..base` must come last. Now the critical fact, and the first real seam: **this is a move, not a copy.** `..a` *moves* `a.user_id` and `a.token` into `b`. Because `token: String` does not implement `Copy`, `a` is partially moved-from and you may no longer use `a` as a whole, nor read `a.token`. You *can* still read `a.user_id` and `a.active` afterwards, because `u64` and `bool` are `Copy` and were copied, not moved. The borrow checker tracks move state per field.

> **⚠️ Pitfall →** Reusing the base after an update that moved a non-`Copy` field:
> ```rust
> let b = Session { active: false, ..a };
> println!("{}", a.token); // error[E0382]: borrow of moved value: `a.token`
> ```
> The compiler reports `value used here after partial move`, and notes that `..a` moved `a.token` into `b`. The fix is to clone what you need (`token: a.token.clone(), ..a`) or to design so the base is genuinely consumed. This is the same move analysis from [ownership](02-ownership-and-moves.md), now applied field-granularly. There is no copy constructor silently saving you, as there would be in C++.

> **⚙️ Under the hood →** `..base` is not a memory trick — it lowers to a field-by-field move/copy into the new value's slots. For `Copy` fields it is a `memcpy` of those bytes; for move-only fields it is an ownership transfer (the source's drop flag for that field is cleared so it is not dropped twice). The whole struct value lives inline on the stack at its declaration site; only heap-backed fields like `String` have their buffer elsewhere.

### Why owned fields, and the lifetime tripwire

Notice `token: String`, not `token: &str`. A struct that stores `&str` is storing a *borrow*, and a borrow is only valid for some region — so the struct itself becomes parametric in that region:

```rust
struct BadSession {
    token: &str,   // error[E0106]: missing lifetime specifier
}
```

The compiler insists you write `struct Session<'a> { token: &'a str }`, tying the struct's validity to the borrowed data. That is correct and sometimes what you want, but it infects every use site with a lifetime parameter. The default, idiomatic choice for an owning data type is owned fields (`String`, `Vec<T>`, `Box<T>`) so the instance owns all its data and outlives nothing external. We treat the `<'a>` machinery properly in [lifetimes](04-lifetimes.md).

> **🎓 Tripos link →** From *Semantics*: the lifetime parameter `'a` is exactly a **region** variable in the region-based memory management you saw formalised. `Session<'a>` is a type abstracted over the region in which its borrowed contents live; the well-formedness condition is that the struct's lifetime is contained in `'a`. Owned fields are the closed case where no region escapes into the type.

## `impl`: behaviour, detached from data

Here is the deepest break from C++/Java: **data and behaviour are declared separately.** The struct definition lists fields and nothing else; methods live in an `impl` block.

```rust
struct Rect {
    w: u32,
    h: u32,
}

impl Rect {
    // Associated function (no `self`) — the constructor idiom.
    fn new(w: u32, h: u32) -> Self {
        Self { w, h }
    }

    // Method — borrows the receiver immutably to read.
    fn area(&self) -> u32 {
        self.w * self.h
    }

    // Method — borrows mutably to write.
    fn scale(&mut self, k: u32) {
        self.w *= k;
        self.h *= k;
    }

    // Method — consumes the receiver.
    fn into_pair(self) -> (u32, u32) {
        (self.w, self.h)
    }
}
```

`Self` inside an `impl` block is an alias for the implementing type (`Rect` here). Within these you distinguish two kinds of item:

- **Associated functions** have no `self` parameter. They are namespaced under the type and called with `::` — `Rect::new(3, 4)`. `String::from` and `Vec::new` are exactly these. There is *no built-in constructor*: `new` is a plain convention, not a keyword, not special, not implicitly called. You may have several (`new`, `square`, `from_pair`) or none.
- **Methods** take `self` first and are called with `.` on an instance — `r.area()`.

> **🦀 From your toolbox →** Coming from Java/C++ this is jarring: there is no class body, no `this`, no constructor with the type's name, no `private:`/`public:` section bundled in. The closest mental model is OCaml modules with the type's functions, or C's "struct + free functions operating on it" — except `impl` gives those functions a namespace and `self`-sugar. Crucially, **Rust has no inheritance.** A struct cannot extend another struct. Code reuse and polymorphism go through [traits](08-generics-and-traits.md) and composition, never an `is-a` class hierarchy. If you reach for `extends`, you want a trait or an embedded field.

You may write **multiple `impl` blocks** for one type; they are simply unioned. Pointless for a single file, but essential once blocks carry different `where` clauses or trait bounds (a generic `impl<T: Ord> Heap<T>` alongside an unconstrained `impl<T> Heap<T>`), which you will meet in [generics](08-generics-and-traits.md).

> **⚙️ Under the hood →** `impl` is purely organisational; it emits no runtime structure. A method `r.area()` monomorphises to an ordinary function call `Rect::area(&r)` — `self` is just the first argument with sugar. There is **no vtable, no per-object function pointer, no hidden field** in `Rect`. The struct is exactly `{ w: u32, h: u32 }`, 8 bytes. Dynamic dispatch only appears when you opt into [trait objects](09-trait-objects-and-oop.md) (`dyn Trait`); plain method calls are static, resolved and inlinable at compile time, just like a templated free function in C++.

## The receiver `self` *is* the borrow rules

This is the unifying insight of the chapter. The first parameter form encodes precisely the access the method needs, and it is the *same* `&`/`&mut`/owned distinction from [references & borrowing](03-references-and-borrowing.md) applied to the invocant:

| Receiver | Desugars to | Meaning | Caller after the call |
|---|---|---|---|
| `&self` | `self: &Self` | read-only borrow | still owns, can call again |
| `&mut self` | `self: &mut Self` | exclusive mutable borrow | still owns, was mutated |
| `self` | `self: Self` | takes ownership | **moved out**, cannot reuse |

So `area(&self)` borrows shared, `scale(&mut self)` borrows exclusively (and therefore requires the caller's binding be `mut` and not otherwise borrowed), and `into_pair(self)` *consumes* the `Rect`: after `r.into_pair()`, `r` is moved and gone. By-value `self` is rare and deliberate — it is for transformations where the old value should not survive (note the `into_` naming convention signalling consumption). All the borrow-checker rules carry over unchanged: you cannot call `&mut self` while a shared borrow of the instance is live, because that would violate aliasing-XOR-mutation.

> **🎓 Tripos link →** From *Concurrent and Distributed Systems*: `&mut self` is the static, compile-time analogue of taking an exclusive lock on the instance for the duration of the call — except enforced by the type system with zero runtime cost, and the proof obligation is discharged before the program runs. `&self` is the shared-read case. The aliasing-XOR-mutation rule is precisely what makes data races *unrepresentable*, which is why [fearless concurrency](13-fearless-concurrency.md) works.

### Autoref / autoderef at the call site

You never write `(&r).area()` or `(&mut r).scale(2)`. Rust's **automatic referencing and dereferencing** inserts the needed `&`, `&mut`, or `*` to make the receiver match the method signature:

```rust
let mut r = Rect::new(3, 4);
r.scale(2);          // auto &mut r
let a = r.area();    // auto &r
```

Given the receiver expression and the method's declared `self` form, there is exactly one coercion that fits, so the compiler applies it silently. This also pierces smart pointers: a method taking `&self` is callable through a `Box<Rect>` because autoderef follows `Deref` (see [smart pointers](15-smart-pointers.md)). It is why Rust needs **no `->` operator**: C/C++ split `.` (value) from `->` (pointer), but `obj.method()` works whether `obj` is a value, a `&Rect`, or a `Box<Rect>` — the compiler reconciles it.

> **🦀 From your toolbox →** This subsumes C++'s `.` vs `->` and the manual `(*p).f()`. Where it stops: autoref/autoderef applies to **method receivers only**, not to arbitrary function arguments. `area(r)` does not auto-borrow `r` into a `&Rect` parameter; you must pass `&r` explicitly. The sugar is a property of the dot-call, not of `&` generally.

A method may share a name with a field (`fn w(&self) -> u32`); `r.w` reads the field, `r.w()` calls the method. This is the basis of **getters**: keep the field private, expose a `&self` method of the same name for read-only access. Privacy is module-scoped (`pub` opts a field or method out of the default-private rule) and we cover it under [crates & modules](19-project-structure.md). Rust generates no getters or setters for you — there is no Lombok, no `@property`; you write them when an invariant or encapsulation demands it, and access fields directly otherwise.

## Derive: trait impls for free

By default a struct can't be printed, compared, or copied — those are trait impls, and you opt in. For the common, mechanically-derivable traits, `#[derive(...)]` writes the `impl` for you (a built-in [procedural macro](18-macros.md)):

```rust
#[derive(Debug, Clone, PartialEq)]
struct Rect { w: u32, h: u32 }

let r = Rect { w: 3, h: 4 };
println!("{r:?}");     // Rect { w: 3, h: 4 }   — requires Debug
println!("{r:#?}");    // pretty-printed, multi-line
dbg!(&r);              // prints file:line + value, returns it
```

`Debug` gives `{:?}`/`{:#?}` formatting (use it on every struct during development; `dbg!` is the quick instrumentation macro). `Clone` gives an explicit deep `.clone()`. `PartialEq`/`Eq` give `==`. `Copy` (only derivable if every field is `Copy`) opts the type into copy-on-move semantics like the primitives — derive it for small, plain-old-data structs and *not* for anything owning a heap allocation. Each derive expands to an `impl` you could have written by hand, field by field; nothing magic, just generated code you can read with `cargo expand`.

> **⚙️ Under the hood →** `#[derive(PartialEq)]` for `Rect` literally emits `impl PartialEq for Rect { fn eq(&self, o: &Self) -> bool { self.w == o.w && self.h == o.h } }`. The derive recurses structurally and requires each field type to already implement the trait — derive `Clone` and a field that isn't `Clone` fails to compile at the derive site, not at use. This is *structural* generation over a *nominal* type, resolved entirely at monomorphisation.

## The builder pattern

Rust has no optional/defaulted/overloaded constructors (no function overloading at all). When a type has many fields, several optional, the idiom is a **builder**: a separate mutable struct that accumulates settings and produces the target with a final `build()`.

```rust
#[derive(Default)]
struct ServerBuilder {
    port: u16,
    workers: usize,
    tls: bool,
}

impl ServerBuilder {
    fn port(mut self, p: u16) -> Self { self.port = p; self }
    fn workers(mut self, n: usize) -> Self { self.workers = n; self }
    fn tls(mut self, on: bool) -> Self { self.tls = on; self }
    fn build(self) -> Server { Server { /* … from self … */ } }
}

let s = ServerBuilder::default().port(8080).tls(true).build();
```

Each setter takes `self` **by value** and returns `Self`, so calls chain and the builder is consumed exactly once at `build()` — the type system enforces the single-construction discipline. `#[derive(Default)]` supplies the zero-initialised starting point (each field's `Default`). This is the canonical answer to "I want named, optional, defaulted arguments," which the language deliberately omits to keep call sites and dispatch simple.

> **🦀 From your toolbox →** This is the Java/C++ builder you already know, but the move-semantics twist matters: by-value `self` setters mean the half-built builder cannot be aliased or reused mid-chain, so partially-initialised state never leaks. Some libraries instead use `&mut self` setters returning `&mut Self` to avoid the moves — a performance/ergonomics trade you choose per type.

Generic structs (`struct Wrapper<T> { value: T }`) compose all of the above with type parameters and monomorphise per instantiation; that is the subject of [generics & traits](08-generics-and-traits.md).

## Mental-model recap

- A struct is a **product type that owns its fields**; the instance lives inline, heap-backed fields point out. Three shapes: named-field, tuple (nominal newtype), unit (zero-size marker).
- **Construction and `..base` update move** non-`Copy` fields out of the source — partial moves are tracked per field by the borrow checker, and there is no copy constructor to hide it.
- Behaviour lives in **`impl` blocks, detached from data; no inheritance, no special constructors.** `new` is a convention; associated functions (`::`, no `self`) vs methods (`.`, take `self`).
- The **receiver form is the borrow discipline applied to the invocant**: `&self` reads, `&mut self` mutates exclusively, `self` consumes. Method calls are static dispatch with autoref/autoderef inserting `&`/`&mut`/`*`.
- `#[derive(...)]` generates trait impls (`Debug`, `Clone`, `PartialEq`, `Copy`, …) mechanically; the **builder pattern** replaces the optional/overloaded constructors Rust omits.

## Exercises

1. Define `struct Point(f64, f64)` and `struct Vector(f64, f64)` as distinct tuple structs. Write a free function expecting a `Vector` and try to pass a `Point`. Read the error; explain why structural identity is not type identity here.

2. Build a `Session { id: u64, token: String }`, create a second instance with `Session { id: 2, ..first }`, then attempt `println!("{}", first.token)`. Predict the `E0382` message *before* compiling, then explain why `first.id` would still be readable but `first.token` is not.

3. Write `area(&self)` and `scale(&mut self, k: u32)` on a `Rect`. In `main`, take a shared borrow `let b = &r;`, then call `r.scale(2)` while `b` is still used afterwards. Identify which borrow rule the compiler cites and why autoref makes the conflict invisible at the call syntax.

4. (★) Implement `fn merge(self, other: Self) -> Self` on a struct that owns a `String`. Call `a.merge(b)`, then try to use `a` and `b` again. Now change the receiver to `&self` and `other: &Self` and observe what you must add to the body. Explain the trade-off between the consuming and borrowing designs in terms of who owns the inputs afterwards.

5. (★) Write a builder whose setters take `&mut self` and return `&mut Self`, and a `build(&self) -> T`. Try to call `build()` twice on the same builder, then try the same with by-value `self` setters. Explain how the receiver choice changes whether the builder is single-use, and what the borrow checker proves in each case.

6. Add `#[derive(Copy, Clone)]` to a struct containing a `String` field. Read the resulting error and explain, in terms of move semantics, why `Copy` cannot be derived when an owning field is present.
