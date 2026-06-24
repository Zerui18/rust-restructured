# 8. Generics & Traits

Two abstraction mechanisms collapse into one design here. **Generics** give you parametric polymorphism over types; **traits** give you *bounded* polymorphism — a way to say "for all types `T` *that support this interface*". You have met both ideas before under different names: Java's interfaces plus generics, Swift's protocols plus generics, OCaml's `'a` type variables. Rust's contribution is to make the whole thing *statically resolved and zero-cost by default*, with a coherence discipline that Java and OCaml both lack. That last point is the one worth fixing in your head: every abstraction in this chapter is compiled away. There is no boxing, no vtable, no reflection, no erasure — unless you explicitly ask for it (which is [chapter 9](09-trait-objects-and-oop.md)).

## Generics are monomorphised, not erased

A generic function is a *template* over types. The syntax echoes Java and Swift: type parameters live in angle brackets, declared between the name and the value parameters.

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

> **🦀 From your toolbox →** This is the **opposite** of Java generics. Java uses **type erasure** — `List<String>` and `List<Integer>` are one class `List` at runtime, with casts inserted and `T` reduced to `Object`. Java pays a uniform boxing/dispatch cost and loses the type at runtime (hence no `new T[]`); Rust keeps full type information and pays nothing at runtime. Swift sits in between: its generics keep type information through "witness tables" but still dispatch through them by default unless the optimiser specialises. Rust *always* specialises. The closest light analogy is a C++ template, which also stamps out a copy per type — and shares the same tradeoff: zero runtime penalty, paid for in compile time and code size. The C++ analogy breaks in one crucial place, covered below: C++ checks a template's body *only when instantiated* (late errors, infamous error messages), whereas Rust type-checks the generic body *once, against its bounds*, before any instantiation exists.

> **⚙️ Under the hood →** Monomorphisation happens after type-checking, during MIR-to-LLVM lowering. rustc collects the set of "mono items" reachable from `main` (or from `pub` API for libraries) and emits one LLVM function per distinct `(fn, type-args)` pair. Identical instantiations across crates are deduplicated at link time. This is also why generic functions usually must live in a crate's source (or be marked `#[inline]`) to be usable downstream — there is nothing to call until a concrete type pins the template down.

> **🎓 Tripos link →** *Compiler Construction.* There are two ways a compiler can implement polymorphism. One is a **uniform representation**: everything is a pointer, one copy of the code, the actual type figured out at runtime (this is how Java and OCaml work). The other is **specialisation/monomorphisation**: one copy of the machine code per concrete type, with the type baked in at compile time. Rust commits to specialisation for generics, which is why a generic `Vec<T>` can store `T` *unboxed and inline* in a flat buffer — impossible under erasure, where every element would have to be a boxed pointer.

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

## Why the bound is mandatory: the late-vs-early-check contrast

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

The intuition is worth stating in plain words: a bound is a *contract you must honour at the call site and may assume inside the body*. When you call `largest(&[1, 2, 3])`, the compiler checks once that `i32: PartialOrd` — that obligation is discharged there, at the call. Inside the function body, you get to treat `T: PartialOrd` as a given fact and use `>` freely. The error appears *before any call exists* precisely because the body must be checkable on the strength of its bounds alone, not on the hope that every future caller happens to pass a comparable type.

## Traits: protocols, interfaces, and shared behaviour

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

The shape `impl Trait for Type { ... }` is the whole game. It is deliberately *separate* from the type's own definition — unlike a Java `class Tweet implements Summary` or a Swift `class Tweet: Summary`, where the conformance list is welded to the type declaration. In Rust the type and the trait implementation are independent items, and that separation is what makes traits more powerful than interfaces or protocols.

> **🦀 From your toolbox →** Two simultaneous analogies, both close:
> - **Java interfaces / Swift protocols.** A trait is a list of required methods a type must provide, exactly like a protocol. The big difference: Java and Swift bolt the conformance list onto the type definition, so you can only declare conformance for types you own. Rust lets you implement a trait for a type you didn't define (subject to the orphan rule below). This is "protocols done right": you can make `i32` implement *your* trait, the way Swift extensions let you add protocol conformance to existing types — except Rust takes it further and lets you add *your own* trait to *standard-library* types.
> - **Python duck typing.** A trait is the explicit, compiler-checked version of "if it has a `.summarize()` method, treat it as summarizable". Python checks that at runtime and crashes if the method is missing; Rust checks it at compile time and the method is guaranteed present.
> Where both analogies break: a trait can be used for **static** dispatch (monomorphised, zero-cost) *or* **dynamic** dispatch (`dyn Trait`, a vtable). Java interfaces and Swift protocol types are normally dynamic; this chapter is all static, and [chapter 9](09-trait-objects-and-oop.md) is the dynamic half.

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

This is the classic "template method" pattern, but enforced by the type system: the trait author writes the *policy* (`summarize`) in terms of a small required *primitive* (`summarize_author`). An implementor writes one method and gets the other. It is the same idea as a Swift protocol extension that provides a default method body, or a Java interface `default` method, or an abstract base class whose concrete method calls an abstract one. You cannot, however, call the overridden default from within an override; there is no `super`.

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

> **🔧 In practice →** You're writing a logging helper that should accept *anything* a caller can already print — a `String`, an `i32`, a `PathBuf`, a custom struct — without forcing them to convert first. Argument-position `impl Trait` is exactly the tool: `fn log_line(label: &str, value: impl Display) { println!("[{label}] {value}"); }`. Now `log_line("port", 8080)`, `log_line("host", "localhost")`, and `log_line("path", some_pathbuf)` all compile, each monomorphised to the concrete type, no boxing. You reach for the named `<T: Display>` form instead the moment two parameters must be the *same* type — e.g. a `fn max_of<T: PartialOrd>(a: T, b: T) -> T` where comparing `a` and `b` only makes sense if they share a type.

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

> **🔧 In practice →** You're building a query layer and you want a function that returns the IDs of active users as an iterator, so the caller can chain `.take(10)` or `.collect()` without you materialising a `Vec` first. The concrete type is a monstrous adaptor chain you'd never want to type by hand:
> ```rust
> fn active_ids(users: &[User]) -> impl Iterator<Item = u64> + '_ {
>     users.iter().filter(|u| u.active).map(|u| u.id)
> }
> ```
> The return type stays readable (`impl Iterator<Item = u64>`), the dispatch stays static, and nothing is allocated. The day you need the function to return *one of two different* iterator shapes depending on a flag is the day you switch the return type to `Box<dyn Iterator<Item = u64>>` and pay for one vtable.

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

You could *imagine* writing this with a generic parameter instead: `trait Iterator<Item>`. Why does the standard library use an associated type? Because a given iterator type yields exactly **one** element type. With `trait Iterator<Item>`, a single type could legally implement `Iterator<u8>` *and* `Iterator<String>`, forcing you to annotate `Item` at every call (`it.next::<u8>()`-style) and breaking inference. With an associated type, `Item` is **an output determined by the implementing type, not an input chosen by the caller** — so `it.next()` infers cleanly.

> **🦀 From your toolbox →** Decide by counting implementations. **Generic parameter** = the trait can be implemented *many times* for one type with different parameters (e.g. `Add<i32>` *and* `Add<Duration>` for one type — see below). **Associated type** = at most *one* implementation makes sense per type, and the type is a *consequence* of the impl, not a choice (the element type of an iterator, the output of indexing). Swift draws the same line: a protocol with an `associatedtype Element` (one per conformer, like `Sequence.Element`) versus a generic constraint the caller supplies. If you've used Swift's `associatedtype`, this is the same mechanism.

> **🎓 Tripos link →** *Foundations of CS / type inference.* You can think of an associated type as a small *function on types*: given the iterator type, it hands back the element type, and there is exactly one answer. Because the answer is determined (one output per input), the compiler's type inference can simply look it up and keep going. A generic parameter is the opposite — it's an *unknown the compiler has to solve for* — which is why allowing two `Iterator<u8>` / `Iterator<String>` impls on one type would leave `Item` ambiguous and force you to annotate it everywhere.

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

> **⚙️ Under the hood →** `a + b` desugars to `Add::add(a, b)`, monomorphised and inlined to the same instructions you'd write by hand — no indirection. For primitives, the `impl Add for i32` exists in core and the optimiser folds the call to a single machine `add`. Rust deliberately does *not* let you invent *new* operator glyphs, keeping parsing and reasoning local: you can give meaning to the fixed set of built-in operators, but you cannot mint brand-new symbols.

## Coherence and the orphan rule

Here is the discipline that Java and OCaml lack. Rust guarantees **global uniqueness**: for any (trait, type) pair, there is *at most one* implementation in the entire program. This property is called **coherence**, and it is what lets `a + b` or `set.insert(x)` resolve deterministically without you ever specifying *which* `Add` or `Hash` impl to use. There is only ever one.

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

> **🎓 Tripos link →** *Databases.* The win from coherence is the same one you get from a *uniqueness constraint* on a key: because there is provably at most one `impl` per (trait, type) pair, looking up "which `Add` for `V2`?" is an unambiguous lookup with exactly one row, decided entirely at compile time. Java sidesteps the question a cruder way — it ties an implementation to the class definition, so you simply *can't* retro-fit an interface onto a type you don't own. That keeps things unambiguous too, but it is strictly less expressive than Rust's "implement it elsewhere, but only once" rule.

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

`trait OutlinePrint: Display` means "you cannot implement `OutlinePrint` for a type unless it also implements `Display`". Inside `OutlinePrint`'s default methods, `Self: Display` is therefore a usable hypothesis — hence the `self.to_string()` call type-checks. This is *not* inheritance in the OOP sense (no shared fields, no subtyping of values); it is a *requirement edge between contracts*. Java's `interface B extends A` and Swift's `protocol B: A` are the closest cousins, and the semantics match: implementing `B` obliges you to satisfy `A`.

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

> **🔧 In practice →** You're modelling a domain object — say a `Money { cents: i64, currency: Currency }` — and you want to use it as a `HashMap` key, compare two values in a test with `assert_eq!`, sort a `Vec<Money>`, and print it while debugging. Rather than hand-write five trait impls, you write one line:
> ```rust
> #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
> struct Money { cents: i64, currency: Currency }
> ```
> Now `map.insert(price, qty)` works (`Hash + Eq`), `assert_eq!(a, b)` prints a readable diff (`Debug + PartialEq`), and `prices.sort()` orders them (`Ord`). The moment your equality needs custom logic — say two `Money` values are equal only after normalising currency — you drop `PartialEq` from the derive list and hand-write that one impl, keeping the rest derived.

> **🎓 Tripos link →** *Foundations of CS.* In OCaml you got structural equality (`=`) and polymorphic comparison built into the language for free, applied to any value whether or not it made sense. Rust refuses implicit structural operations — but `derive` recovers the convenience *opt-in and explicitly*, per trait, so you always know (and the compiler always knows) precisely which capabilities a type has. No surprise comparison on a record that happens to contain a function.

## Marker traits: capabilities with no methods

Some traits have *zero* methods; they exist purely to tag a type with a compile-time property the compiler reasons about. You have already met `Copy`. The two that matter most arrive in [fearless concurrency](13-fearless-concurrency.md):

- **`Send`** — values of this type may be *moved* to another thread.
- **`Sync`** — `&T` may be *shared* across threads (equivalently, `&T: Send`).

These are **auto traits**: the compiler *derives them automatically and structurally* — a struct is `Send` iff all its fields are `Send`. You rarely write them by hand. Their entire purpose is to let the type system prove data-race freedom statically: a bound like `T: Send` on `thread::spawn` is what *makes* concurrency fearless, by rejecting at compile time any attempt to ship a non-thread-safe value (like `Rc<T>`) across a thread boundary.

> **🎓 Tripos link →** *Concurrent and Distributed Systems.* The whole problem you study there — data races, where two threads touch the same memory and at least one writes — Rust pushes into the type system. `Send` and `Sync` are the tags that say "this value is safe to move to / share with another thread". A function like `thread::spawn` requires its closure to be `Send`, so the compiler simply refuses to compile code that would hand a thread-unsafe value (like `Rc<T>`) to another thread. The races you'd otherwise have to find with careful reasoning are rejected before the program runs.

## Mental-model recap

- **Generics monomorphise.** rustc stamps a specialised copy per concrete type — like a C++ template, *not* like Java erasure. Zero runtime cost, paid in code size; types survive to runtime so `Vec<T>` stores `T` unboxed.
- **A generic body is type-checked once against its bounds**, before any instantiation. `T` can do *nothing* the bounds don't promise — a bound is a contract checked at the call site and assumed inside the body. This is why the body errors before any call site exists.
- **A trait is an interface / protocol done right.** Implementations are separate items, so you can implement your trait for foreign types and foreign traits for your types — subject to the **orphan rule**, which guarantees **coherence**: one impl per (trait, type), globally. Newtype-wrap to escape it.
- **Associated type vs generic param = output vs input.** One impl per type (iterator's `Item`, index's `Output`) → associated type, inference-friendly. Many impls per type (`Add<Rhs>`) → generic param.
- **`impl Trait` argument-position = caller picks (generic); return-position = callee hides one fixed type.** Different types per branch ⇒ you need `Box<dyn Trait>`, not `impl Trait`. Operators, derives, and markers (`Copy`/`Send`/`Sync`) are *all* just traits.

## Exercises

1. You have `fn largest<T: PartialOrd>(xs: &[T]) -> &T`. A colleague wants the by-value version `fn largest<T: PartialOrd>(xs: &[T]) -> T` so the caller owns the result. Predict the compile error before trying it, then decide: should the fix be adding `T: Copy` or `T: Clone`? Give one concrete element type for which `Copy` suffices and one for which only `Clone` will do — and explain why the original *reference*-returning version needed neither.

2. Here are three signatures. Which compile, and what does each one *let the caller do* that the others don't?
   ```rust
   fn a(x: &impl Summary, y: &impl Summary);
   fn b<T: Summary>(x: &T, y: &T);
   fn c(x: &dyn Summary, y: &dyn Summary);
   ```
   For each, say whether `x` and `y` may be different concrete types, and which one(s) you'd pick if the body needs to put `x` and `y` into the same `Vec`.

3. You're given a foreign type `chrono::DateTime` (you don't own it) and a foreign trait `serde::Serialize` (you don't own it either), and you need `DateTime` to be serializable. Explain why a direct `impl Serialize for DateTime` is rejected, then describe — in terms of *which* clause of the orphan rule each option satisfies — the two different escape hatches: the newtype wrapper, and (the one real projects actually use) serde's `#[serde(with = "...")]` adapter. Why does the newtype add zero runtime cost?

4. A teammate proposes changing `Iterator` from `trait Iterator { type Item; ... }` to `trait Iterator<Item> { ... }` "for flexibility". Construct one concrete struct that the generic-parameter version would let you implement *two ways* but the associated-type version forbids. Then explain what breaks in everyday `for x in it { ... }` loops if the standard library had taken your teammate's advice.

5. **(★)** You're designing a units library. You want `Meters + Meters`, `Meters + Centimeters`, and also a single canonical "length of this value" notion per `Meters`. Decide for each of these whether it should be a *generic parameter* or an *associated type*, and implement the `Add` parts (two impls on `Meters`). Then state, in your own words, the one-sentence rule for choosing between the two — and explain why the fact that `Add<Rhs = Self>` has a *default* is a separate question from coherence.

6. Suppose Rust let you write *two* overlapping blanket impls:
   ```rust
   impl<T: Display> Loud for T { /* uppercased Display */ }
   impl<T: Debug>   Loud for T { /* uppercased Debug   */ }
   ```
   Pick a single type that matches *both* bounds (most std types do), and walk through exactly what goes wrong when the compiler tries to resolve `that_value.shout()`. Use that to explain in one sentence why coherence forbids overlapping impls, and what a hypothetical `specialization` feature would need to add to make a choice unambiguous.
