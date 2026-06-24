# 9. Trait Objects, Dynamic Dispatch & "OOP"

[Generics and traits](08-generics-and-traits.md) gave you *static* polymorphism: `fn render<T: Draw>(x: &T)` is, after monomorphisation, a family of distinct functions — one per concrete `T`, each with the call to `draw` resolved at compile time and eligible for inlining. That is the Java-with-erasure story turned inside out (Compiler Construction): where Java keeps one bytecode body and erases the type, Rust keeps the type and clones the body, stamping out a separate copy per concrete type.

Monomorphisation has a hard precondition: **the set of concrete types must be known when the generic is instantiated.** A `Vec<T>` is homogeneous — every element is the same `T`. The moment you want a *heterogeneous* collection — a list of GUI widgets where the library author cannot enumerate the widget types because downstream crates will invent new ones — static dispatch cannot help. You need a single type that erases the concrete identity of its contents but retains the ability to call a known set of methods. That type is `dyn Trait`, and reaching it costs a pointer indirection per call. This chapter covers that machinery, the rules that make it sound, and the question underneath: is Rust OOP, and if not, what replaces inheritance?

## `dyn Trait` is an unsized type behind a fat pointer

`dyn Draw` is a real type, but a peculiar one: it has no statically known size. A `Button` is 12 bytes, a `SelectBox` is 32, and `dyn Draw` could be either — so `dyn Draw` is `!Sized` (does not implement the `Sized` marker trait). You can never hold one by value on the stack, since the compiler cannot reserve space for an object of unknown size; you touch it through a pointer: `&dyn Draw`, `&mut dyn Draw`, `Box<dyn Draw>`, `Rc<dyn Draw>`.

That pointer is **fat**. A `&Button` is one machine word; a `&dyn Draw` is *two*:

```text
&dyn Draw
┌────────────┬────────────┐
│  data ptr  │ vtable ptr │
└────────────┴────────────┘
      │              │
      │              └──────────►  vtable for `Button as Draw`  (static, in .rodata)
      │                           ┌──────────────────────────┐
      │                           │ drop_in_place(*mut Button)│
      │                           │ size  = 12                │
      │                           │ align = 4                 │
      │                           │ &<Button as Draw>::draw   │
      │                           └──────────────────────────┘
      └──────────────────────────►  the Button bytes themselves
```

The data pointer points at the value; the vtable pointer points at a per-(type, trait) table of function pointers plus the destructor, size, and alignment. On a 64-bit target, `size_of::<&Button>() == 8` but `size_of::<&dyn Draw>() == 16`; likewise `Box<Button>` is 8 bytes and `Box<dyn Draw>` is 16. A call `d.draw()` compiles to: load the vtable pointer, load the `draw` slot, call it with the data pointer as `&self`.

> **⚙️ Under the hood →** The vtable is *not* embedded in the value — this is the crucial divergence from C++. A `Button` instance contains zero overhead: no vptr field, no per-object cost, identical layout whether or not `Draw` is implemented. The vtable lives wherever the `&dyn` is constructed. A single concrete type can have *many* vtables (one per trait it's viewed through: `&dyn Draw`, `&dyn Debug`, …), and they are coined at the `&value as &dyn Trait` coercion site, deduplicated into read-only static memory by the linker.

> **🦀 From your toolbox →** In Java, every object header carries a hidden class pointer used both for dynamic dispatch and for `instanceof` — the type info travels *inside* the object. Swift does much the same for classes, and a class with overridable methods has a per-instance vtable pointer. Rust flips this: the object stays thin and the vtable rides alongside the *reference*. A Java reference is one word and self-describing (it can always find its own class); a Rust `&dyn Draw` is two words, but a plain `&Button` carries no type info at all — it's just an address. (If you've seen C++, the closest light touch is that a class with a virtual method grows a hidden pointer inside every object; Rust never does that.) Rust also has no object header and no built-in `instanceof`; `dyn Any` ([Chapter 17](17-advanced-traits-and-types.md)) bolts on downcasting explicitly. The analogy breaks hardest here: **Rust separates "the value" from "the trait view of the value,"** which is why the same `Button` can sit behind a `&dyn Draw` and a `&dyn Debug` at once with no extra storage in the `Button`.

> **🎓 Tripos link →** This is the dispatch-table mechanism from Compiler Construction with one choice flipped: in a typical class-layout pass the vtable pointer is emitted as a *field of the object*. Rust passes the table *alongside* the value instead of embedding it. Concretely: when you write `&button as &dyn Draw`, the compiler bundles the address of the button with a pointer to a small static table of that type's `Draw` method addresses, and method calls jump through that table. Same dispatch idea, different place to keep the table.

## Static vs dynamic dispatch: the explicit tradeoff

You now have two ways to write "operate on anything that implements `Draw`":

```rust
fn render_static<T: Draw>(item: &T) { item.draw(); }   // monomorphised per T
fn render_dynamic(item: &dyn Draw)  { item.draw(); }    // one body, indirect call
```

`render_static` is compiled once per concrete `T`, each copy resolving the call to a direct, inlinable call to that `T`'s `draw`: code bloat (N copies) for zero runtime indirection and full optimisation across the call boundary. `render_dynamic` is compiled exactly once; every call does a vtable load and an indirect jump the optimiser cannot see through — no inlining, no constant-propagation into `draw`, a possible branch mispredict: code thinness for an indirection.

The decision rule for *this* reader:

- **Homogeneous & hot, or you want maximal optimisation:** generics. `Vec<Button>` where everything is a `Button`.
- **Heterogeneous (must store mixed types together), or you're crossing a plugin/extensibility boundary you don't control:** `dyn`. `Vec<Box<dyn Draw>>` where elements are `Button`, `SelectBox`, downstream `Image`, …
- **Compile-time blowup matters** (huge generic function, embedded binary size): `dyn` collapses N monomorphisations into one.

The cost of `dyn` is real but usually small — one cache-resident indirect call. Reach for it when you genuinely need runtime heterogeneity, not reflexively.

> **🔧 In practice →** You're building a logging layer and want callers to plug in any number of sinks at runtime — stdout, a file, a network socket, a test buffer — chosen from config you read at startup, not at compile time. You can't name the set of sink types in advance, and you want them all in one list:
> ```rust
> trait Sink { fn write_line(&mut self, line: &str); }
>
> struct Logger { sinks: Vec<Box<dyn Sink>> }
>
> impl Logger {
>     fn log(&mut self, line: &str) {
>         for s in &mut self.sinks { s.write_line(line); }
>     }
> }
> ```
> A `Vec<Box<dyn Sink>>` is exactly right here: the config decides at runtime which sinks exist, and a third-party crate can add a `SyslogSink` without you recompiling `Logger`. The one indirect call per sink is invisible next to the I/O it's doing. If instead you had *one* known sink type and a hot loop, you'd drop the `dyn` and make `Logger<S: Sink>` generic.

## A worked example: the extensibility boundary

The canonical motivation is a library whose author cannot know all the types its users will plug in. A widget framework defines the contract:

```rust
pub trait Render {
    fn paint(&self) -> String;
}

pub struct Canvas {
    pub widgets: Vec<Box<dyn Render>>,
}

impl Canvas {
    pub fn repaint(&self) {
        for w in &self.widgets {
            println!("{}", w.paint());
        }
    }
}
```

`Canvas::repaint` does not know whether each `w` is a slider, a label, or a widget invented in a crate that did not exist when `Canvas` was compiled; it only knows the value answers `paint`. The generic alternative `struct Canvas<T: Render> { widgets: Vec<T> }` pins one `Canvas` to a *single* widget type — all sliders or all labels — because monomorphisation must choose one `T`. The `dyn` version is strictly more flexible at the cost of the indirection. A downstream crate adds a type the framework never anticipated:

```rust
struct Gauge { value: f64 }

impl Render for Gauge {
    fn paint(&self) -> String { format!("[gauge {:.0}%]", self.value * 100.0) }
}

let c = Canvas {
    widgets: vec![
        Box::new(Gauge { value: 0.7 }),
        Box::new(Gauge { value: 0.2 }),
    ],
};
c.repaint();
```

This is structurally duck typing — "if it answers `paint`, it goes in the list" — but Rust checks the duck statically: no runtime `respondsTo:`, no `NoSuchMethodError`. If you've leaned on Python's duck typing, this is the same instinct ("I only care that it has a `paint` method") with the check moved to compile time. The coercion `Box::new(Gauge{..}) as Box<dyn Render>` only typechecks because `Gauge: Render`, and that is the *only* place the check happens.

> **⚠️ Pitfall →** Put a `String` into that `Vec<Box<dyn Render>>` and the compiler rejects it at the insertion site: `error[E0277]: the trait bound String: Render is not satisfied`, with a note that it is required for the cast from `Box<String>` to `Box<dyn Render>`. The error names the coercion site, not the eventual `.paint()` call — the unsizing coercion from `Box<String>` to `Box<dyn Render>` is exactly where the proof that `String: Render` must be produced, and it cannot be. The fix is either to store a type that implements the trait, or to `impl Render for String`.

## Object safety, now called *dyn compatibility*

Not every trait can become a `dyn Trait`. To build the vtable above, each method must be callable knowing only `&self` (a thin reference to unknown-but-`Sized` data) plus the vtable. Methods that violate this are excluded, and a trait is **dyn-compatible** (the modern name for "object-safe") only if *every* method is dispatchable. The two rules you will hit:

**1. No generic methods.** A generic method would need a vtable slot *per monomorphisation*, but the set of instantiations is open-ended — you cannot lay out a finite table. The compiler is explicit:

```text
error[E0038]: the trait `Container` is not dyn compatible
  ...because method `store` has generic type parameters
  = help: consider moving `store` to another trait
```

**2. No `Self` by value or in return position** (`fn duplicate(&self) -> Self`). The caller holding a `&dyn Trait` has erased the concrete type, so it cannot name `Self` to receive a returned `Self` by value, nor allocate space for it:

```text
error[E0038]: the trait `Cloneable` is not dyn compatible
  ...because method `duplicate` references the `Self` type in its return type
```

(Other disqualifiers exist — associated constants, statics — but generic params and bare `Self` are the two you trip over weekly.) The escape hatch is to gate the offending method so it *vanishes from the vtable*:

```rust
trait Widget {
    fn paint(&self) -> String;                  // in the vtable: dispatchable
    fn clone_box(&self) -> Box<dyn Widget>;      // OK: return type is not `Self`
    fn debug_dump(&self) where Self: Sized { }   // excluded from vtable, callable only statically
}
```

`where Self: Sized` tells the compiler "this method does not exist for the unsized `dyn Widget`," so it does not block vtable construction; you can still call it on a concrete `Button`. Note `clone_box` returns a *boxed trait object*, not `Self` — the idiomatic way to "clone a trait object" precisely because returning `Self` is forbidden.

> **🎓 Tripos link →** Think of this as a plain representability constraint (the intuition behind type-checking in Semantics of Programming Languages, stated without the formalism). A vtable is a finite list of function addresses, and each entry has to be *one* address that works over a thin `self`. A generic method would need a different address for every type you might instantiate it with — infinitely many, so there's no single slot to put. A method returning `Self` by value would need the caller to set aside room for a result whose size it can't know once the concrete type is erased. Both have no uniform representation, so the compiler refuses to build the table at all rather than letting you make a call the machine couldn't carry out. The guarantee you get back: a `&dyn Trait` *always* has a valid vtable entry for every method it exposes.

## Is Rust object-oriented?

Score it against the usual triad:

- **Objects (data + behaviour):** yes. `struct`/`enum` hold data; `impl` blocks attach methods. The Gang-of-Four definition is satisfied even though the community avoids the word "object" — deliberately, because Rust keeps data (`struct`) and behaviour (`impl`, `trait`) *syntactically separate*.
- **Encapsulation:** yes, via `pub`. Fields default to private; you expose an API and refactor internals behind it (swap a `Vec` field for a `HashSet`) without breaking callers. See [project structure](19-project-structure.md) for module-level visibility.
- **Inheritance:** **no.** There is no way to make one `struct` inherit another's fields and method bodies — a deliberate omission.

So Rust gives you two of three, and the third was rejected on purpose.

### Why inheritance was rejected: composition + traits instead

Inheritance bundles two separable things: (a) *implementation reuse* — sharing method bodies — and (b) *subtype polymorphism* — using a child where a parent is expected. Rust unbundles them.

- For **implementation reuse**: **default trait methods** (a method with a body that implementors inherit unless they override) and **composition** (embed a field, delegate to it). This sidesteps the *fragile base class problem*: in a deep hierarchy a change to a base method silently alters every descendant, possibly violating an invariant a subclass relied on. Traits have no implicit `super` call chain and no protected mutable state shared down a hierarchy, so the failure mode does not exist.
- For **subtype polymorphism**: **trait bounds** (`T: Draw`, static) or **trait objects** (`dyn Draw`, dynamic). You get substitutability without forcing the substituted types into a shared ancestry.

The deeper objection is that inheritance over-shares: a subclass inherits *all* of the parent, including methods that may not make sense for it (the classic `Square extends Rectangle` with independent `setWidth`/`setHeight`). Traits let a type opt into exactly the capabilities it has, with no "single inheritance" restriction.

> **🦀 From your toolbox →** A Java `interface` is the honest analogue of a Rust `trait`, and `class C implements I1, I2` maps cleanly to `impl I1 for C` / `impl I2 for C`. Swift's `protocol` plays the same role — `struct C: P1, P2` becomes `impl P1 for C` / `impl P2 for C` — and Swift even has protocol extensions with default method bodies, exactly like a trait's default methods. The break: Java still has `extends` for classes and Swift has class inheritance, and Rust has *nothing* like it — no `super`, no protected fields, no abstract base class carrying state. Where Java or Swift would use an abstract base class to share both state and behaviour, Rust forces you to split it: shared *behaviour* goes in a trait's default methods; shared *state* goes in a struct you embed by composition and delegate to. (Light C++ touch: there is also no `: public Base` and so no diamond problem — there's no implementation inheritance to form a diamond in the first place.)

## One pattern, two ways: the state machine

The sharpest way to feel the difference is to implement the same FSM twice: a blog post with states *Draft → PendingReview → Published*, where only a published post yields its content and illegal transitions are no-ops.

### The trait-object way (the OOP transcription)

States become structs implementing a private `State` trait; the post holds one as a boxed trait object. Each state owns its own transition rules:

```rust
pub struct Post {
    state: Option<Box<dyn State>>,
    body: String,
}

impl Post {
    pub fn new() -> Post {
        Post { state: Some(Box::new(Draft {})), body: String::new() }
    }
    pub fn add_text(&mut self, t: &str) { self.body.push_str(t); }

    pub fn content(&self) -> &str {
        self.state.as_ref().unwrap().content(self)
    }
    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review());
        }
    }
    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve());
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
    fn approve(self: Box<Self>) -> Box<dyn State>;
    fn content<'a>(&self, _post: &'a Post) -> &'a str { "" } // default
}

struct Draft {}
impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<dyn State> { Box::new(PendingReview {}) }
    fn approve(self: Box<Self>) -> Box<dyn State> { self } // no-op
}

struct PendingReview {}
impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<dyn State> { self }
    fn approve(self: Box<Self>) -> Box<dyn State> { Box::new(Published {}) }
}

struct Published {}
impl State for Published {
    fn request_review(self: Box<Self>) -> Box<dyn State> { self }
    fn approve(self: Box<Self>) -> Box<dyn State> { self }
    fn content<'a>(&self, post: &'a Post) -> &'a str { &post.body }
}
```

Three things here are not optional:

1. **The `Option<Box<dyn State>>` and `.take()`.** The transition methods take `self: Box<Self>` — they *consume* the old state to produce the new (you cannot transition by `&mut`, since turning a `Draft` into a `PendingReview` is a move that invalidates the old value). But `request_review(&mut self)` only borrows the post; you cannot move `self.state` out of a borrowed field and leave a hole. `Option::take` swaps in `None`, hands you the owned `Box`, and you write the result back — the ownership system forcing the "temporarily no state" window to be explicit, a window other languages leave implicit and bug-prone.

2. **`self: Box<Self>`** is the *arbitrary self type* that makes the consuming transition dyn-compatible: the receiver is the boxed trait object itself, so `take`-then-call moves the box into the method.

3. **`content` is a default method, and the duplication can't be removed.** You cannot give `request_review`/`approve` a default body returning `self`: through a `dyn State` the concrete `Self` is unknown, so a returned `self` has no statically known size — *a dyn-compatibility violation*, the previous section's rule biting in practice.

The payoff is the OOP one: `Post`'s methods contain no `match` on the state; behaviour lives in the state structs, and adding a `Scheduled` state means adding a struct and its `impl`, touching `Post` not at all. The cost: illegal use is caught at *runtime* (`content` on a draft returns `""` — a silent wrong answer, not a compile error), and the states are coupled (each knows its successor).

### The typestate way (the punchline)

Now encode each state as a *distinct type*, and make transitions consume one type and return the next. Methods that are illegal in a state simply *do not exist* on that type, so misuse fails to compile:

```rust
pub struct DraftPost { body: String }
pub struct PendingReviewPost { body: String }
pub struct Post { body: String }   // the *published* post

impl DraftPost {
    pub fn new() -> DraftPost { DraftPost { body: String::new() } }
    pub fn add_text(&mut self, t: &str) { self.body.push_str(t); }
    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost { body: self.body }
    }
    // no `content` method — a draft has no readable content, by construction
}

impl PendingReviewPost {
    pub fn approve(self) -> Post { Post { body: self.body } }
    // no `content`, no `add_text` — both meaningless while under review
}

impl Post {
    pub fn content(&self) -> &str { &self.body }   // only the published type has this
}
```

Usage threads the value through the type-changing transitions; `let` shadowing rebinds because each transition returns a *new type*:

```rust
let mut post = DraftPost::new();
post.add_text("shipped the borrow checker");
let post = post.request_review();   // DraftPost  -> PendingReviewPost
let post = post.approve();          // PendingReviewPost -> Post
assert_eq!("shipped the borrow checker", post.content());
```

Try `DraftPost::new().content()` and the compiler refuses with the exact diagnostic `error[E0599]: no method named `content` found for struct `DraftPost`` … `the method was found for `Post``. The illegal transition is not a runtime no-op; **it is not expressible.** Because the transition methods take `self` by value, the old state is *moved out and consumed* — no lingering `DraftPost` survives `request_review` to be misused. The state machine has been lifted into the type system; the borrow checker is now your model checker.

> **⚙️ Under the hood →** Typestate is zero-cost. The three structs have identical layout; `request_review` is a move of the `String` (often a no-op reinterpretation, since the layouts match) and produces no vtable, no heap box, no `Option` dance. The trait-object version, by contrast, heap-allocates a fresh `Box<dyn State>` on every transition and pays an indirection on every `content` call. Typestate trades runtime checks and allocations for *compile-time* state tracking — you pay nothing at runtime for a guarantee the OOP version enforces only dynamically, and partially.

> **🔧 In practice →** Typestate shines on resource and protocol APIs where calling things out of order is a real bug. Picture a TCP-style connection that must be opened, then sent on, then closed — and where you must never write to a closed socket. Encode each stage as its own type:
> ```rust
> struct Closed;
> struct Open { fd: i32 }
>
> impl Closed {
>     fn connect(self, addr: &str) -> Open { /* ... */ Open { fd: 3 } }
> }
> impl Open {
>     fn send(&mut self, bytes: &[u8]) { /* ... */ }
>     fn close(self) -> Closed { /* ... */ Closed }   // consumes self
> }
> ```
> Once you call `close`, the `Open` value is *moved away* and gone — `send` on it won't even compile, so "write after close" is impossible to express. Builder APIs use the same trick to enforce "you must set the URL before calling `.build()`". This is the high-value reason to reach for typestate: turning a class of runtime misuse into a compile error, with zero runtime cost.

> **🎓 Tripos link →** Foundations of Computer Science (the OCaml course) gives the intuition: there, the only way to obtain a value of a given type is to build it with one of that type's constructors. Here it's the same discipline, enforced by privacy. The only route to a `Post` (the published one) is the chain `DraftPost → PendingReviewPost → Post`, because the constructors are private and the transition methods are the only doors between the types. Holding a `PendingReviewPost` is itself evidence that a review was requested — you couldn't have got one any other way. So the reachable states of the FSM become exactly the types you can construct, and the legal transitions become the functions between them; an illegal transition is simply a call that doesn't typecheck. The same idea generalises to a `PhantomData<State>` marker when you want one struct parameterised by a state type rather than three distinct structs.

The lesson is not "never use trait objects." It is: **when the legal transitions are known at compile time and you want misuse to be a compile error, prefer typestate; reach for `dyn` when the set of types is open or genuinely runtime-chosen.** The trait-object state pattern earns its keep when downstream code must add new states the library cannot foresee — the same extensibility argument as `Canvas`.

## The decision guide: `impl Trait` vs `dyn Trait` vs generic bound

Three syntaxes return-or-accept "something that implements `Draw`," and beginners conflate them. They are distinct:

```rust
fn make_one() -> impl Render { Gauge { value: 0.5 } }     // one hidden concrete type
fn make_boxed() -> Box<dyn Render> { Box::new(Gauge { value: 0.5 }) } // any, erased
fn take_generic<T: Render>(x: T) { /* monomorphised per T */ }
fn take_dyn(x: &dyn Render) { /* one body, indirect */ }
```

- **`-> impl Trait` (return position):** "I return *exactly one* concrete type, but I'm not naming it." Static dispatch, no allocation, no indirection, fully inlinable; the caller gets an opaque type usable only via the trait. Use it to return closures and iterators (whose types are unnameable) or to hide an implementation type. You **cannot** return two different types from two branches — `impl Trait` is one concrete type.
- **`Box<dyn Trait>` (return or storage):** "I return one of *several* types chosen at runtime," or "I store a heterogeneous collection." Dynamic dispatch, one heap allocation, the indirection. The only one of the three that lets a function return different concrete types from different `match` arms.
- **`fn f<T: Trait>(x: T)` / `impl Trait` in argument position:** static dispatch at the call site, monomorphised. The best default for accepting parameters — maximal performance, caller's concrete type preserved.
- **`fn f(x: &dyn Trait)`:** dynamic dispatch, one compiled body. Use when you need argument heterogeneity or want to curb monomorphisation bloat in a large function.

> **🦀 From your toolbox →** `-> impl Trait` is "the caller sees an abstract type it can only use through the trait" — like a Java method declared to return an `interface` type, except Rust pins it to *one* hidden concrete type and inlines it (no boxing). `Box<dyn Trait>` is the one that genuinely behaves like a Java interface-typed reference or a Swift class reference held through a protocol: a heap value reached by indirection, where the concrete type is erased and dispatch goes through a table. `fn f<T: Trait>` is closest to Java generics — but where Java erases `T` to one bytecode body, Rust monomorphises into a separate stamped-out copy per type (Compiler Construction). One nicety over many template-style systems: Rust typechecks the *generic body* against the bound `T: Trait` once, at the definition ([generics and traits](08-generics-and-traits.md)), so you get a clean error where the function is written rather than a wall of errors at each call site.

## Mental-model recap

- `dyn Trait` is an unsized type; you only ever hold it behind a pointer, and that pointer is **fat** — `(data ptr, vtable ptr)`, two words. The vtable is per-(type, trait), lives in static memory, and is *not* embedded in the value (unlike a Java object's class pointer or a C++ vptr), so concrete values stay thin and one value can be viewed through many independent vtables.
- **Static dispatch** (generics, `impl Trait`) is monomorphised: N copies, direct inlinable calls, no runtime cost. **Dynamic dispatch** (`dyn`) is one body plus a vtable indirection: thin binary, runtime heterogeneity, no inlining. Choose `dyn` only when you need runtime mixing or an open extensibility boundary.
- **Dyn-compatibility** (object safety) excludes generic methods and by-value/return `Self`, because a vtable slot must be a single function pointer over a thin `self`. Gate convenience methods with `where Self: Sized` to keep a trait usable as `dyn`.
- Rust has encapsulation and polymorphism but **no inheritance**: implementation reuse comes from default trait methods + composition; subtype polymorphism from trait bounds or trait objects. This dodges the fragile-base-class problem by design.
- The **typestate** pattern encodes each FSM state as a type, with transitions consuming `self` and returning the next state's type; illegal transitions become non-existent methods, so misuse fails to compile. It is zero-cost. Prefer it over the trait-object state pattern when the states are closed and you want compile-time enforcement.

## Exercises

1. **Predict the coercion.** You have `let v: Vec<Box<dyn Render>> = vec![Box::new(Gauge { value: 0.5 })];` and now you try to read `v[0].value` directly. Before compiling, say what happens and why — then say which single line you'd add to the `Render` trait to make the gauge's percentage reachable through the trait object. (The point: a `dyn Render` exposes the *trait's* surface, not the concrete type's fields.)

2. **Which compiles, and why?** Three traits, each used as `&dyn T`. Without running the compiler, decide for each whether it is dyn-compatible, and name the rule:
   ```rust
   trait A { fn id(&self) -> u32; }
   trait B { fn merge(&self, other: &Self) -> Self; }
   trait C { fn tag<T: Into<String>>(&self, t: T) -> String; }
   ```
   For the two that fail, propose the smallest signature change that keeps the method *useful on concrete types* while letting the trait still be used as `&dyn T`.

3. **Choose the tool.** For each scenario, pick `-> impl Trait`, `Box<dyn Trait>`, or a generic bound `<T: Trait>`, and justify in one sentence: (a) a function that returns the iterator `data.iter().filter(...).map(...)`; (b) a `parse` function that returns one of three different error-formatter types depending on a runtime flag; (c) a hot inner routine called only with `Vec<f64>` that you want fully inlined.

4. (★) **Strengthen the blog state machine.** Take the *trait-object* `Post` and try to add a `reject(&mut self)` transition (PendingReview → Draft) by giving `State` a *default* `fn reject(self: Box<Self>) -> Box<dyn State> { self }`. Predict the compiler's reaction, name the dyn-compatibility rule involved, and then explain how the *typestate* version would model `reject` instead — and what misuse it would catch at compile time that the trait-object version cannot.

5. **Cost reasoning, no code to write.** A teammate makes every function in a math-heavy module take `&dyn Add`-style trait objects "to keep the binary small." On a tight numeric loop, explain concretely what runtime costs this introduces that a generic bound would avoid, and name one situation where their instinct is actually the right call.
