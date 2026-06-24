# 8. Generics & Traits

Two abstraction mechanisms collapse into one design here. **Generics** give you parametric polymorphism over types; **traits** give you *bounded* polymorphism — a way to say "for all types `T` *that support this interface*". You have met both ideas before under three different names: ML/OCaml's `'a` type variables, Haskell's type classes, and Java's interfaces with generics. Rust's contribution is to make the whole thing *statically resolved and zero-cost by default*, with a coherence discipline that ML and Java both lack. That last point is the one worth fixing in your head: every abstraction in this chapter is compiled away. There is no boxing, no vtable, no reflection, no erasure — unless you explicitly ask for it (which is [chapter 9](09-trait-objects-and-oop.md)).

## Generics are monomorphised, not erased

A generic function is a *template* over types. The syntax echoes C++ and Java: type parameters live in angle brackets, declared between the name and the value parameters.

```rust
fn largest<T: PartialOrd>(xs: &[T]) -> &T {
    let mut best = &xs[0];
    for x in xs {
        if x > best {
            best = x;
        }
    }
    best
}
```

Read `fn largest<T>` as "largest is generic over some type `T`". The `T: PartialOrd` is a *bound* — ignore it for one paragraph and we will return to why it is mandatory.

The decisive question is what the compiler *does* with this. Rust **monomorphises**: at every call site, rustc reads off the concrete type argument and stamps out a specialised copy of the function with `T` substituted. `largest(&[1i32, 2, 3])` and `largest(&['a', 'z'])` compile to two entirely separate machine-code functions, `largest::<i32>` and `largest::<char>`, each with the comparison inlined to the right instruction. After monomorphisation there is no `T` left anywhere in the binary.

> **🦀 From your toolbox →** This is *exactly* C++ template instantiation, and the tradeoff is identical: zero abstraction penalty at runtime, paid for in compile time and code size (the "code bloat" you know from heavily templated C++). It is the **opposite** of Java generics, which use **type erasure** — `List<String>` and `List<Integer>` are one class `List` at runtime, with casts inserted and `T` reduced to `Object`. Java pays a uniform boxing/dispatch cost and loses the type at runtime (hence no `new T[]`); Rust keeps full type information and pays nothing at runtime. The analogy to C++ templates breaks down in one crucial place, covered below: C++ checks a template's body *only when instantiated* (duck typing, late errors, infamous error messages), whereas Rust type-checks the generic body *once, against its bounds*, before any instantiation exists.

> **⚙️ Under the hood →** Monomorphisation happens after type-checking, during MIR-to-LLVM lowering. rustc collects the set of "mono items" reachable from `main` (or from `pub` API for libraries) and emits one LLVM function per distinct `(fn, type-args)` pair. Identical instantiations across crates are deduplicated at link time. This is also why generic functions usually must live in a crate's source (or be marked `#[inline]`) to be usable downstream — there is nothing to call until a concrete type pins the template down.

> **🎓 Tripos link →** *Compiler Construction.* You studied two implementation strategies for polymorphism: the **boxed/uniform representation** (everything is a pointer, one code copy, runtime dispatch — ML and Java) versus **specialisation/monomorphisation** (one code copy per type, static dispatch). Rust commits to specialisation for generics, which is why a generic `Vec<T>` can store `T` *unboxed and inline* in a flat buffer, exactly as a C++ `std::vector<T>` does — impossible under erasure.

Generics are not limited to functions. Structs, enums, and `impl` blocks all take type parameters:

```rust
struct Pair<T> {
    a: T,
    b: T,
}

enum Tree<T> {
    Leaf(T),
    Node(Box<Tree<T>>, Box<Tree<T>>),
}

impl<T> Pair<T> {
    fn new(a: T, b: T) -> Self {
        Self { a, b }
    }
}
```

Note the `impl<T> Pair<T>` repetition. You declare the parameter *twice*: `impl<T>` introduces `T` into scope, and `Pair<T>` says which type the block is for. This is not redundant — it lets you write `impl Pair<i32>` to add methods *only* to the `i32` specialisation, or constrain it as `impl<T: Display> Pair<T>` to add methods only when `T` is printable. The `Option<T>` and `Result<T, E>` you have been using since [enums](07-enums-and-pattern-matching.md) are nothing but generic enums from the standard library; their definitions are three lines each.

## Why the bound is mandatory: the C++ contrast

Drop the `PartialOrd` bound and the generic `largest` does *not* compile:

```text
error[E0369]: binary operation `>` cannot be applied to type `&T`
 |
 |     for x in xs { if x > best { ... } }
 |                      - ^ - &T
help: consider restricting type parameter `T` with trait `PartialOrd`
 | fn largest<T: std::cmp::PartialOrd>(xs: &[T]) -> &T {
 |             ++++++++++++++++++++++
```

This error is the philosophical centre of the chapter. In a C++ template, `x > best` would be accepted at *definition* time and only rejected later, per-instantiation, if some `T` happened to lack `operator>` — producing a wall of errors pointing into library internals. Rust refuses to type-check a generic body against capabilities the type parameter has not been *proven* to have. Inside `largest`, the compiler knows *nothing* about `T` except what its bounds promise. `T: PartialOrd` is a promise that `>` works; without it, `>` is simply not in scope for `T`.

> **🎓 Tripos link →** *Logic and Proof.* A bounded generic is **bounded quantification**: `largest` has the logical type `∀T. (T: PartialOrd) ⇒ &[T] → &T`. The bound is a *constraint on the universally quantified variable*. The compiler discharges the proof obligation at each call site (it checks `i32: PartialOrd`), and inside the body it may *assume* the constraint holds. This is precisely the difference between unrestricted `∀T. P(T)` and guarded `∀T. C(T) ⇒ P(T)` — the body gets `C(T)` as a hypothesis. C++ templates correspond to the unguarded, untyped version that is only checked after substitution.

## Traits: type classes, interfaces, and shared behaviour

A **trait** is a named set of method signatures — a contract. Types *implement* a trait to declare they satisfy that contract.

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct Tweet {
    pub user: String,
    pub body: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.user, self.body)
    }
}
```

The shape `impl Trait for Type { ... }` is the whole game. It is deliberately *separate* from the type's own definition — unlike a Java `class Tweet implements Summary`, where the interface list is welded to the class declaration. In Rust the type and the trait implementation are independent items, and that separation is what makes traits more powerful than interfaces.

> **🦀 From your toolbox →** Three simultaneous analogies, all partly true:
> - **Java/Swift interfaces/protocols.** A trait is a list of required methods a type must provide. But Java bolts the implements-list onto the class; Rust lets you implement a trait for a type you didn't define (subject to the orphan rule below). This is "interfaces done right": you can make `i32` implement *your* trait.
> - **Haskell type classes.** This is the closest fit. `trait` ≈ `class`, `impl Trait for T` ≈ `instance Class T`, trait bound `T: Trait` ≈ context `Class t =>`. Default methods ≈ default class methods. Associated types ≈ associated type families. If you know type classes, you already know traits; the syntax is the only delta.
> - **OCaml.** No direct equivalent — OCaml uses functors and explicit module passing for the same job. A trait bound is like an implicit, compiler-inferred functor argument.
> Where every analogy breaks: a trait can be used for **static** dispatch (monomorphised, zero-cost) *or* **dynamic** dispatch (`dyn Trait`, a vtable). Java interfaces are always dynamic; this chapter is all static, and [chapter 9](09-trait-objects-and-oop.md) is the dynamic half.

### Default methods

A trait may supply method bodies, not just signatures. Implementors inherit the default and may override it.

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    // Default, expressed in terms of the required method.
    fn summarize(&self) -> String {
        format!("(read more from {}…)", self.summarize_author())
    }
}

impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.user)
    }
    // summarize is inherited for free.
}
```

This is the classic "template method" pattern, but enforced by the type system: the trait author writes the *policy* (`summarize`) in terms of a small required *primitive* (`summarize_author`). An implementor writes one method and gets the other. Note that a default body can call required methods that have *no* default — exactly like Haskell's `class` defaults, or an abstract base class's concrete method calling an abstract one. You cannot, however, call the overridden default from within an override; there is no `super`.

## Trait bounds: spelling and the impl Trait pair

Once a trait exists, you constrain generics with it. There are several equivalent spellings; choosing among them is a style decision with one semantic gotcha.

```rust
// 1. Bound directly on the type parameter.
fn notify<T: Summary>(item: &T) { /* ... */ }

// 2. `impl Trait` in argument position — sugar for (1) with an anonymous T.
fn notify(item: &impl Summary) { /* ... */ }

// 3. Multiple bounds with `+`.
fn notify<T: Summary + Display>(item: &T) { /* ... */ }

// 4. `where` clause — identical meaning, readable when bounds pile up.
fn process<T, U>(t: &T, u: &U) -> i32
where
    T: Summary + Clone,
    U: Display + Debug,
{
    todo!()
}
```

Forms 1 and 2 are *almost* interchangeable, and the difference is worth internalising. `impl Summary` in argument position introduces a *fresh anonymous* type parameter. So:

```rust
fn pair(a: &impl Summary, b: &impl Summary);   // a and b may be DIFFERENT types
fn pair<T: Summary>(a: &T, b: &T);             // a and b must be the SAME type
```

If you need two arguments to share one type (to compare them, swap them, store them in one `Vec`), you *must* name the parameter `<T>`; `impl Trait` cannot express that constraint because each `impl Summary` is its own type. Reach for `where` once you have more than one or two bounds — it pulls the noise out from between the function name and its parameters.

### impl Trait in return position is a different beast

The same `impl Trait` syntax in *return* position means something subtly but importantly distinct from its argument-position use.

```rust
fn make_summarizer() -> impl Summary {
    Tweet { user: "ferris".into(), body: "🦀".into() }
}
```

In argument position, the *caller* chooses the concrete type and the function is generic. In return position, the *callee* (this function) picks **one** fixed concrete type, and merely *hides* its name from the caller. The caller knows only "it implements `Summary`". This is an *opaque* return type — the compiler still monomorphises to the real type `Tweet`, but the name does not leak into the signature.

The single-type restriction bites immediately:

```rust
fn pick(b: bool) -> impl Summary {
    if b { Tweet { .. } } else { Article { .. } }   // does NOT compile
}
```

```text
error[E0308]: `if` and `else` have incompatible types
 |     if b { Tweet { .. } } else { Article { .. } }
 |            -------------          ^^^^^^^^^^^^^^^ expected `Tweet`, found `Article`
help: you could change the return type to be a boxed trait object
 |   fn pick(b: bool) -> Box<dyn Summary> {
```

> **⚠️ Pitfall →** "But both implement `Summary`!" — yes, but `impl Trait` return is a *single hidden type*, not a union. `Tweet` and `Article` are different types and the two branches disagree. The compiler's own fix is the right one: erase to `Box<dyn Summary>` (a trait object, dynamic dispatch — [chapter 9](09-trait-objects-and-oop.md)). The rule of thumb: `impl Trait` return when there is exactly one concrete type but you'd rather not (or can't) write its name; `Box<dyn Trait>` when the type genuinely varies at runtime.

The reason `impl Trait` exists at all is that some types are *unnameable* — every closure has a unique anonymous compiler-generated type, and iterator adaptor chains produce types like `Map<Filter<Zip<...>>>` that are absurd to write by hand. `impl Iterator<Item = u32>` lets you return them concisely while preserving static dispatch. You will lean on this constantly in [closures and iterators](12-closures-and-iterators.md).

## Associated types vs generic type parameters

A trait can carry *associated types* — type members chosen by the implementor:

```rust
trait Container {
    type Item;                                  // associated type
    fn get(&self, i: usize) -> Option<&Self::Item>;
}

impl<T> Container for Vec<T> {
    type Item = T;                              // fixed by this impl
    fn get(&self, i: usize) -> Option<&T> { self.as_slice().get(i) }
}
```

The canonical example is `Iterator`:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

You could *imagine* writing this with a generic parameter instead: `trait Iterator<Item>`. Why does the standard library use an associated type? Because of *functional dependency*: a given iterator type yields exactly **one** element type. With `trait Iterator<Item>`, a single type could legally implement `Iterator<u8>` *and* `Iterator<String>`, forcing you to annotate `Item` at every call (`it.next::<u8>()`-style) and breaking inference. With an associated type, `Item` is **an output determined by the implementing type, not an input chosen by the caller** — so `it.next()` infers cleanly.

> **🦀 From your toolbox →** Decide by counting implementations. **Generic parameter** = the trait can be implemented *many times* for one type with different parameters (e.g. `Add<i32>` *and* `Add<Duration>` for one type — see below). **Associated type** = at most *one* implementation makes sense per type, and the type is a *consequence* of the impl, not a choice (the element type of an iterator, the output of indexing). This is Haskell's distinction between a multi-parameter type class and an associated `type` family, verbatim.

> **🎓 Tripos link →** *Foundations of CS / type inference.* An associated type acts as a **type-level function**: `<Vec<T> as Iterator>::Item` *computes* to `T`. Because it is a function (one output per input), the inference engine can run it forwards and the Hindley–Milner-style unifier stays decidable. A generic parameter is an *input* the unifier must *solve for*, which is why multiple impls would make `Item` ambiguous.

## Operator overloading is just trait implementation

Rust has no special operator-overloading syntax. `+` *is* a call to `std::ops::Add::add`; `[]` is `Index::index`; `*` (deref) is `Deref::deref`. Overload an operator by implementing the corresponding trait — and notice it uses an associated type for its `Output`:

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct V2 { x: f64, y: f64 }

impl Add for V2 {
    type Output = V2;
    fn add(self, rhs: V2) -> V2 {
        V2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}
// V2 { x: 4.0, y: 6.0 } from V2{1,2} + V2{3,4}
```

`Add` is *generic over the right-hand side* with a defaulted parameter — its real signature is `trait Add<Rhs = Self>`. That default is why `impl Add for V2` means `impl Add<V2> for V2`. Because `Rhs` is a *generic parameter* (not associated), you may implement `Add` several times for one type with different operands — `Meters + Meters` and `Meters + Millimeters` are two separate impls. This is the textbook case where a generic parameter is correct and an associated type would be wrong: there are genuinely many implementations.

> **⚙️ Under the hood →** `a + b` desugars to `Add::add(a, b)`, monomorphised and inlined to the same instructions you'd write by hand — no indirection. For primitives, the `impl Add for i32` exists in core and the optimiser folds the call to a single machine `add`. Rust deliberately does *not* let you invent *new* operator glyphs (no C++-style `operator""` zoo beyond the fixed trait set), keeping parsing and reasoning local.

## Coherence and the orphan rule

Here is the discipline that ML and Java lack. Rust guarantees **global uniqueness**: for any (trait, type) pair, there is *at most one* implementation in the entire program. This property is called **coherence**, and it is what lets `a + b` or `set.insert(x)` resolve deterministically without you ever specifying *which* `Add` or `Hash` impl to use. There is only ever one.

To preserve coherence across separately-compiled crates, Rust enforces the **orphan rule**: in `impl Trait for Type`, *at least one* of `Trait` or `Type` must be defined in your crate. You may implement *your* trait for `Vec<T>`, and you may implement `Display` (foreign) for *your* `Tweet` — but you cannot implement a foreign trait for a foreign type:

```rust
impl std::fmt::Display for Vec<i32> { /* ... */ }   // forbidden
```

```text
error[E0117]: only traits defined in the current crate can be
              implemented for types defined outside of the crate
 | impl std::fmt::Display for Vec<i32> {
 | ^^^^^^^^^^^^^^^^^^^^^^^^^^^-------- `Vec` is not defined in the current crate
 = note: define and implement a trait or new type instead
```

> **⚠️ Pitfall →** This bites everyone eventually: "I want to add a method to `Vec`" or "I want `serde::Serialize` for `chrono::DateTime`" and both are foreign. The idiomatic fix is the **newtype pattern**: wrap the foreign type in a local one-field tuple struct, `struct Wrapper(Vec<i32>)`, and implement the trait on `Wrapper`. The wrapper is local, so the orphan rule is satisfied, and it compiles to nothing (single-field structs are layout-transparent). The `#[derive(...)]` or a `Deref` impl restores ergonomic access to the inner value.

> **🎓 Tripos link →** *Further Java / type systems.* Haskell hit this exact problem — "orphan instances" can silently change program behaviour depending on import order, breaking modular reasoning. Rust elevates the informal Haskell convention into a *hard compiler rule*. The payoff is a soundness guarantee you can state crisply: trait resolution is a *function* from (trait, type) to impl, total and deterministic, decided entirely at compile time. Java sidesteps the issue by tying implementation to class definition (you simply *can't* retro-fit), which is strictly less expressive.

## Conditional and blanket impls

Trait bounds on `impl` blocks let you add methods *conditionally*, depending on what the inner type can do:

```rust
use std::fmt::Display;

struct Pair<T> { x: T, y: T }

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self { Self { x, y } }   // always available
}

impl<T: Display + PartialOrd> Pair<T> {
    fn show_larger(&self) {                         // only when T: Display + PartialOrd
        if self.x >= self.y { println!("larger: {}", self.x) }
        else { println!("larger: {}", self.y) }
    }
}
```

`Pair<File>` gets `new` but not `show_larger`, because `File` is neither `Display` nor `PartialOrd`. The method *exists only for the type arguments that earn it* — a precision Java generics cannot express.

The most powerful form is a **blanket impl**: implementing a trait for *every* type satisfying some bound. The standard library does this pervasively. `ToString` is implemented once, for all `Display` types:

```rust
impl<T: Display> ToString for T {
    // ...uses T's Display to build a String
}
```

That single impl is why `3.to_string()` and `vec2.to_string()` and any `Display` type's `.to_string()` all work — none of them implements `ToString` directly. Blanket impls compose abstraction: state a capability once in terms of a more primitive one.

> **⚙️ Under the hood →** Blanket impls are why the rustdoc page for a trait has an "Implementors" section that includes types you never wrote an `impl` for. They are also a coherence minefield — two overlapping blanket impls would violate uniqueness, so Rust (currently) forbids *overlapping* impls outright. The unstable `specialization` feature would relax this; on stable, if two blanket impls could both apply to some type, you get a conflict error.

## Supertraits

A trait may *require* another trait of its implementors, declared with the same `: Bound` syntax as a generic bound:

```rust
use std::fmt::Display;

trait OutlinePrint: Display {          // Display is a supertrait
    fn outline(&self) {
        let s = self.to_string();      // allowed: Self: Display is guaranteed
        println!("** {s} **");
    }
}
```

`trait OutlinePrint: Display` means "you cannot implement `OutlinePrint` for a type unless it also implements `Display`". Inside `OutlinePrint`'s default methods, `Self: Display` is therefore a usable hypothesis — hence the `self.to_string()` call type-checks. This is *not* inheritance in the OOP sense (no shared fields, no subtyping of values); it is a *requirement edge between contracts*. Java's `interface B extends A` is the closest cousin, and the semantics match: implementing `B` obliges you to satisfy `A`.

## Derivable traits: codegen you get for free

`#[derive(...)]` is a procedural macro ([chapter 18](18-macros.md)) that *writes a trait impl for you* based on your type's structure. The common ones, and what they generate:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default)]
struct Point { x: i32, y: i32 }
```

- **`Debug`** — `{:?}` formatting; generates a field-by-field dump. Required for `assert_eq!` failure messages.
- **`Clone`** — an explicit deep-ish `.clone()`; recursively clones each field.
- **`Copy`** — a *marker* (no methods): opts the type into implicit bitwise-copy move semantics from [ownership](02-ownership-and-moves.md). Only derivable if every field is `Copy`; requires `Clone` as a supertrait.
- **`PartialEq` / `Eq`** — `==`; field-wise comparison. `Eq` is a marker promising reflexivity (so `f64`, which has `NaN != NaN`, is `PartialEq` but **not** `Eq`).
- **`PartialOrd` / `Ord`** — `<`, `>`, sorting; lexicographic over fields in declaration order.
- **`Hash`** — feeds each field to a hasher; needed for `HashMap`/`HashSet` keys ([collections](10-collections.md)).
- **`Default`** — `Point::default()`; each field's own default.

Derive is generated source code: it monomorphises, costs nothing extra, and fails to compile if a field doesn't support the trait (you can't `derive(Hash)` if a field isn't `Hash`). For custom logic, hand-write the `impl` instead.

> **🎓 Tripos link →** *Foundations of CS.* In OCaml you got structural equality (`=`) and polymorphic comparison built into the language for free. Rust refuses implicit structural operations — but `derive` recovers the convenience *opt-in and explicitly*, per trait, so you always know (and the compiler always knows) precisely which capabilities a type has. No surprise `compare` on a closure-containing record.

## Marker traits: capabilities with no methods

Some traits have *zero* methods; they exist purely to tag a type with a compile-time property the compiler reasons about. You have already met `Copy`. The two that matter most arrive in [fearless concurrency](13-fearless-concurrency.md):

- **`Send`** — values of this type may be *moved* to another thread.
- **`Sync`** — `&T` may be *shared* across threads (equivalently, `&T: Send`).

These are **auto traits**: the compiler *derives them automatically and structurally* — a struct is `Send` iff all its fields are `Send`. You rarely write them by hand. Their entire purpose is to let the type system prove data-race freedom statically: a bound like `T: Send` on `thread::spawn` is what *makes* concurrency fearless, by rejecting at compile time any attempt to ship a non-thread-safe value (like `Rc<T>`) across a thread boundary.

> **🎓 Tripos link →** *Concurrent and Distributed Systems.* You proved properties about data races informally with happens-before reasoning. `Send`/`Sync` encode a slice of that reasoning *in the type system* as bounded quantification: `spawn<F: Send>`. Data-race freedom for safe Rust becomes a *type-soundness theorem* discharged by the borrow checker plus these markers — the absence of races is checked, not hoped for.

## Mental-model recap

- **Generics monomorphise.** rustc stamps a specialised copy per concrete type — C++ templates, not Java erasure. Zero runtime cost, paid in code size; types survive to runtime so `Vec<T>` stores `T` unboxed.
- **A generic body is type-checked once against its bounds**, before any instantiation. `T` can do *nothing* the bounds don't promise — bounded quantification, `∀T. C(T) ⇒ …`. This is why the body errors before any call site exists.
- **A trait is a type class / interface done right.** Implementations are separate items, so you can implement your trait for foreign types and foreign traits for your types — subject to the **orphan rule**, which guarantees **coherence**: one impl per (trait, type), globally. Newtype-wrap to escape it.
- **Associated type vs generic param = output vs input.** One impl per type (iterator's `Item`, index's `Output`) → associated type, inference-friendly. Many impls per type (`Add<Rhs>`) → generic param.
- **`impl Trait` argument-position = caller picks (generic); return-position = callee hides one fixed type.** Different types per branch ⇒ you need `Box<dyn Trait>`, not `impl Trait`. Operators, derives, and markers (`Copy`/`Send`/`Sync`) are *all* just traits.

## Exercises

1. Write `fn largest<T>(xs: &[T]) -> &T` and deliberately omit the bound. Read the `E0369` error. Now add `T: PartialOrd` — it compiles. Then change the return type to `T` (by value, returning `*best`) and explain the new error in terms of `T: Copy` versus `T: Clone`. Which bound is the right fix, and why does returning a *reference* avoid the question entirely?

2. Define `trait Area { fn area(&self) -> f64; }` and implement it for `Circle` and `Rectangle`. Write `fn total_area(shapes: &[???]) -> f64`. Try `&[impl Area]` — it fails. Explain why (hint: `impl Trait` is one anonymous type; a slice is homogeneous). What two *different* tools fix this depending on whether the slice is monomorphic or genuinely mixed?

3. **(★)** You want `MyVec(Vec<i32>)` to be printable with `{}`. First try `impl Display for Vec<i32>` directly and read the `E0117` orphan-rule error. Then fix it with the newtype pattern. Explain precisely which clause of the orphan rule the newtype satisfies, and why a single-field tuple struct adds no runtime cost.

4. The standard library declares `trait Iterator { type Item; … }`, not `trait Iterator<Item>`. Construct a concrete scenario — a single struct you'd *want* to implement the trait for two ways — that the associated-type version forbids but a generic-parameter version would allow. Then argue why forbidding it makes `for x in it` infer `x`'s type without annotations.

5. **(★)** Implement `Add` for a `Counter(u32)` newtype *twice*: once for `Counter + Counter`, once for `Counter + u32`, using the `Rhs` parameter. Now try to make `Iterator::Item` "two-valued" the same way and observe it is impossible. Use this to state the rule for choosing associated type vs generic parameter in your own words, then explain why `Add<Rhs = Self>` having a *default* is unrelated to the coherence question.

6. Write a blanket impl: `trait Loud { fn shout(&self) -> String; }` implemented for `impl<T: Display> Loud for T` as `self.to_string().to_uppercase()`. Confirm `42.shout()` and `"hi".shout()` both work with no per-type impls. Then add a *second* blanket impl `impl<T: Debug> Loud for T` and read the coherence-conflict error. Explain why overlapping blanket impls would violate global uniqueness, and what `specialization` would have to do to make it sound.
