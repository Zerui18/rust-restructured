# 12. Closures & Iterators

Java gave you lambdas and `Stream.map`; Swift gave you closures and `.map`/`.filter` on arrays; Python gave you `lambda`, comprehensions, and generators. So the *vocabulary* of this chapter is familiar. What is new is that Rust threads its entire ownership discipline through both features, statically and at zero runtime cost. A closure is not a heap-allocated object pointing at a garbage-collected environment, as in Java or Python. It is an anonymous, compiler-generated `struct` whose fields *are* the captured variables, and whose "callability" is expressed through one of three traits. An iterator chain is not a sequence of allocating intermediate collections, as a naive Python `map`-then-`filter` over lists would be. It is a tower of nested structs that the compiler monomorphises and inlines into a single loop with no allocation and no indirection.

The thesis of this chapter: *Rust makes capture mode part of the type, and makes iteration a trait with one required method.* Everything else — the `Fn`/`FnMut`/`FnOnce` hierarchy, laziness, the zero-cost claim — follows from those two design decisions.

## Closures are anonymous structs

A closure literal `|params| body` desugars to an anonymous `struct` whose fields are exactly the environment variables the body mentions, plus an `impl` of a call trait. Consider:

```rust
let factor = 3;
let scale = |x: i32| x * factor;
```

`rustc` synthesises roughly this:

```rust
struct __Scale<'a> { factor: &'a i32 }   // a field per captured variable
impl<'a> Fn(i32) -> i32 for __Scale<'a> { /* body: x * *self.factor */ }
let scale = __Scale { factor: &factor };
```

(The trait syntax is illustrative; you cannot hand-write `impl Fn` on stable.) The crucial consequence is that **every closure has a distinct, unnameable type**. Two closures with identical signatures and bodies are different types, exactly as two distinct `struct`s would be. This is why you cannot write `let f: ??? = |x| x + 1;` with a concrete named type — there isn't one — and why closures are passed via generics (`F: Fn(...)`) or trait objects (`Box<dyn Fn(...)>`).

> **🦀 From your toolbox →** In Swift, a closure captures variables from its surrounding scope much like Rust's does, and you can store one in a variable typed `(Int) -> Int`. The difference: Swift gives you a single nameable function type, and its closures live behind ARC (automatic reference counting), so capture has a runtime cost and the lifetime is managed for you. Rust instead generates a *fresh, unnameable type* per closure and decides capture mode at compile time, with no reference counting. Java's lambdas are similar — each is an anonymous object implementing a functional interface — but the JVM's garbage collector keeps captured objects alive for as long as the lambda references them, so capture is never a lifetime question. Rust has no GC and no ARC, so capture mode is load-bearing: it is exactly *how* the captured data enters the closure and *when* it is freed. (Where the analogy breaks down: in Java and Swift a captured local is reachable through a reference and the runtime keeps it alive; in Rust a captured borrow must be *proven* to outlive the closure, or it does not compile.)

### Type inference and the locked-in signature

Unlike `fn` items, closures usually need no type annotations. A `fn` exposes a public interface, so its types are mandatory; a closure is a local implementation detail, so the compiler infers parameter and return types from the body and the first call site. But inference picks *one* concrete type and freezes it:

```rust
let identity = |x| x;
let _a = identity(String::from("hi")); // x, return both inferred as String
// let _b = identity(42);              // ERROR: expected String, found integer
```

```text
error[E0308]: mismatched types
  |
  |     let _b = identity(42);
  |              -------- ^^ expected `String`, found integer
note: expected because the closure was earlier called with an argument of type `String`
```

This is not generics. A closure is not polymorphic over its argument types; inference simply defers the choice to the first use and then commits. If you want polymorphism, write a generic `fn`.

## The Fn / FnMut / FnOnce hierarchy

How a closure interacts with its environment determines *which* call traits it implements. There are three, and they form a hierarchy by capability:

| Trait | Method signature | Captures used as | Callable |
|-------|------------------|------------------|----------|
| `FnOnce` | `fn call_once(self, …)` | moves something *out* | exactly once |
| `FnMut` | `fn call_mut(&mut self, …)` | mutates captures | many times |
| `Fn` | `fn call(&self, …)` | reads captures (or captures nothing) | many times, concurrently |

The defining detail is the receiver. `FnOnce::call_once` takes `self` *by value*, so calling it consumes the closure struct — that is the type-level encoding of "callable at most once." `FnMut::call_mut` takes `&mut self`, so it needs exclusive access for each call. `Fn::call` takes `&self`, so it can be called through a shared reference, even from multiple threads.

These are *supertraits* in the capability sense: `Fn: FnMut: FnOnce`. Anything you can call with `&self` you can call with `&mut self` (just ignore the extra power), and anything callable repeatedly is callable once. So **every closure implements `FnOnce`**; closures that don't move-out also implement `FnMut`; closures that don't mutate also implement `Fn`. The compiler implements the *largest* applicable set automatically.

> **🦀 From your toolbox →** This is the trait-bound machinery from [generics and traits](08-generics-and-traits.md), applied to compiler-generated types. Think of Java's single-method interfaces: `Comparator<T>` has one method, and a lambda satisfies it. An `FnMut(&T) -> K` bound is the same idea — a one-method "interface" — except the implementer is an anonymous struct chosen at compile time rather than an object dispatched through the JVM. The receiver type (`self` vs `&mut self` vs `&self`) is the clever part: because `FnOnce::call_once` takes the closure *by value*, calling it uses the closure up, so a second call simply won't type-check. "At most once" is enforced by the type system, not by a runtime flag you have to remember to check — closer in spirit to a value you can hand over exactly once, after which you no longer hold it.

### Capture mode is chosen by the body

The compiler picks the *least* powerful capture for each variable that still lets the body compile: shared `&` if the body only reads, `&mut` if it mutates, by-value move if it moves the value out or needs ownership.

```rust
let data = vec![1, 2, 3];
let read = || println!("{data:?}");   // captures &data  -> Fn

let mut log = Vec::new();
let mut record = |x| log.push(x);     // captures &mut log -> FnMut

let owned = String::from("gone");
let consume = move || drop(owned);    // moves owned in, drops it -> FnOnce
```

Note `read` borrows `data` immutably, so `data` remains usable elsewhere until the closure's last use. `record` borrows `log` mutably, so — by the [borrowing](03-references-and-borrowing.md) rules — nothing else may touch `log` while `record` is live, and `record` itself must be `mut` (calling it needs `&mut record`).

> **⚠️ Pitfall →** A closure that mutates a capture must be bound to a `mut` variable, and the captured variable is exclusively borrowed for the closure's whole lifetime:
> ```rust
> let mut total = 0;
> let mut add = |x| total += x;
> // println!("{total}");   // ERROR: cannot borrow `total` as immutable
> add(5);                   //        because it is also borrowed as mutable
> println!("{total}");      // OK: add's borrow ended after its last use
> ```
> ```text
> error[E0502]: cannot borrow `total` as immutable because it is also
>               borrowed as mutable
> ```
> The fix is to not interleave: finish using the closure before touching the captured variable directly. NLL (non-lexical lifetimes) ends `add`'s borrow at its last call, so moving the `println!` *after* the last `add(...)` compiles.

### The `move` keyword

`move` forces every capture to be by value, overriding the read/mutate inference. The body's behaviour is unchanged; only *how variables enter the closure struct* changes — borrows become moves (or copies, for `Copy` types). This does **not** by itself make the closure `FnOnce`: a `move` closure that only *reads* its now-owned captures is still `Fn`. What determines `FnOnce` is moving a value *out* in the body, not moving it *in* via `move`.

```rust
let label = String::from("ping");
let f = move || println!("{label}"); // moves label IN, but only reads it
f(); f();                            // still Fn: callable repeatedly
```

The canonical use is sending data to another thread, where the closure must own its captures because the spawning frame may exit first:

```rust
use std::thread;
let nums = vec![10, 20, 30];
thread::spawn(move || println!("worker sees {nums:?}"))
    .join()
    .unwrap();
// nums is no longer accessible here — it was moved into the thread
```

> **🔧 In practice →** You're writing an HTTP handler that fires off a background task — logging the request to a slow audit store without blocking the response. The data the task needs (a request id, a user id) lives in the handler's stack frame, which disappears the instant you return the response. `move` is what lets the spawned closure *own* a copy of that data so it stays valid after the handler returns:
> ```rust
> let request_id = req.id.clone();
> let user = req.user.clone();
> std::thread::spawn(move || {
>     // request_id and user are owned by this closure now,
>     // so they outlive the handler that spawned the task
>     audit_log.write(request_id, user);
> });
> // handler returns here; the background thread keeps running safely
> ```
> Drop the `move` and the compiler refuses: the closure would borrow `request_id` from a frame that is about to vanish.

> **⚙️ Under the hood →** Without `move`, the thread closure would capture `&nums`, a reference into `main`'s stack frame. `thread::spawn` requires `F: Send + 'static`; a borrow of a local is not `'static`, so it fails to compile — the borrow checker proving statically what would otherwise be a use-after-free when `main`'s frame is popped while the worker still runs. We return to `Send`/`'static` in [fearless concurrency](13-fearless-concurrency.md).

### Bounds in practice: `unwrap_or_else` vs `sort_by_key`

A function that takes a closure picks the *weakest* bound that suffices, to accept the most closures. `Option::unwrap_or_else` calls its closure at most once, so it asks for `FnOnce`:

```rust
pub fn unwrap_or_else<F: FnOnce() -> T>(self, f: F) -> T {
    match self { Some(x) => x, None => f() }
}
```

Because the bound is `FnOnce` (the most permissive — *every* closure qualifies), you can pass anything, including a closure that moves captures out. By contrast `slice::sort_by_key` calls its key function once per element, so it requires `FnMut`. A closure that moves a non-`Copy` value out of its environment is `FnOnce` only, and fails the `FnMut` bound:

```rust
let mut sizes = [3u32, 1, 2];
let mut calls = 0;
sizes.sort_by_key(|&n| { calls += 1; n }); // FnMut: mutates `calls`, fine
```

Replacing `calls += 1` with `log.push(owned_string)` (moving an owned `String` out each call) yields:

```text
error[E0507]: cannot move out of `owned_string`, a captured variable
              in an `FnMut` closure
```

The fix the compiler wants is to mutate-in-place (a counter) or `clone()`, never move-out, because move-out is incompatible with being called more than once.

### Returning closures

A returned closure has an unnameable type, so you return it behind `impl Fn` (static, monomorphised, zero-overhead) or `Box<dyn Fn>` (dynamic, one heap allocation + vtable):

```rust
fn adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n                    // `move` required: n must outlive the frame
}

fn dispatch(neg: bool) -> Box<dyn Fn(i32) -> i32> {
    if neg { Box::new(|x| -x) } else { Box::new(|x| x) }
}
```

Use `impl Fn` when one concrete closure type is returned on all paths; the compiler still knows the exact type and can inline. Use `Box<dyn Fn>` when different paths return *different* closure types (as in `dispatch`) — they must be unified behind a trait object, which means dynamic dispatch through a vtable, exactly the mechanism in [trait objects](09-trait-objects-and-oop.md). `move` is almost always required on a returned closure: any captured local would otherwise be a dangling reference once the function returns.

> **🔧 In practice →** You're building a configurable filter for a search feature: the user picks a minimum price, and you want a reusable predicate to apply across thousands of items. A factory function that bakes the threshold into a returned closure is exactly the tool:
> ```rust
> fn min_price_filter(threshold: u32) -> impl Fn(&Item) -> bool {
>     move |item| item.price >= threshold   // threshold is captured by value
> }
>
> let keep = min_price_filter(user_min);
> let results: Vec<_> = catalog.iter().filter(|it| keep(it)).collect();
> ```
> Return `impl Fn` here, not `Box<dyn Fn>`: there's only one closure type, so the compiler keeps it concrete and inlines the call into the `filter` loop with zero overhead. Reach for `Box<dyn Fn>` only when different branches return genuinely different closures that must share one return type.

> **🦀 From your toolbox →** `Box<dyn Fn(i32) -> i32>` is the moral equivalent of Java storing a lambda in a `Function<Integer, Integer>` field, or Swift storing one in a `(Int) -> Int` property: the concrete callable is hidden behind a single reference type and called indirectly. `impl Fn` has no direct analogue in those languages — it is closer to "this function returns *some specific* function type that I'm not naming, but the compiler knows exactly which one," so it can be inlined. In Java and Swift, every stored function is effectively the boxed, dynamically-dispatched form; Rust gives you a static option that costs nothing.

## The Iterator trait: one method to rule them all

An iterator is any type implementing `Iterator`, whose *entire* required surface is one method:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // ~75 provided methods, all default-implemented in terms of next()
}
```

`Item` is an *associated type* (see [advanced traits](17-advanced-traits-and-types.md)): the element type is a property of the iterator type, not a free generic parameter, so a given iterator yields exactly one element type. `next` takes `&mut self` because pulling an element advances internal state (a cursor); it returns `Some(item)` until exhaustion, then `None` forever after. That is the whole protocol. Every other method — `map`, `filter`, `sum`, `collect`, `fold` — is a *default method* built on `next`. Implement `next` and you inherit all of them.

> **🦀 From your toolbox →** This is Python's generator protocol made into a trait. A Python generator is an object whose `__next__()` either returns the next value or raises `StopIteration`; Rust's `next()` returns `Some(item)` until exhausted, then `None`. Both are *lazy* — nothing is computed until you pull the next element. Java's `Iterator` interface is the closer structural match (`hasNext()` + `next()`), but Rust folds both checks into a single call returning `Option`, so there's no way to ask "is there more?" without also taking the element. The Swift analogue is `IteratorProtocol`, which likewise has one method returning an optional `next()`.

A `for` loop is sugar over this trait. `for x in it { … }` desugars to taking `it` by value (via `IntoIterator`), making it `mut`, and looping `while let Some(x) = it.next() { … }`. That is why you never write the index arithmetic and never make the iterator `mut` yourself.

### IntoIterator and the three iteration forms

`for` does not call `Iterator` directly; it calls `IntoIterator::into_iter`, which converts a collection into an iterator. Collections offer three conversions, distinguished by what they yield and what they do to the source:

```rust
let v = vec![1, 2, 3];

for x in v.iter()      { /* x: &i32     */ }  // borrows v immutably
for x in v.iter_mut()  { /* x: &mut i32 */ }  // borrows v mutably (v must be mut)
for x in v.into_iter() { /* x: i32      */ }  // consumes v; v unusable after
```

These are exactly the three borrow modes of [references and borrowing](03-references-and-borrowing.md), surfaced as method names. The `for` loop's implicit `IntoIterator` call is overloaded by reference type: `for x in &v` calls `(&v).into_iter()` ≡ `v.iter()`; `for x in &mut v` ≡ `v.iter_mut()`; `for x in v` ≡ `v.into_iter()` and moves `v`. Choosing the wrong one is a common borrow-checker lesson:

> **⚠️ Pitfall →** `for x in v { … }` moves `v`; using `v` afterwards fails:
> ```rust
> let v = vec![1, 2, 3];
> for x in v { println!("{x}"); }
> // println!("{v:?}");   // ERROR: borrow of moved value: `v`
> ```
> ```text
> error[E0382]: borrow of moved value: `v`
>   = note: `for` loop temporary takes ownership via IntoIterator
> ```
> Write `for x in &v` (or `v.iter()`) to iterate by reference and keep `v`. The compiler's note even tells you the `for` loop consumed it via `IntoIterator`.

### Lazy adapters vs eager consumers

Iterator methods split cleanly in two. **Adapters** return a new iterator and do nothing until pulled — they are lazy. **Consumers** call `next` in a loop and produce a final value, exhausting the iterator.

Adapters wrap the source iterator in another struct. `v.iter().map(f)` returns a `Map<Iter<'_, i32>, F>` — it has not called `f` on anything. Forgetting to consume is a compile-time *warning*, because building an adapter and dropping it is almost always a bug:

```rust
let v = vec![1, 2, 3];
v.iter().map(|x| x + 1);   // warning: unused `Map` that must be used
                           // note: iterators are lazy and do nothing unless consumed
```

The workhorse lazy adapters:

```rust
(1..=10)
    .filter(|n| n % 2 == 0)      // keep evens:        2 4 6 8 10
    .map(|n| n * n)              // square:            4 16 36 64 100
    .take(3)                     // first 3:           4 16 36
    .enumerate()                 // pair with index:   (0,4) (1,16) (2,36)
    .for_each(|(i, sq)| println!("#{i}: {sq}"));
```

- `map` / `filter` — transform / select (closure decides).
- `take(n)` / `skip(n)` — prefix / suffix; `take` makes infinite iterators usable.
- `enumerate` — yields `(index, item)`, replacing manual index counters.
- `zip(other)` — yields `(a, b)` pairs, stopping when *either* side ends.
- `chain(other)` — concatenate two iterators of the same `Item`.
- `flat_map(f)` — map each item to an iterator, then flatten the results into one stream.

Consumers (eager — they run the pipeline):

```rust
let words = ["alpha", "beta", "gamma"];

let total: usize     = words.iter().map(|w| w.len()).sum();          // 14
let joined: String   = words.iter().flat_map(|w| w.chars()).collect(); // "alphabetagamma"
let any_long: bool   = words.iter().any(|w| w.len() > 4);            // true (short-circuits)
let first_g          = words.iter().find(|w| w.starts_with('g'));    // Some(&"gamma")
let product: usize   = (1..=5).fold(1, |acc, n| acc * n);            // 120
```

- `collect` — gather into a collection; the target type drives behaviour (see below).
- `sum` / `product` / `count` — reduce to a scalar.
- `fold(init, f)` — the general left-fold: start from `init` and combine each element into a running accumulator; `sum` is `fold(0, +)`. (This is Python's `functools.reduce` with an explicit initial value, or Java's `Stream.reduce`.)
- `for_each` — run a closure for its side effects (the consuming cousin of `map`).
- `find` / `position` / `any` / `all` — searches that *short-circuit*: they stop pulling `next` the moment the answer is known.

`collect` is the one to dwell on. It is generic over the target via the `FromIterator` trait, so you choose the result type by annotation, and `collect` is polymorphic in its return:

```rust
let evens: Vec<i32>      = (1..=6).filter(|n| n % 2 == 0).collect();
let set:   std::collections::HashSet<i32> = [1, 1, 2, 3].into_iter().collect();
let map:   std::collections::HashMap<&str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
```

A particularly Rusty trick: `collect` over an iterator of `Result`s produces a single `Result<Vec<_>, E>`, short-circuiting on the first `Err`. This is the standard way to validate a batch — covered further in [error handling](11-error-handling.md):

```rust
let parsed: Result<Vec<i32>, _> = ["1", "2", "x", "4"].iter().map(|s| s.parse::<i32>()).collect();
// Err(ParseIntError) — stops at "x"
```

> **🔧 In practice →** You're parsing a CSV column of numbers from user-uploaded data, and you want to reject the whole upload if *any* cell is malformed — not silently skip bad rows. Annotate `collect` to target `Result<Vec<_>, _>` and it does exactly that, handing you the first parse error to surface back to the user:
> ```rust
> fn parse_amounts(rows: &[&str]) -> Result<Vec<i32>, std::num::ParseIntError> {
>     rows.iter().map(|cell| cell.trim().parse::<i32>()).collect()
> }
>
> match parse_amounts(&["10", "20", "oops", "40"]) {
>     Ok(amounts) => process(amounts),
>     Err(e) => return bad_request(format!("invalid amount: {e}")),
> }
> ```
> The same pipeline, collected into `Vec<i32>` instead, would force you to handle each `Result` element by hand. Choosing the target type *is* choosing the error-handling strategy.

> **🦀 From your toolbox →** Python's `[f(x) for x in xs if g(x)]` and Swift's `xs.map(f).filter(g)` both build a fresh list/array at each step and run immediately. Java's `Stream` is the close match to Rust's model: lazy adapters, a single terminal operation. Rust's adapters allocate *nothing* and run *nothing* until a consumer pulls — the chain is a description, not a computation. The break from Python and Swift: there is no intermediate array materialised between `map` and `filter`; the data flows one element at a time through the whole pipe, just as it does through a Java stream.

### Implementing Iterator for your own type

Implement `next`, get the whole library. Here is a bounded Fibonacci generator:

```rust
struct Fib { a: u64, b: u64, remaining: u32 }

impl Iterator for Fib {
    type Item = u64;
    fn next(&mut self) -> Option<u64> {
        if self.remaining == 0 { return None; }
        self.remaining -= 1;
        let next = self.a;
        self.a = self.b;
        self.b = next + self.b;
        Some(next)
    }
}

fn fib(n: u32) -> Fib { Fib { a: 0, b: 1, remaining: n } }

let sum_even: u64 = fib(20).filter(|n| n % 2 == 0).sum();
```

`fib(20)` is a value of our struct; because it is an `Iterator`, `.filter(...).sum()` work for free. We wrote `next` and inherited the entire adapter/consumer tower. Returning `None` once is the contract; a well-behaved iterator returns `None` forever after.

## The zero-cost claim

The marketing line is "zero-cost abstraction." Here is the precise mechanism. Each adapter is a generic struct parameterised by the previous iterator and the closure type. Because closure types are concrete and the generics are resolved by **monomorphisation** — the compiler stamps out a separate, specialised copy of the code for each concrete type combination, rather than using one shared erased version — `rustc` generates a specialised `next` for the *entire* chain with all closure bodies and all `next` calls statically known. There are no vtables, no `dyn` dispatch, no boxed-function indirection. The optimiser then **inlines** the nested `next` calls into one another, collapsing the tower into a single loop body. Bounds checks on slice access are elided where the index is provably in range, and loop unrolling applies as it would to a hand-written loop.

```rust
// These compile to essentially identical machine code:
fn sum_loop(v: &[i32]) -> i32 {
    let mut s = 0;
    for i in 0..v.len() { s += v[i]; }   // explicit index, bounds-checked
    s
}
fn sum_iter(v: &[i32]) -> i32 {
    v.iter().sum()                       // no index, no manual bounds check
}
```

> **⚙️ Under the hood →** The `Map`/`Filter` structs are stack values holding the inner iterator and a closure (itself a zero-field struct if it captures nothing). After monomorphisation the call graph `Sum::sum → Filter::next → Map::next → Iter::next` is fully concrete; inlining flattens it. The standard-library benchmark — searching the full text of *Sherlock Holmes* for a word — shows the `for`-loop and iterator versions within noise of each other (~19.2M vs ~19.6M ns/iter). What you don't use you don't pay for; what you do use you couldn't hand-code better. The iterator form often gets *better* codegen than a manual index loop because `v.iter()` carries the no-out-of-bounds invariant in the type, letting the optimiser drop bounds checks the index version may keep.

> **🎓 Course link →** This is the classic *Compiler Construction* tradeoff between monomorphisation (the compiler stamps out a separate specialised copy of the code per concrete type) and dynamic dispatch (one shared copy that looks up the right code through a vtable at runtime, like Java's erased generics). For iterator chains Rust always picks monomorphisation. The cost is binary size — one specialised loop per distinct chain type — and the payoff is that the high-level pipeline and the low-level hand loop compile to the same machine code after inlining and bounds-check elimination. The abstraction genuinely vanishes once you turn the optimiser on.

## Idiom: pipelines replace index loops

Given the zero-cost guarantee, the community idiom is unambiguous: prefer the iterator pipeline over a `for`-with-index loop. It is no slower, it eliminates the off-by-one and out-of-bounds failure modes that index arithmetic invites, and it reads as a declarative description of the transformation. Reach for an explicit `for` loop only when the body has complex control flow (early `return`, `break` with a value, `continue` across multiple conditions) that does not map cleanly onto adapters.

```rust
// Imperative, index-driven — avoid:
let mut out = Vec::new();
for i in 0..items.len() {
    if items[i].active { out.push(items[i].id * 2); }
}

// Idiomatic — prefer:
let out: Vec<_> = items.iter()
    .filter(|it| it.active)
    .map(|it| it.id * 2)
    .collect();
```

## Mental-model recap

- A closure is an anonymous, unnameable `struct` whose fields are its captures; capture mode (`&`, `&mut`, by-value) is **inferred from the body** and checked by the borrow checker. `move` forces by-value capture but does not by itself imply `FnOnce`.
- The call traits encode callability in the receiver: `FnOnce` (`self`, moves out, once), `FnMut` (`&mut self`, mutates, many), `Fn` (`&self`, reads, many/concurrent). Every closure is `FnOnce`; the compiler adds `FnMut`/`Fn` when the body permits. Take the **weakest** bound your function can tolerate.
- `Iterator` requires only `next(&mut self) -> Option<Item>`; implement it and inherit ~75 default methods. `for` is sugar over `IntoIterator` + `while let Some = next()`.
- Adapters (`map`, `filter`, `take`, `zip`, `enumerate`, `chain`, `flat_map`) are **lazy** and allocate nothing; consumers (`collect`, `sum`, `fold`, `for_each`, `find`, `any`) are **eager** and may short-circuit. `collect`'s result type is driven by annotation via `FromIterator`.
- Monomorphisation + inlining make an iterator chain compile to the same (often better, due to bounds-check elision) assembly as a hand loop. Prefer pipelines.

## Exercises

1. Here are three closures, each passed to a function with a different bound. Predict which compile and which don't, and explain each in one sentence:
   - `(0..3).map(|_| s.clone()).collect::<Vec<_>>()` where `s: String` — does the `map` closure need to be `FnMut`, and does cloning satisfy it?
   - `let g = move || drop(buf); g(); g();` where `buf: Vec<u8>` — what bound does `g` have, and what does the second call report?
   - `option.unwrap_or_else(move || expensive)` where `expensive: String` is moved out — why does this compile even though the closure moves a value out of its environment?

2. You have `let words: Vec<String> = ...;` and want the total length of all words *over* 3 characters. Write it as an iterator pipeline. Then decide: should the pipeline start with `words.iter()`, `&words`, or `words.into_iter()` if you need to keep `words` usable afterward? Justify the choice in terms of what each yields and does to the source.

3. The chapter says a function should take the *weakest* bound it can tolerate. Suppose you're designing a `retry(times: u32, action: ...)` helper that calls `action` repeatedly until it succeeds. Which of `Fn`/`FnMut`/`FnOnce` should the `action` parameter require, and why would requiring `FnOnce` be wrong here while requiring `Fn` would be needlessly restrictive?

4. Take `let pairs = vec![("a", 1), ("b", 2), ("a", 3)];`. Using `collect`, build (a) a `HashMap<&str, i32>` and (b) a `Vec<(&str, i32)>` from the same pipeline by changing only the target type annotation. Predict what the `HashMap` contains for key `"a"` and explain why, then say in one sentence what general fact about `collect` this demonstrates.

5. (★) Consider `(2u64..).filter(|n| is_prime(*n)).map(|n| n * n).take(4)`. This chains a `filter` and a `map` over an *infinite* range and never loops forever at construction. Explain the pull mechanism that makes this terminate: which method actually drives the iteration, and in what order do `take`, `map`, and `filter` call `next` on each other for a single produced element?
