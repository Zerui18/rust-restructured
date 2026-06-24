# 2. Ownership & Moves

This is the chapter the rest of the book pivots on. Ownership is the single mechanism by which Rust statically guarantees memory safety and freedom from data races without a garbage collector and without manual `free`. Everything later — [references](03-references-and-borrowing.md), [lifetimes](04-lifetimes.md), [smart pointers](15-smart-pointers.md), [fearless concurrency](13-fearless-concurrency.md) — is a refinement of, or an escape valve around, the rules introduced here.

You already know two extreme answers to the question *"when does this allocation get freed?"*. The garbage-collected answer (Java, Python, JS): a runtime traces reachability and frees whenever it likes, at a cost in latency and determinism. The reference-counted answer (Swift's ARC, CPython's refcounting): the runtime keeps a count and frees when it hits zero, paying for the counting and the bookkeeping at run time. The manual answer (a C handle you must `free` by hand): you call `free` yourself, and the compiler does not check that you do it exactly once at the right moment. Rust takes a different position: the *type system* tracks who is responsible for each value, and the compiler inserts the deallocation for you at a statically known point. No tracing, no reference counts in the common case, no manual `free`, no leaks-by-default, and — crucially — it is all decided at compile time, so there is zero runtime overhead.

## Ownership: a value you can use at most once

> **🦀 From your toolbox →** Think about a Swift `class` instance under ARC, or a Python object: many variables can point at the same object, and the runtime counts how many before freeing. Rust flips this default on its head. For a value that owns a resource (like a `String`), there is at any moment exactly **one** binding responsible for it, and you can hand that responsibility on but never silently duplicate it. The closest everyday intuition: a value you can **use at most once** — once you pass it somewhere, it's "used up" and the original name no longer refers to anything live. Where the analogy *breaks down*: in Swift and Python you can freely make a second variable point at the same object and both stay valid; in Rust that second binding *takes over* and the first becomes unusable. Sharing in Rust is opt-in (you borrow, or you reach for `Rc`/`Arc`), not the default.

The three rules, stated precisely enough to reason with:

1. **Every value has exactly one owning binding.**
2. **There is exactly one owner at a time** (ownership can transfer, but not duplicate, for non-`Copy` types).
3. **When the owner goes out of scope, the value is *dropped*** — its destructor runs and its resources are released.

Rule 3 is the deterministic-destruction rule.

## Scope-bound destruction: cleanup runs automatically at end of scope

> **🦀 From your toolbox →** The mental model is closest to Python's `with`-block or Java's try-with-resources, except it's automatic and applies to *every* owning value, not just things you remember to wrap. A `String` owns a heap buffer; when it leaves scope, that buffer is freed for you — like a `__del__` or a `close()` that the language guarantees will fire, deterministically, at a point you can see in the source. The light C++ touch: it's the "a destructor runs at scope end" idea, but you almost never write the destructor yourself. Where the analogy *breaks down*: Python's `__del__` fires whenever the garbage collector gets around to it (and may never, on a reference cycle); Java finalizers are worse and effectively deprecated. Rust's drop point is fixed at compile time — the closing brace — so cleanup is both guaranteed and punctual.

```rust
{
    let buf = String::from("hello");   // buf owns a heap allocation here
    // ... use buf ...
}                                       // `}`: drop(buf) runs, heap memory freed
```

The compiler does not insert a runtime check to decide whether to free. The closing brace is a *statically known* program point, so `rustc` emits the deallocation call right there, exactly once. This is the heart of "no overhead": the drop is as cheap as the `free` you would have written by hand in C, minus the bugs. Where Java would let the GC reclaim the memory at some unpredictable later moment, and Swift would decrement a reference count at run time, Rust has already decided at compile time exactly where the free happens.

> **⚙️ Under the hood →** For a `String`, the compiler-generated drop calls the allocator's `dealloc` on the owned pointer. For a struct, drop glue runs any explicit `Drop::drop` first, then recursively drops each field. The compiler tracks, per code path, whether a value has been moved out; if a move *might* have happened on some paths but not others, it inserts a hidden boolean **drop flag** on the stack to decide at runtime whether to drop. Straight-line code with no conditional moves needs no flag — drop points are fully static.

## Stack and heap, and why Rust makes you care

You know the mechanics from *Operating Systems / Computer Architecture*: the stack is a contiguous, LIFO region tied to call frames; the heap is a general allocator you request blocks from and must return. What is new is that Rust surfaces this distinction *in the type system*, because it determines move behaviour.

A value whose size is known at compile time and contains no owned resources — `i32`, `bool`, `char`, `f64`, a fixed array of such, a tuple of such — lives entirely on the stack (or in a register). Duplicating it is a `memcpy` of a fixed number of bytes; there is nothing to free and no aliasing hazard.

A `String` or a `Vec<T>` is different. The *handle* — three words: `(pointer, length, capacity)` — lives on the stack, but it points at a heap buffer it owns.

```text
  stack (the String handle)        heap (the owned buffer)
  +----------+----------+        +---+---+---+---+---+
  | ptr      | --------------->  | h | e | l | l | o |
  | len  = 5 |                   +---+---+---+---+---+
  | cap  = 8 |
  +----------+
```

`len` is the bytes currently used; `cap` is the bytes the allocator handed out. The handle is `Copy`-able bit-for-bit, but the *buffer* is not — and that asymmetry is what forces the move rule.

> **🦀 From your toolbox →** This is exactly the shape of a Java `ArrayList` (a small object holding a reference to a backing array plus a `size`), a Swift `Array` (a struct holding a reference to heap storage plus a count), or a Python `list` (a header pointing at a growable array of element pointers). In all three the small front-object is cheap to copy around but the backing storage is shared or managed by the runtime. The difference is purely *who frees the buffer and when*: Java's GC, Swift's ARC, and Python's refcounting all decide at run time, whereas Rust answers it with ownership decided at compile time.

## Move semantics: assignment transfers the handle, then poisons the source

Consider:

```rust
let s1 = String::from("hello");
let s2 = s1;            // the handle is moved from s1 into s2
// println!("{s1}");    // ERROR: borrow of moved value: `s1`
```

`let s2 = s1;` does a bitwise copy of the *handle* (the three words) into `s2`. It does **not** copy the heap buffer. So momentarily both handles point at the same buffer — which would be a double-free waiting to happen when both go out of scope. Rust's resolution: after the move, `s1` is statically marked invalid. The buffer now has exactly one owner, `s2`, and exactly one drop will run.

There is no flag set at runtime, no nulling-out of `s1`. The *compiler* simply refuses to let you read `s1` again. The move is a "destructive memcpy of the handle": the bytes are copied, the source binding ceases to exist as far as the type checker is concerned, and there is no leftover moved-from object.

> **🦀 From your toolbox →** Contrast this with what you're used to. In Java or Python, `s2 = s1` makes `s2` a second reference to the *same* object, and both keep working — the runtime's GC/refcounting sorts out freeing later. In Swift the same is true for a class instance (ARC bumps the count) and for a value-type like `Array` you'd get copy-on-write semantics, but in every case the original variable stays usable. Rust deliberately does *not* leave you with two usable names for one resource: after `let s2 = s1;` the name `s1` is gone, and touching it is a **compile error**, not a silent shared reference. That's the price of having no runtime counting or tracing — there can only ever be one owner, so the compiler retires the old name.

> **⚠️ Pitfall →** Reading a moved-from value gives `error[E0382]: borrow of moved value: 's1'`, with the note *"move occurs because 's1' has type 'String', which does not implement the 'Copy' trait"*. The compiler usually suggests `s1.clone()`. The right fix depends on intent: if you genuinely need two independent owners, clone; if you only needed to *read* `s1`, borrow it with `&s1` (see [References & Borrowing](03-references-and-borrowing.md)) and never move at all.

> **🔧 In practice →** This shows up the first time you build any pipeline. Say you read a request body into a `String` and want to both log it and parse it:
> ```rust
> let body = read_body();          // String
> log_request(body);               // moves body away...
> let parsed = parse(body);        // ERROR: body was moved into log_request
> ```
> The compiler stops you before this becomes a bug. The fix you'll reach for 90% of the time is *don't move at all* — make `log_request(&body)` borrow, so `body` is still yours to `parse`. You only `clone()` when both callees genuinely need to *own* an independent copy (e.g. you hand one copy to a background thread and keep one locally). The error is Rust nudging you to decide: borrow, or pay for a copy on purpose.

### Assigning over a live binding drops the old value

The mirror image: assigning a fresh value to an existing `mut` binding drops the previous one *immediately*, not at end of scope.

```rust
let mut s = String::from("hello");
s = String::from("ahoy");          // the "hello" buffer is dropped right here
println!("{s}");                   // "ahoy"
```

Nothing points at `"hello"` after the second line, so the compiler frees it at the point of reassignment.

## `Copy` vs `Clone`: implicit cheap duplication vs explicit deep copy

For stack-only types there is no buffer to alias, so invalidating the source would be pointless. These types implement the **`Copy`** marker trait, and assignment/passing *copies* the bits instead of moving:

```rust
let x = 5;
let y = x;            // i32 is Copy: x is bit-copied into y
println!("{x} {y}");  // both valid — no move happened
```

`Copy` means "duplicating this value is a bitwise `memcpy` and the original remains valid." It is opt-in but trivially derivable, and the standard library implements it for all the scalars, `char`, `bool`, raw pointers, shared references `&T`, and tuples/arrays whose elements are all `Copy`. Anything that owns a resource — `String`, `Vec<T>`, `Box<T>`, a file handle — is **not** `Copy`, because a bitwise duplicate would create two owners of one buffer.

**`Clone`** is the explicit, possibly-expensive sibling. `s1.clone()` performs whatever deep copy the type defines — for `String`, a fresh heap allocation and a `memcpy` of the contents — producing a second independent owner:

```rust
let s1 = String::from("hello");
let s2 = s1.clone();              // new heap buffer; s1 still valid
println!("{s1} {s2}");
```

The design principle is visibility: Rust *never* deep-copies your data implicitly. Any implicit duplication is therefore guaranteed cheap (`Copy`, a fixed `memcpy`), and anything expensive is spelled out with a `.clone()` you can grep for. When you see `clone()`, you know arbitrary, potentially costly code runs there.

> **⚙️ Under the hood →** `Copy` and `Clone` are a supertrait pair: `Copy: Clone`, so every `Copy` type is also `Clone`. `Copy` carries no method; it is a flag the compiler reads to decide whether a `let`/argument bind is a non-destructive copy or a destructive move. `Clone::clone(&self) -> Self` is an ordinary method call you can override. You typically write `#[derive(Clone, Copy)]`; the derive generates a field-wise clone and, for `Copy`, simply asserts every field is `Copy`.

> **🦀 From your toolbox →** This maps cleanly onto a distinction you already feel in Java and Swift. `Copy` is like Java's primitives (`int`, `double`) or Swift's small value-types: assigning them just duplicates the bits and both sides are independent and valid. `Clone` is like calling `.clone()` on a Java object or writing an explicit deep copy in Swift — it's a method you invoke *by name*, and it can do real work. The point Rust adds is that the expensive case is *never* implicit: there's no silent deep copy hiding behind an assignment, so a `.clone()` in the source is the only place a heap copy can happen, and you can see (and grep for) every one of them.

## `Copy` and `Drop` are mutually exclusive — and that is not arbitrary

You cannot implement both `Copy` and `Drop` for the same type; the compiler rejects it. The reason follows directly from the model. `Copy` says "a bitwise duplicate is a complete, independent value." `Drop` says "this value owns something that needs cleanup." If a type were both, then copying it would produce two values, each of which would run its destructor — two cleanups for one logical resource, i.e. a double-free or double-close. So `Copy` is exactly the set of types for which "duplicate freely, never run special cleanup" is sound, and `Drop` is exactly the set for which it is not.

## Ownership flows through function boundaries

Passing an argument and returning a value obey the same move/copy rule as `let`. Passing a non-`Copy` value to a function *moves* it; the callee becomes the owner and drops it (unless it moves it out again):

```rust
fn consume(s: String) {       // s owned here
    println!("{s}");
}                             // drop(s): buffer freed

fn main() {
    let text = String::from("hi");
    consume(text);            // text moved into consume
    // println!("{text}");    // ERROR: text was moved away
    let n = 5;
    takes_copy(n);            // i32 is Copy: n still valid afterwards
    println!("{n}");
}
fn takes_copy(_n: i32) {}
```

Returning a value moves ownership *out* to the caller. The early days of Rust forced an awkward pattern: take a value, do something, and hand it back so the caller could keep using it.

```rust
fn len_and_return(s: String) -> (String, usize) {
    let n = s.len();
    (s, n)                    // give s back so the caller isn't stranded
}
```

This is the move discipline working correctly but painfully. It is the motivation for the next chapter: **borrowing** lets a function *use* a value through a reference without taking ownership, so you do not have to choreograph handing everything back.

> **⚠️ Pitfall →** A loop that moves an owned value out of a binding and then continues iterating will fail: `error[E0382]: use of moved value`, because after the first iteration the binding is empty. If you meant to reuse the value across iterations, borrow it (`&value`) or clone per iteration; if you meant to consume a collection element-by-element, iterate with `into_iter()` so the move is over the elements, not a single binding.

## The `Drop` trait: custom destructors

Drop glue is synthesised automatically, but when your type wraps a resource the standard library does not know how to release — a C handle, a lock guard, a temp file — you implement `Drop` to run cleanup at end of scope.

```rust
struct TempFile {
    path: String,
}

impl Drop for TempFile {
    fn drop(&mut self) {
        // real code would call std::fs::remove_file(&self.path)
        println!("removing {}", self.path);
    }
}

fn main() {
    let _t = TempFile { path: String::from("/tmp/scratch") };
    println!("working...");
}   // "removing /tmp/scratch" prints here, automatically
```

`Drop::drop` takes `&mut self`, not `self` — you are *finalising* the value in place, not consuming it (the value is already being destroyed; if `drop` took `self` by value it would need to drop *that*, recursing forever). `Drop` is in the prelude, so no import is needed.

> **🔧 In practice →** `Drop` is how you make "this always gets cleaned up, on every exit path" a property of the *type* rather than something every caller has to remember. The canonical real-world case is a lock guard: you lock a `Mutex`, get back a guard, and the moment that guard leaves scope — whether you returned normally, hit an early `return`, or bailed out with `?` — the lock is released. You never write the unlock.
> ```rust
> fn transfer(accounts: &Mutex<Ledger>, from: u32, to: u32, amt: u64) -> Result<(), Error> {
>     let mut ledger = accounts.lock().unwrap();   // guard acquired
>     ledger.debit(from, amt)?;   // if this returns Err, `?` exits early...
>     ledger.credit(to, amt)?;    // ...and the guard's Drop still unlocks the Mutex
>     Ok(())
> }   // guard drops here on the happy path
> ```
> This is the same trick behind temp-file cleanup, closing sockets, flushing buffered writers, and decrementing metrics counters. If you've ever leaked a lock in Java because an exception skipped your `finally`, this is the structural fix: cleanup is welded to scope exit and the compiler won't let you forget it. ([Smart pointers](15-smart-pointers.md) covers `MutexGuard` in detail.)

### Drop order is defined, deterministic, and worth memorising

Within a scope, **variables drop in reverse order of declaration** (LIFO — the last binding created is the first destroyed, matching stack discipline). Within a single value, after any explicit `Drop::drop` body runs, **fields drop in forward declaration order**. Both are guaranteed by the language, so you can rely on them for ordering-sensitive teardown (e.g. a child resource that must outlive nothing but is declared after its parent).

```rust
struct Loud(&'static str);
impl Drop for Loud {
    fn drop(&mut self) { println!("drop {}", self.0); }
}

fn main() {
    let _a = Loud("a");
    let _b = Loud("b");
}   // prints "drop b" then "drop a"  (reverse declaration order)
```

> **⚙️ Under the hood →** The compiler emits drop calls at every scope exit in reverse binding order. For a struct with an explicit `Drop`, it emits a call to your `drop` then recursively drops fields in source order. If control flow makes a drop conditional (value moved on one branch, not another), `rustc` introduces an implicit stack-allocated **drop flag** and branches on it — the only runtime cost ownership ever imposes, and it is elided whenever the analysis is unconditional.

### You cannot call `.drop()` yourself; use `std::mem::drop`

Explicitly invoking the destructor method is forbidden:

```rust
let t = TempFile { path: String::from("/tmp/x") };
// t.drop();   // ERROR
```

> **⚠️ Pitfall →** `t.drop()` gives `error[E0040]: explicit use of destructor method`. The reason: the automatic end-of-scope drop would *still* run afterwards, double-freeing. To finalise a value early — e.g. release a lock before the end of a long scope — pass it by value to the free function `std::mem::drop`, which takes ownership and lets the value fall out of scope inside the function:

```rust
let t = TempFile { path: String::from("/tmp/x") };
drop(t);                       // runs cleanup now; t is moved, can't be used again
// t is invalid here — exactly the move rule, so no second drop happens
```

`std::mem::drop` is almost comically simple: `pub fn drop<T>(_x: T) {}`. It takes `T` *by value*, so calling it *moves* the argument in; the function body is empty, so the value immediately goes out of scope at the function's closing brace and its destructor runs there. Because the value was moved, the original binding is invalidated by the ordinary move rule, so the end-of-scope drop is suppressed — no double free. `drop` is in the prelude.

## Why moves are inevitable, not arbitrary

Step back and the rules collapse into a single invariant the compiler is enforcing: **for any heap allocation (or other owned resource), there is exactly one binding responsible for freeing it, and that binding is statically identifiable at every program point.** Everything else is downstream:

- Implicit deep copies are banned ⇒ duplicating a handle would create two owners ⇒ assignment must *move* (transfer responsibility) for non-`Copy` types.
- A moved-from binding must be unusable ⇒ otherwise two bindings could both try to free.
- `Copy` is precisely the types where "duplicate, both valid" is sound (no owned resource) ⇒ those don't move.
- `Drop` + `Copy` is unsound ⇒ they're mutually exclusive.
- Drop runs exactly once at the owner's scope exit ⇒ deterministic cleanup, no GC, no manual `free`.

The cost is that you cannot freely share an owned value the way you would pass a Java object reference or a Python object around. The escape from "move everything" — using a value without consuming it — is [borrowing](03-references-and-borrowing.md): take a `&T` or `&mut T`, which lends access for a bounded region without transferring ownership. That is the next chapter, and it is what makes the ownership model ergonomic rather than merely safe.

## Mental-model recap

- Ownership means each non-`Copy` value has **one owner**, may be moved (used) **at most once**, and is **dropped** deterministically when its owner leaves scope — automatic, leak-free, zero-overhead cleanup at a compile-time-fixed point, unlike Java's GC, Swift's ARC, or Python's refcounting that all decide at run time.
- A move is a **destructive `memcpy` of the handle** that statically invalidates the source. Unlike a Java/Python reference assignment, you are not left with two usable names for one resource; use-after-move is a *compile* error (`E0382`), never a silent shared alias or UB.
- `Copy` (implicit, cheap, bitwise, source stays valid) is for stack-only resource-free types; `Clone` (explicit, named, possibly a heap allocation) is the only way to deep-copy. Implicit duplication is therefore always cheap.
- `Copy` and `Drop` are mutually exclusive: a type with cleanup cannot be silently duplicated, or you'd double-free.
- Drop order is defined: **variables reverse-of-declaration, struct fields forward-of-declaration**. You can't call `.drop()` (`E0040`); force early cleanup with `std::mem::drop(x)`, which works purely by *moving* `x` in.

## Exercises

1. **Predict the error, then choose the fix.** You have `let cfg = load_config();` (a `String`), and you want to call `validate(cfg)` and then `apply(cfg)`, both of which currently take `String` by value. Before touching the code, state which call fails and the error code you'll see. Now you have three options: clone for the first call, change one signature to borrow, or change both to borrow. Which do you pick, and what would have to be true about `validate`/`apply` for cloning to be the *right* choice rather than a workaround?

2. **Which of these compiles?** Without running them, decide for each whether it builds, and why:
   (a) `let a = 5; let b = a; println!("{a} {b}");`
   (b) `let a = String::from("hi"); let b = a; println!("{a} {b}");`
   (c) `let a = (1_i32, 2_i32); let b = a; println!("{a:?} {b:?}");`
   (d) `let a = (String::from("hi"), 2); let b = a; println!("{b:?}");` — and would adding `println!("{a:?}")` change the answer?
   State the single rule that decides each case.

3. **Predict the print order.** Declare four `Loud` values in one scope, with two of them inside a nested `{ }` block, then declare one more after the block closes. Write down the exact output *before* compiling, then verify. Explain the order using the reverse-declaration rule and what the nested scope's closing brace does.

4. **Make a design decision.** You're writing a `Connection` type that holds an OS socket handle that must be closed exactly once. Should `Connection` be `Copy`, `Clone`, both, or neither — and should it implement `Drop`? Justify each choice in terms of "how many owners can be responsible for closing the socket," and explain why the compiler would reject one of the combinations outright.

5. (★) **Adapt the early-drop pattern.** Using a `Guard` struct that holds a `&'static str` name and prints in its `Drop`, you want the guard finalised *before* a long computation runs (imagine it holds a lock you want to release early). Write the few lines that achieve this with `std::mem::drop`, then predict: what is the runtime output order relative to a "computation done" print, and what error (and code) do you get if you reference the guard after the early drop? Explain how the move rule guarantees the destructor still runs exactly once and not twice.
