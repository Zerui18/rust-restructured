# 17. Advanced Traits & Types

[Module 8](08-generics-and-traits.md) gave you the working core of the trait system: declare an interface, bound a generic on it, get monomorphised static dispatch. That is 90% of day-to-day Rust. This chapter is the remaining 10% — the machinery that the standard library itself is built on, and that you reach for when you are *designing* an abstraction rather than consuming one. Two threads run through it. The first is **how the type system lets a trait carry types and constants, not just methods**. The second is **what counts as a "type" at all** — the never type, dynamically sized types, function pointers — which turns out to be where Rust's safety story meets the machine. Everything here is still resolved statically; nothing acquires a runtime cost it did not already have.

## Associated types: a trait that outputs a type

A trait can declare a *type member* that each implementor fills in. The canonical example is `Iterator`:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

`Item` is a placeholder. When you `impl Iterator for Counter`, you write `type Item = u32;` once, and from then on `Self::Item` *is* `u32` everywhere in that impl. The decisive question for this reader: why is this not just a generic parameter? Why not `trait Iterator<Item>`?

The difference is **cardinality of implementations**. A generic trait `Iterator<Item>` could be implemented *many times* for one type — `impl Iterator<u32> for Counter` and `impl Iterator<String> for Counter` would coexist. The associated-type form pins it to *exactly one*: there is one `impl Iterator for Counter`, so `Counter`'s `Item` is determined, full stop. That uniqueness is what lets you write `counter.next()` with no turbofish — the compiler does not need you to disambiguate which iterator you meant, because there is only one.

The rule of thumb: **associated type when the implementor determines the type as a function of `Self`; generic parameter when the caller chooses it.** `Iterator::Item` is fixed by the iterator. `Add<Rhs>` (below) is parameterised because you genuinely want `f32 + f32` *and* `f32 + &f32`.

> **🦀 From your toolbox →** This is the input/output type distinction from Haskell type classes. A generic trait parameter is an *input* type — it participates in instance selection, so `Add<i32>` and `Add<f64>` are different instances. An associated type is an *output* type — functionally determined by the inputs, like a type family / functional dependency. OCaml has no direct analogue; the nearest is a module signature with an abstract `type t` that a functor fixes on application. The analogy breaks in that OCaml modules are explicitly applied, whereas Rust's impl is selected implicitly by trait resolution.

> **🎓 Tripos link →** *Semantics of Programming Languages.* An associated type is a type-level function `Self ↦ Self::Item`, defined by the impl. Trait coherence (the orphan rule plus the one-impl restriction) is exactly the side-condition that makes this a *function* and not a relation — without it, `Self::Item` would not be well-defined, and type inference would lose the principal-type property.

### Associated constants

The same idea extends to constants. A trait may declare a `const` that each impl supplies, available as `Self::NAME`:

```rust
trait Bounded {
    const MIN: Self;
    const MAX: Self;
}

impl Bounded for i8 {
    const MIN: Self = -128;
    const MAX: Self = 127;
}
```

This is how `i32::MAX` and friends are actually defined in `core`. Because the value is a compile-time constant tied to the type, it monomorphises away — `T::MAX` in a generic function becomes an immediate after instantiation, no lookup.

## Default type parameters and operator overloading

A generic *parameter* (not associated type) may carry a default, written `<T = Default>`. The std library's `Add` is the textbook case:

```rust
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

`Rhs` defaults to `Self`, so the common case — adding two values of the same type — needs no annotation. Note the two roles working together: `Rhs` is a *generic parameter* (the caller can vary it), `Output` is an *associated type* (the impl fixes it). To overload `+` for your own type, implement `Add`:

```rust
use std::ops::Add;

#[derive(Clone, Copy, PartialEq, Debug)]
struct Vec2 { x: f64, y: f64 }

impl Add for Vec2 {            // Rhs defaults to Vec2
    type Output = Vec2;
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}
```

Now `a + b` desugars to `Add::add(a, b)`. You can override the default `Rhs` to make *heterogeneous* operators — scaling a `Vec2` by a scalar, say:

```rust
impl Add<f64> for Vec2 {      // Rhs = f64, a distinct impl
    type Output = Vec2;
    fn add(self, k: f64) -> Vec2 {
        Vec2 { x: self.x + k, y: self.y + k }
    }
}
```

Both impls coexist because `Add<Vec2>` and `Add<f64>` are different traits as far as coherence is concerned — the generic parameter is part of the trait's identity.

Rust will not let you invent new operator glyphs (no `**` for power), and you cannot overload `&&`, `||`, or `?`. The overloadable set is fixed: the traits in `std::ops` (`Add`, `Sub`, `Mul`, `Index`, `Deref`, `Neg`, `AddAssign`, …).

> **🦀 From your toolbox →** C++ operator overloading is freeform — any `operator+` member or free function, with ad-hoc overload resolution. Rust routes everything through a *named trait*, which means (a) the overload set is closed and discoverable, and (b) generic code can bound on `T: Add<Output = T>` and rely on `+` working. The cost is rigidity: no overloading `<<` to mean "stream insert", because `Shl` means shift and that is the only meaning.

> **⚙️ Under the hood →** `a + b` is sugar; rustc lowers it to the trait-method call and monomorphises. For a `Copy` `Vec2`, the two `f64` adds inline to scalar FP instructions with no struct ever materialised on the stack in optimised builds. Operator overloading carries zero dispatch cost — it is the same machinery as any other generic trait call.

## Disambiguation: fully-qualified syntax

Nothing stops two traits — or a trait and an inherent impl — from defining a method of the same name on one type. Method-call syntax `x.foo()` resolves by an ordering rule: inherent methods win, then trait methods in scope. When that picks the wrong one, you name the trait explicitly.

```rust
trait Pilot { fn announce(&self) -> &str; }
trait Sailor { fn announce(&self) -> &str; }

struct Crewmate;
impl Pilot for Crewmate  { fn announce(&self) -> &str { "cleared for takeoff" } }
impl Sailor for Crewmate { fn announce(&self) -> &str { "anchors aweigh" } }

let c = Crewmate;
Pilot::announce(&c);            // disambiguate by trait, &self inferred
Sailor::announce(&c);
```

When the method takes `&self`, naming the trait suffices: the receiver's type tells rustc which impl. The genuinely hard case is an **associated function with no `self`** — there is no receiver to read the type off, so trait-name syntax fails:

```rust
trait Named { fn canonical() -> String; }
struct Widget;
impl Widget { fn canonical() -> String { "inherent".into() } }
impl Named for Widget { fn canonical() -> String { "trait".into() } }

// Named::canonical()  // error[E0790]: cannot call associated function
//                     // on trait without specifying the impl type
let s = <Widget as Named>::canonical();   // "trait"
```

The `<Type as Trait>::method(...)` form is **fully-qualified syntax**. It says "treat `Widget` as a `Named` and call *that* `canonical`." The general shape is `<Type as Trait>::function(receiver_if_any, args...)`. You only need it when name resolution is genuinely ambiguous; everywhere else, rustc elides the parts it can infer.

> **⚠️ Pitfall →** Calling `Named::canonical()` without the `<Widget as ...>` prefix gives `error[E0790]: cannot call associated function on trait without specifying the corresponding 'impl' type`. The fix is the cast form, which rustc even suggests verbatim. This bites most often with `Default::default()` and `FromStr::from_str` inside generic code — write `T::default()` or `<T as Default>::default()`.

## Supertraits: requiring another trait

A trait may *depend on* another: to implement it, you must also implement the supertrait. Syntax is the same as a bound, on the trait itself.

```rust
use std::fmt::Display;

trait Framed: Display {
    fn framed(&self) -> String {
        let body = self.to_string();          // to_string comes from Display
        let bar = "=".repeat(body.len() + 2);
        format!("{bar}\n {body}\n{bar}")
    }
}
```

`Framed: Display` means every `Framed` type is also `Display`, so inside `framed` you may call `Display`'s methods (`to_string`). Try to `impl Framed for Point` without a `Display` impl and you get `error[E0277]: Point doesn't implement std::fmt::Display`, with a note pointing at the `required by a bound in Framed`. Supertraits are not inheritance — there is no subtyping, no `is-a` — they are a *precondition* on the impl, exactly a trait bound that happens to live in the trait header rather than on a function.

> **🦀 From your toolbox →** Looks like Java `interface Framed extends Display`, and the surface effect (a `Framed` is usable where `Display` is wanted) matches. But there is no class hierarchy and no dynamic upcast: a `Box<dyn Framed>` does not silently coerce to `Box<dyn Display>` (trait-object upcasting only stabilised recently and must be spelled out). The relationship is a static obligation discharged at impl time, not a runtime type relation.

## The newtype pattern: a tuple struct with one field

Wrap an existing type in a single-field tuple struct and you get a brand-new, distinct type at zero runtime cost. This one pattern solves three different problems.

**(a) Sidestep the orphan rule.** Coherence forbids `impl ForeignTrait for ForeignType` — you can implement a trait only if the trait or the type is local. To `impl Display for Vec<String>` (both foreign), wrap the `Vec`:

```rust
use std::fmt;
struct Csv(Vec<String>);

impl fmt::Display for Csv {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.0.join(","))   // self.0 is the inner Vec
    }
}
```

`Csv` is local, so the impl is allowed.

**(b) Enforce invariants and units in the type system.** A `struct Meters(f64)` and a `struct Seconds(f64)` are *non-interchangeable*: a function taking `Meters` rejects a `Seconds` or a bare `f64` at compile time. This is the units-of-measure / typestate technique — encode a domain distinction that the underlying representation does not capture, and let the type checker enforce it for free.

```rust
struct Validated(String);    // can only be built by `validate`

fn validate(raw: String) -> Option<Validated> {
    if raw.chars().all(|c| c.is_ascii_alphanumeric()) {
        Some(Validated(raw))
    } else {
        None
    }
}
```

If the inner field is private (in its own module), the *only* way to obtain a `Validated` is through `validate`. Any function taking `Validated` now has a static proof the string was checked — a poor-man's refinement type.

**(c) Hide the implementation.** A newtype exposes whatever public API you give it and nothing else. Wrap a `HashMap<u64, String>` as `struct Registry(HashMap<u64, String>)`, expose `insert(&mut self, name: &str)`, and callers cannot depend on the fact that you used a hashmap — you can swap it for a `BTreeMap` without breaking them.

> **⚙️ Under the hood →** A single-field tuple struct has *identical layout* to its field; the wrapper is erased. `Meters(3.0)` is one `f64` in a register. This is why the pattern is free — newtypes are a compile-time fiction. If you want the wrapper to forward the inner type's methods, implement `Deref` ([smart pointers](15-smart-pointers.md)); if you want only a subset, write just those.

> **🎓 Tripos link →** *Semantics.* This is nominal typing deployed deliberately on top of a structurally-identical representation — the static enforcement of an abstraction boundary you proved sound when you studied data abstraction and representation independence. The `Validated` newtype is a constructive existential: holding one is evidence the invariant held at construction.

## Type aliases vs newtypes: same bits, opposite meaning

A `type` alias is *not* a new type — it is a synonym, fully transparent to the type checker.

```rust
type Metres = f64;
let d: Metres = 3.0;
let t: f64 = d;          // fine — Metres IS f64
```

Contrast `struct Metres(f64)`, which would reject that assignment. Aliases give you **no** safety; their purpose is purely to shorten verbose types and centralise them. The two idiomatic uses:

```rust
type Handler = Box<dyn Fn(i32) -> i32 + Send + 'static>;   // tame a long type
```

and the partially-applied `Result` you have already been using — `std::io::Result<T>` is literally `type Result<T> = std::result::Result<T, std::io::Error>;`. Because it is the *same* type, the `?` operator and every `Result` method work on it unchanged. Decide between the two by asking: *do I want the compiler to stop me mixing this up with the underlying type?* Yes → newtype. No, I just want a shorter name → alias.

## The never type `!` and divergence

`!` is the type with **no values** — the empty/uninhabited type, called *never*. A function annotated `-> !` promises it will never return normally:

```rust
fn fatal(msg: &str) -> ! {
    eprintln!("{msg}");
    std::process::exit(1);
}
```

`panic!`, `std::process::exit`, `continue`, `break`, `return`, and an infinite `loop {}` all have type `!`. The point of a type you cannot construct is **coercion**: `!` unifies with *every* type. That is what makes mixed `match` arms type-check:

```rust
let n: u32 = match s.trim().parse() {
    Ok(v) => v,                 // : u32
    Err(_) => continue,         // : !  →  coerces to u32
};
```

Both arms must agree on a type. The `Ok` arm is `u32`; the `Err` arm is `!`, which the unifier happily narrows to `u32` because there is no value of `!` that could violate it. Same mechanism in `Option::unwrap`: the `Some(v) => v` arm is `T`, the `None => panic!(...)` arm is `!`, the whole `match` is `T`. The expression "returns" `T` precisely because the panic arm provably contributes no value.

> **🎓 Tripos link →** *Logic and Proof / Semantics.* Under Curry–Howard, `!` is the type of *false* (⊥), the empty sum. `! → T` for any `T` is **ex falso quodlibet** — from a proof of false, anything follows. A diverging function is an inhabitant of `⊥`'s eliminator: it discharges the obligation to produce a `T` by demonstrating control never reaches a return. This is also why `!` is a subtype of every type in the coercion lattice: the empty set is a subset of everything.

> **⚙️ Under the hood →** `!` carries no data, so a value of type `!` occupies zero bytes and the codegen for a diverging arm is just the unconditional jump/trap — there is no "convert `!` to `u32`" instruction because control never arrives at the join point. On stable Rust `!` is not yet nameable in arbitrary positions (the `never_type` feature), but it is fully load-bearing in inference, which is all the above relies on.

## Dynamically sized types and `Sized`

rustc must know a type's size to put it on the stack, pass it by value, or store it inline. Most types are `Sized`: their size is a compile-time constant. A few are **dynamically sized types (DSTs)** — their size is only known at runtime. The three you meet are `str`, `[T]` (the slice, not the array `[T; N]`), and `dyn Trait`.

```rust
// let bad: str = *"hello";   // error[E0277]: the size of str cannot be
//                            // known at compilation time
let ok: &str = "hello";       // a fat pointer: (ptr, len)
```

`str` is unsized because two `str` values can have different byte lengths, so no single stack slot fits all of them. **The golden rule of DSTs: a value of a DST must always live behind a pointer.** `&str`, `Box<str>`, `Rc<str>`, `&[T]`, `Box<dyn Trait>` are all fine — and crucially they are **fat pointers**, carrying the extra metadata that recovers the missing size: a length for `str`/`[T]` ([slices](05-slices-and-duality.md) cover this), a vtable pointer for `dyn Trait` ([trait objects](09-trait-objects-and-oop.md)).

The `Sized` trait marks "size known at compile time." It is an **auto trait** the compiler implements for you; you never write `impl Sized`. Its quiet importance is that rustc inserts `T: Sized` as an *implicit* bound on every generic parameter. These two are the same:

```rust
fn print_it<T>(x: T) {}
fn print_it<T: Sized>(x: T) {}      // identical
```

To accept a DST generically you must *opt out* with the special `?Sized` ("optionally sized") bound — and then you must take it behind a pointer, because you no longer know its size:

```rust
fn announce<T: ?Sized + std::fmt::Display>(x: &T) {
    println!(">> {x}");
}
announce("a str slice");            // T = str, unsized, fine via &T
announce(&42);                      // T = i32, sized, also fine
```

`?Sized` is the *only* `?Trait` relaxation in the language — it exists solely because `Sized` is the one bound applied by default.

> **🦀 From your toolbox →** A fat pointer is what C++ programmers reconstruct by hand as `{T* ptr; size_t len}` for a span, or what a vtable-carrying pointer is under the hood for virtual dispatch. Rust makes both first-class and type-safe: `&[T]` *is* the (ptr,len) pair, `&dyn Trait` *is* the (ptr,vtable) pair, and the compiler tracks which kind of metadata each DST needs. The analogy to a C++ reference breaks down because a C++ `T&` is always thin — there is no language-level notion of an unsized referent.

> **🎓 Tripos link →** *Computer Architecture & OS.* `Sized` is the type-system surface of "how many bytes does this occupy on the stack frame." A DST has no answer until runtime, so it cannot be a stack local; the pointer-plus-metadata representation is exactly the indirection you would hand-code to store variable-length data, lifted into the type checker so misuse is a compile error rather than a buffer overrun.

## Function pointers vs closures

You can pass a *named function* where a value is expected; it coerces to the type `fn(...) -> ...` — the **function pointer** (lowercase `fn`, distinct from the `Fn` closure traits).

```rust
fn double(x: i32) -> i32 { x * 2 }

fn apply_twice(f: fn(i32) -> i32, x: i32) -> i32 {
    f(f(x))
}

apply_twice(double, 3);     // 12
```

A bare `fn` is a thin pointer to code — no captured environment, so it is `Copy`, has a fixed size, and is FFI-compatible. A closure is a compiler-generated struct holding its captures (see [closures](12-closures-and-iterators.md)). Crucially, `fn` pointers implement all three closure traits (`Fn`, `FnMut`, `FnOnce`), so a function pointer is accepted anywhere a closure is. The converse fails: a closure with captures has no `fn` type. The idiom is therefore to write APIs generic over `F: Fn(...)`, which accepts both; reach for the concrete `fn` type only when you must — chiefly **FFI callbacks**, since C takes function pointers and has no concept of a capturing closure.

A neat consequence: tuple-struct and enum-variant constructors *are* functions, so they are valid `fn` pointers:

```rust
let xs: Vec<Option<u32>> = (0..3).map(Some).collect();  // Some used as fn(u32)->Option<u32>
```

## Returning closures

A closure has an anonymous, un-nameable type, so you cannot write its type in a return position by hand. Two ways out, which you have met before:

```rust
fn adder(n: i32) -> impl Fn(i32) -> i32 {     // static: opaque, monomorphised
    move |x| x + n
}

fn adder_boxed(n: i32) -> Box<dyn Fn(i32) -> i32> {   // dynamic: heap, vtable
    Box::new(move |x| x + n)
}
```

`impl Fn` returns an **opaque type**: the caller sees only "some type implementing `Fn`," but it is one concrete monomorphised type with no allocation or dispatch — prefer it. Its limitation is that *each* `impl Trait` site is a *distinct* opaque type, so you cannot return different closures from the same function or store two `impl Fn`-returning results in one `Vec`:

```rust
// error[E0308]: ... distinct uses of `impl Trait` result in different opaque types
// let v = vec![adder(1), adder_with_other_body(2)];
```

When you need a *homogeneous* collection of differently-bodied closures sharing a signature, erase to a trait object — `Box<dyn Fn(i32) -> i32>` — which gives every element the same type at the cost of one heap box and a vtable hop.

## A look further: const generics, `PhantomData`, GATs

**Const generics** let a type be generic over a *value*, most often an array length. This is what makes `[T; N]` uniformly usable:

```rust
struct Ring<const N: usize> { buf: [u32; N], head: usize }

impl<const N: usize> Ring<N> {
    fn capacity(&self) -> usize { N }      // N is an associated const, in effect
}
let r = Ring::<8> { buf: [0; 8], head: 0 };
```

`N` is monomorphised exactly like a type parameter: `Ring<8>` and `Ring<16>` are separate types with separately-stamped code. This finally lets you write one `impl` covering arrays of every length, instead of the macro-generated per-size impls that predated stable const generics.

**`PhantomData<T>`** is a zero-sized marker that makes a type *act* as though it owns or borrows a `T` without storing one. You need it when a type parameter appears in a struct's *logic* but not its fields — to drive variance, drop-check, and the `Send`/`Sync` auto-trait analysis correctly:

```rust
use std::marker::PhantomData;
struct Id<T> { raw: u64, _marker: PhantomData<T> }   // an Id<User> ≠ an Id<Order>
```

`Id<User>` and `Id<Order>` are distinct types — a typestate tag — yet both are just a `u64` at runtime, because `PhantomData` occupies zero bytes.

**Generic associated types (GATs)** lift associated types to take their own generic parameters, most importantly lifetimes — letting an associated type *borrow from `Self`*. This is what a lending iterator needs:

```rust
trait Lend {
    type Item<'a> where Self: 'a;
    fn get<'a>(&'a self) -> Self::Item<'a>;
}
```

GATs were stabilised relatively late because the borrow-checking and well-formedness rules around a lifetime-parameterised type member are subtle (note the `where Self: 'a` obligation, which says the borrow cannot outlive `Self`). You will rarely *write* one, but you will see them in iterator and async-trait libraries. The advanced corners of this — variance, drop-check, the soundness conditions PhantomData feeds — live in [unsafe Rust](16-unsafe-and-ffi.md).

## Mental-model recap

- **Associated type vs generic parameter**: associated type when the *implementor* determines it (one impl, no annotation — `Iterator::Item`); generic parameter when the *caller* varies it (many impls — `Add<Rhs>`). Default type params (`Rhs = Self`) make the common case annotation-free.
- **Newtype** = single-field tuple struct, zero runtime cost: it defeats the orphan rule, encodes units/invariants/typestate, and hides representation. A `type` alias does *none* of this — it is a transparent synonym for shortening verbose types. Choose by whether you want the compiler to forbid mix-ups.
- **`!`** is the empty type; it coerces into any type, which is *why* `match` arms ending in `panic!`/`continue`/`return` type-check (Curry–Howard: ⊥ ⊢ anything).
- **DSTs** (`str`, `[T]`, `dyn Trait`) have no compile-time size, so they live only behind **fat pointers** carrying length or a vtable. `Sized` is an implicit bound on every generic; `?Sized` (the only `?Trait` form) opts out, forcing a `&T` receiver.
- **`fn` pointers** are thin, `Copy`, FFI-safe, and implement all `Fn*` traits; closures with captures do not coerce to `fn`. Return closures with `impl Fn` (opaque, free) unless you need heterogeneous ones in one collection, then `Box<dyn Fn>`.

## Exercises

1. Define `trait Producer { type Output; fn make(&self) -> Self::Output; }` and implement it twice for one struct returning different `Output` types. Read the error, then explain in one sentence why the associated-type design forbids what a generic `Producer<T>` would have allowed.

2. Implement `std::ops::Mul<f64> for Vec2` (scalar scaling) *and* `Mul<Vec2> for Vec2` (component-wise). Confirm both `v * 2.0` and `v * w` compile. Then try adding a *second* `impl Mul<f64> for Vec2` and identify the coherence error by code.

3. Write `struct Email(String)` with a private field and a `fn parse(s: &str) -> Option<Email>` constructor in its own module. From `main`, attempt `Email("x".into())` directly. Which error fires, and what does that prove about every value of type `Email` reaching the rest of your program?

4. The expression `let x: i32 = if cond { 5 } else { panic!() };` compiles. Replace `panic!()` with `0i64` and explain the new error in terms of arm unification and why `!` succeeded where `i64` failed.

5. (★) Write `fn longest<T: ?Sized + AsRef<str>>(a: &T, b: &T) -> &T`. Call it once with `&str` arguments and once with `&String`. Then remove `?Sized` and explain precisely which call breaks and why the implicit `Sized` bound was the culprit (name the type that is not `Sized`).

6. (★) Write two functions both returning `impl Fn(u32) -> u32` with different bodies, then try to put their results in one `Vec`. Read `error[E0308]` about distinct opaque types. Fix it with `Box<dyn Fn>` and explain what changed about the elements' static type — and what runtime cost you just accepted.
