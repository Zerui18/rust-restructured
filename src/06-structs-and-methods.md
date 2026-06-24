# 6. Structs & Methods

A `struct` is Rust's **product type**: a labelled bundle of its field types, exactly the records you built in OCaml, the `struct`s you wrote in C, the classes-without-methods you'd write in Java or Swift. Nothing here will surprise you syntactically. What is worth your attention is the *seams* — the places where Rust's design diverges from Java and Swift in ways that fall directly out of [ownership](02-ownership-and-moves.md) and [borrowing](03-references-and-borrowing.md). There are three: structs own their fields (so construction and update *move*), behaviour lives in detached `impl` blocks rather than inside the type, and the method receiver `self` is just a borrow whose form (`&`, `&mut`, by-value) *is* the access discipline. Get those three and the rest is notation.

> **🎓 Tripos link →** From *Foundations of Computer Science*: a struct is the dual of an enum. Enums are "one of these" types (one variant active at a time); structs are "all of these together" types (every field present at once). Together with [enums](07-enums-and-pattern-matching.md) they are the OCaml variant-and-record pair you already know — Rust just gives the record half named fields and the variant half [pattern matching](07-enums-and-pattern-matching.md). The count of possible values multiplies: a struct of a `bool` and a `u8` can be in `2 × 256` distinct states.

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

The named-field form is the OCaml record (and the Java/Swift `class`/`struct` with stored properties). The **tuple struct** is the one to dwell on: `Metres` and `Seconds` are *distinct types* despite identical representation, so the compiler rejects `fn fall(t: Seconds)` called with a `Metres`. This is the **newtype** idiom — nominal typing layered onto a structural payload, used constantly to give units, IDs, or wrapped invariants their own type identity (we revisit it under [advanced traits](17-advanced-traits-and-types.md), where it also sidesteps the orphan rule). The **unit struct** has no data; it exists to be a place to hang a trait `impl`, e.g. a marker or a zero-sized strategy object. It is genuinely zero bytes.

> **🦀 From your toolbox →** A tuple struct is Swift's single-case wrapper around a value, or in Java the discipline of wrapping a `long userId` in a `final class UserId { ... }` so the compiler won't let you pass an order ID where a user ID was expected — except in Rust it's one line and costs nothing at runtime. (A light C++ touch: it's the strong-typedef trick people reach for when a bare `double` keeps getting confused with another `double`.) Where the analogy to a C `struct` breaks: a Rust struct has *no defined field layout* by default — `repr(Rust)` lets the compiler reorder and pad fields freely for packing. You cannot reinterpret its raw bytes across an FFI boundary or assume field offsets without `#[repr(C)]` (see [unsafe & FFI](16-unsafe-and-ffi.md)). Unlike a Java field declaration, there is no guarantee the in-memory order matches your source order.

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

> **🔧 In practice →** struct update earns its keep when you're producing a *modified copy* of a config or request object. Say a web handler receives default request options and you want the same options but with a longer timeout for one slow endpoint:
> ```rust
> let slow = RequestOpts { timeout_ms: 30_000, ..base_opts };
> ```
> One line, every other field carried over verbatim. The catch to keep in mind: if `base_opts` holds a non-`Copy` field (a `String` header value, a `Vec` of cookies), it gets *moved* into `slow` — so reach for `..base_opts.clone()` when you still need the original afterwards, or build the modified copy last when you don't.

> **⚠️ Pitfall →** Reusing the base after an update that moved a non-`Copy` field:
> ```rust
> let b = Session { active: false, ..a };
> println!("{}", a.token); // error[E0382]: borrow of moved value: `a.token`
> ```
> The compiler reports `value used here after partial move`, and notes that `..a` moved `a.token` into `b`. The fix is to clone what you need (`token: a.token.clone(), ..a`) or to design so the base is genuinely consumed. This is the same move analysis from [ownership](02-ownership-and-moves.md), now applied field-granularly. Nothing copies the heap buffer behind your back — moving out of `a.token` leaves `a` partially gone, and the compiler refuses to pretend otherwise.

> **⚙️ Under the hood →** `..base` is not a memory trick — it lowers to a field-by-field move/copy into the new value's slots. For `Copy` fields it is a `memcpy` of those bytes; for move-only fields it is an ownership transfer (the source's drop flag for that field is cleared so it is not dropped twice). The whole struct value lives inline on the stack at its declaration site; only heap-backed fields like `String` have their buffer elsewhere.

### Why owned fields, and the lifetime tripwire

Notice `token: String`, not `token: &str`. A struct that stores `&str` is storing a *borrow*, and a borrow is only valid for some region — so the struct itself becomes parametric in that region:

```rust
struct BadSession {
    token: &str,   // error[E0106]: missing lifetime specifier
}
```

The compiler insists you write `struct Session<'a> { token: &'a str }`, tying the struct's validity to the borrowed data. That is correct and sometimes what you want, but it infects every use site with a lifetime parameter. The default, idiomatic choice for an owning data type is owned fields (`String`, `Vec<T>`, `Box<T>`) so the instance owns all its data and outlives nothing external. We treat the `<'a>` machinery properly in [lifetimes](04-lifetimes.md).

> **🎓 Tripos link →** From *Semantics of Programming Languages*, taken purely as intuition: the lifetime parameter `'a` is a label for "the stretch of program during which the borrowed data is guaranteed to be alive." Writing `Session<'a>` says *this struct must not outlive the thing it points at* — if the `String` you borrowed from is dropped, any `Session<'a>` still holding a reference to it would be dangling, so the compiler forbids it. Owned fields are the simple case where the struct points at nothing external, so no such label is needed.

## `impl`: behaviour, detached from data

Here is the deepest break from Java and Swift: **data and behaviour are declared separately.** The struct definition lists fields and nothing else; methods live in an `impl` block.

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

> **🦀 From your toolbox →** Coming from Java or Swift this is jarring: there is no class body, no `this`/`self` field, no `init`/constructor named after the type, no `private`/`public` modifiers bundled into the declaration. The closest mental model is OCaml modules grouping a type with its functions, or "a Java class whose fields and methods got split into two separate files" — except `impl` gives those functions a namespace and `self`-sugar. Crucially, **Rust has no inheritance.** A struct cannot extend another struct; there is no `extends`, no `: BaseClass`, no superclass call. Code reuse and polymorphism go through [traits](08-generics-and-traits.md) (Java interfaces, Swift protocols) and composition, never an `is-a` class hierarchy. If you reach for `extends`, you want a trait or an embedded field.

You may write **multiple `impl` blocks** for one type; they are simply unioned. Pointless for a single file, but essential once blocks carry different `where` clauses or trait bounds (a generic `impl<T: Ord> Heap<T>` alongside an unconstrained `impl<T> Heap<T>`), which you will meet in [generics](08-generics-and-traits.md).

> **⚙️ Under the hood →** `impl` is purely organisational; it emits no runtime structure. A method `r.area()` compiles to an ordinary function call `Rect::area(&r)` — `self` is just the first argument with sugar. There is **no vtable, no per-object function pointer, no hidden field** in `Rect`. The struct is exactly `{ w: u32, h: u32 }`, 8 bytes. Dynamic dispatch only appears when you opt into [trait objects](09-trait-objects-and-oop.md) (`dyn Trait`); plain method calls are static, resolved and inlinable at compile time.

## The receiver `self` *is* the borrow rules

This is the unifying insight of the chapter. The first parameter form encodes precisely the access the method needs, and it is the *same* `&`/`&mut`/owned distinction from [references & borrowing](03-references-and-borrowing.md) applied to the invocant:

| Receiver | Desugars to | Meaning | Caller after the call |
|---|---|---|---|
| `&self` | `self: &Self` | read-only borrow | still owns, can call again |
| `&mut self` | `self: &mut Self` | exclusive mutable borrow | still owns, was mutated |
| `self` | `self: Self` | takes ownership | **moved out**, cannot reuse |

So `area(&self)` borrows shared, `scale(&mut self)` borrows exclusively (and therefore requires the caller's binding be `mut` and not otherwise borrowed), and `into_pair(self)` *consumes* the `Rect`: after `r.into_pair()`, `r` is moved and gone. By-value `self` is rare and deliberate — it is for transformations where the old value should not survive (note the `into_` naming convention signalling consumption). All the borrow-checker rules carry over unchanged: you cannot call `&mut self` while a shared borrow of the instance is live, because that would violate aliasing-XOR-mutation (at most one mutable borrow, or any number of shared ones, never both).

> **🔧 In practice →** the receiver choice is a real API-design decision you make constantly. A method that *reports* something takes `&self` (`config.is_valid()`, `cart.total()`) so callers keep using the value. A method that *updates in place* takes `&mut self` (`cart.add_item(x)`, `buffer.clear()`). And `self` by value is how you build deliberate "this object is now consumed" conversions: a `Request` that you finalise into a `PreparedRequest`, or a file handle's `into_raw_fd()` that hands off ownership so you can't accidentally use the old wrapper afterwards. Picking `self` over `&mut self` is you telling every caller "after this call, the old value is gone" — and the compiler enforces it for free.

> **🎓 Tripos link →** From *Concurrent and Distributed Systems*, as intuition only: `&mut self` is the compile-time analogue of grabbing an exclusive lock on the instance for the duration of the call — but there is no actual lock and no runtime cost, because the type system checks the "no other access while I'm writing" rule *before the program ever runs*. `&self` is the shared-read case (many readers, no writer). This is exactly the discipline that makes data races impossible, which is why [fearless concurrency](13-fearless-concurrency.md) works.

### Autoref / autoderef at the call site

You never write `(&r).area()` or `(&mut r).scale(2)`. Rust's **automatic referencing and dereferencing** inserts the needed `&`, `&mut`, or `*` to make the receiver match the method signature:

```rust
let mut r = Rect::new(3, 4);
r.scale(2);          // auto &mut r
let a = r.area();    // auto &r
```

Given the receiver expression and the method's declared `self` form, there is exactly one coercion that fits, so the compiler applies it silently. This also pierces smart pointers: a method taking `&self` is callable through a `Box<Rect>` because autoderef follows `Deref` (see [smart pointers](15-smart-pointers.md)). It is why Rust needs **no `->` operator**: `obj.method()` works whether `obj` is a value, a `&Rect`, or a `Box<Rect>` — the compiler reconciles it. (Java and Swift never had an `->` to begin with; this is mostly a relief if you've juggled `.` versus `->` in C.)

> **🦀 From your toolbox →** In Java and Swift the dot already works through references transparently, so this feels natural — the difference is that *here* the compiler is silently choosing between a shared borrow, an exclusive borrow, and a move on your behalf, driven by the method's declared receiver. Where it stops: autoref/autoderef applies to **method receivers only**, not to arbitrary function arguments. `area(r)` does not auto-borrow `r` into a `&Rect` parameter; you must pass `&r` explicitly. The sugar is a property of the dot-call, not of `&` generally.

A method may share a name with a field (`fn w(&self) -> u32`); `r.w` reads the field, `r.w()` calls the method. This is the basis of **getters**: keep the field private, expose a `&self` method of the same name for read-only access. Privacy is module-scoped (`pub` opts a field or method out of the default-private rule) and we cover it under [crates & modules](19-project-structure.md). Rust generates no getters or setters for you — there is no Lombok `@Getter`, no Swift computed-property shorthand handed to you; you write them when an invariant or encapsulation demands it, and access fields directly otherwise.

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

> **🔧 In practice →** `#[derive(Debug, Clone, PartialEq)]` is the line you'll type on almost every domain struct. `Debug` so it shows up in log lines and test failure messages (`assert_eq!` prints both sides via `Debug` when it fails — without the derive the test won't even compile). `Clone` so you can hand a copy to another thread or stash one in a cache. `PartialEq` so `assert_eq!(got, expected)` works in your unit tests. A typical real struct opens with:
> ```rust
> #[derive(Debug, Clone, PartialEq, Eq, Hash)]
> struct UserId(u64);   // Hash + Eq so it can be a HashMap key
> ```
> The rule of thumb: derive what your tests and collections demand, and leave `Copy` off anything holding a `String` or `Vec`.

> **⚙️ Under the hood →** `#[derive(PartialEq)]` for `Rect` literally emits `impl PartialEq for Rect { fn eq(&self, o: &Self) -> bool { self.w == o.w && self.h == o.h } }`. The derive recurses structurally and requires each field type to already implement the trait — derive `Clone` and a field that isn't `Clone` fails to compile at the derive site, not at use. The generated code is plain, ordinary code with no special privileges; `cargo expand` shows you exactly what it wrote.

## The builder pattern

Rust has no optional/defaulted/overloaded constructors (no function overloading at all — unlike Java's overloaded constructors or Swift's many `init`s). When a type has many fields, several optional, the idiom is a **builder**: a separate mutable struct that accumulates settings and produces the target with a final `build()`.

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

> **🦀 From your toolbox →** This is the Java/Swift builder you already know — the same fluent `.port(...).tls(...).build()` chain — but with a move-semantics twist that matters: by-value `self` setters mean the half-built builder cannot be aliased or reused mid-chain, so partially-initialised state never leaks (in Java a builder reference can be stashed and called twice; here the compiler stops you). Some libraries instead use `&mut self` setters returning `&mut Self` to avoid the moves — a performance/ergonomics trade you choose per type.

Generic structs (`struct Wrapper<T> { value: T }`) compose all of the above with type parameters and produce a separate stamped-out copy per instantiation; that is the subject of [generics & traits](08-generics-and-traits.md).

## Mental-model recap

- A struct is a **product type that owns its fields**; the instance lives inline, heap-backed fields point out. Three shapes: named-field, tuple (nominal newtype), unit (zero-size marker).
- **Construction and `..base` update move** non-`Copy` fields out of the source — partial moves are tracked per field by the borrow checker, and nothing silently copies the heap data for you.
- Behaviour lives in **`impl` blocks, detached from data; no inheritance, no special constructors.** `new` is a convention; associated functions (`::`, no `self`) vs methods (`.`, take `self`).
- The **receiver form is the borrow discipline applied to the invocant**: `&self` reads, `&mut self` mutates exclusively, `self` consumes. Method calls are static dispatch with autoref/autoderef inserting `&`/`&mut`/`*`.
- `#[derive(...)]` generates trait impls (`Debug`, `Clone`, `PartialEq`, `Copy`, …) mechanically; the **builder pattern** replaces the optional/overloaded constructors Rust omits.

## Exercises

1. You have `struct Celsius(f64)` and `struct Fahrenheit(f64)`. A function `fn boil_check(t: Celsius) -> bool` exists. A colleague calls it as `boil_check(Fahrenheit(212.0))` and is surprised it won't compile. Predict the error, then explain in one sentence why making `Celsius` and `Fahrenheit` tuple structs (rather than both being `f64`) was the *point*, not an inconvenience.

2. Given `let b = Session { active: false, ..a };` where `Session` has fields `user_id: u64`, `token: String`, `active: bool`: decide, for each of `a.user_id`, `a.active`, and `a.token`, whether it is still usable after this line — and state the single property of the field's *type* that decides each answer.

3. Which of these compiles, and why? (a) calling `r.area()` (which takes `&self`) twice in a row; (b) calling `r.scale(2)` (which takes `&mut self`) on a binding declared `let r = ...` (no `mut`); (c) using `r` after `r.into_pair()` (which takes `self`). For each that fails, name what the receiver form is telling the compiler.

4. You're designing a `Transaction` type with a `commit` method. Would you make it `fn commit(self)`, `fn commit(&mut self)`, or `fn commit(&self)` — and what does each choice promise the caller about whether the transaction can be committed twice? Pick one and justify it in terms of who should own the transaction afterwards.

5. (★) A builder uses `&mut self` setters that return `&mut Self`, with `fn build(&self) -> Server`. A teammate switches them to by-value `self` setters returning `Self`, with `fn build(self) -> Server`. For each design, predict whether `let x = b.build(); let y = b.build();` (two builds from the same builder) compiles, and explain what the receiver choice proves about single-use versus reuse — then say which you'd ship for a config builder and why.
