# 2. Ownership & Moves

This is the chapter the rest of the book pivots on. Ownership is the single mechanism by which Rust statically guarantees memory safety and freedom from data races without a garbage collector and without manual `free`. Everything later — [references](03-references-and-borrowing.md), [lifetimes](04-lifetimes.md), [smart pointers](15-smart-pointers.md), [fearless concurrency](13-fearless-concurrency.md) — is a refinement of, or an escape valve around, the rules introduced here.

You already know two extreme answers to the question *"when does this allocation get freed?"*. The garbage-collected answer (Java, OCaml, Python, JS): a runtime traces reachability and frees whenever it likes, at a cost in latency and determinism. The manual answer (C): you call `free` yourself, and the compiler does not check that you do it exactly once at the right moment. Rust takes a third position: the *type system* tracks who is responsible for each value, and the compiler inserts the deallocation for you at a statically known point. No tracing, no manual `free`, no leaks-by-default, and — crucially — it is all proven at compile time, so there is zero runtime overhead.

## Ownership is an affine type discipline

> **🎓 Tripos link →** In *Semantics of Programming Languages* and *Logic and Proof* you met linear logic, where a hypothesis must be used *exactly once*. Rust's ownership is the slightly weaker **affine** discipline: a value may be used *at most once* (zero uses is fine — the value is just dropped unused). A binding to a non-`Copy` value is a resource; "using" it by move consumes it. Curry–Howard gives you the framing precisely: the type `String` is not a proposition you may reuse freely; it is a resource whose proof of ownership is spent when you pass it on. This is why `let s2 = s1;` invalidates `s1` — the resource has moved, and the affine logic forbids a second use of the spent hypothesis.

The three rules, stated formally enough to reason with:

1. **Every value has exactly one owning binding.**
2. **There is exactly one owner at a time** (ownership can transfer, but not duplicate, for non-`Copy` types).
3. **When the owner goes out of scope, the value is *dropped*** — its destructor runs and its resources are released.

Rule 3 is the deterministic-destruction rule, and it is the one your C++ instincts will recognise immediately.

## Scope-bound destruction is RAII without the footguns

> **🦀 From your toolbox →** This is C++ RAII. A `String` is morally a `std::string`: it owns a heap buffer, and when it leaves scope its destructor frees that buffer. Where the analogy *breaks*: in C++ you must hand-write the destructor, copy constructor, move constructor, and assignment operators (the "rule of five") to get this right, and a forgotten or wrong one is a silent leak or double-free. In Rust the compiler synthesises the drop *glue* for every type by recursively dropping its fields; you only write a destructor when you have something special to release. And there is no copy constructor at all — copying is never implicit for owning types (see moves, below).

```rust
{
    let buf = String::from("hello");   // buf owns a heap allocation here
    // ... use buf ...
}                                       // `}`: drop(buf) runs, heap memory freed
```

The compiler does not insert a runtime check to decide whether to free. The closing brace is a *statically known* program point, so `rustc` emits the deallocation call right there, exactly once. This is the heart of "no overhead": the drop is as cheap as the `free` you would have written by hand in C, minus the bugs.

> **⚙️ Under the hood →** For a `String`, the compiler-generated drop calls the allocator's `dealloc` on the owned pointer. For a struct, drop glue runs any explicit `Drop::drop` first, then recursively drops each field. The compiler tracks, per code path, whether a value has been moved out; if a move *might* have happened on some paths but not others, it inserts a hidden boolean **drop flag** on the stack to decide at runtime whether to drop. Straight-line code with no conditional moves needs no flag — drop points are fully static.

## Stack and heap, and why Rust makes you care

You know the mechanics from *Computer Architecture & OS*: the stack is a contiguous, LIFO region tied to call frames; the heap is a general allocator you request blocks from and must return. What is new is that Rust surfaces this distinction *in the type system*, because it determines move behaviour.

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

> **🦀 From your toolbox →** This handle is exactly `std::vector`'s `{begin, end, end_of_storage}` or a `std::string`'s small layout, and the `(ptr, len, cap)` triple is the same idea as a Swift `Array`'s storage reference or an OCaml `Bytes.t` boxed behind a pointer. The difference is purely *who frees the buffer and when*, and Rust answers that with ownership rather than ARC (Swift) or the GC (OCaml/Java).

## Move semantics: assignment transfers the handle, then poisons the source

Consider:

```rust
let s1 = String::from("hello");
let s2 = s1;            // the handle is moved from s1 into s2
// println!("{s1}");    // ERROR: borrow of moved value: `s1`
```

`let s2 = s1;` does a bitwise copy of the *handle* (the three words) into `s2`. It does **not** copy the heap buffer. So momentarily both handles point at the same buffer — which would be a double-free waiting to happen when both go out of scope. Rust's resolution: after the move, `s1` is statically marked invalid. The buffer now has exactly one owner, `s2`, and exactly one drop will run.

There is no flag set at runtime, no nulling-out of `s1`. The *compiler* simply refuses to let you read `s1` again. The move is a "destructive memcpy of the handle": the bytes are copied, the source binding ceases to exist as far as the type checker is concerned, and there is no leftover moved-from object.

> **🦀 From your toolbox →** This is the decisive break from C++. A C++ move (`std::string s2 = std::move(s1);`) runs a *move constructor* that leaves `s1` in a "valid but unspecified" state — you may still call methods on it, assign to it, and its destructor *will still run*. That residual moved-from state is a class of bugs (use-after-move that compiles cleanly and does something subtly wrong). Rust has no move constructor and no moved-from state: a moved-from binding is *gone*, and touching it is a **compile error**, not undefined behaviour. Consequently every Rust move is a trivial `memcpy` — the type author never writes move logic, and the compiler is free to elide the copy entirely.

> **⚠️ Pitfall →** Reading a moved-from value gives `error[E0382]: borrow of moved value: 's1'`, with the note *"move occurs because 's1' has type 'String', which does not implement the 'Copy' trait"*. The compiler usually suggests `s1.clone()`. The right fix depends on intent: if you genuinely need two independent owners, clone; if you only needed to *read* `s1`, borrow it with `&s1` (see [References & Borrowing](03-references-and-borrowing.md)) and never move at all.

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

> **🦀 From your toolbox →** Map this onto C++: `Copy` is roughly a trivially-copyable type with an implicit copy constructor that is a `memcpy`; `Clone` is an explicit, user-defined copy you must *call by name*. The break: C++ copy constructors fire implicitly on every pass-by-value, which is why accidental deep copies are a classic C++ performance trap. Rust makes the expensive case syntactically loud (`clone()`) and the cheap case the only implicit one.

## `Copy` and `Drop` are mutually exclusive — and that is not arbitrary

You cannot implement both `Copy` and `Drop` for the same type; the compiler rejects it. The reason follows directly from the affine model. `Copy` says "a bitwise duplicate is a complete, independent value." `Drop` says "this value owns something that needs cleanup." If a type were both, then copying it would produce two values, each of which would run its destructor — two cleanups for one logical resource, i.e. a double-free or double-close. So `Copy` is exactly the set of types for which "duplicate freely, never run special cleanup" is sound, and `Drop` is exactly the set for which it is not.

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

> **🎓 Tripos link →** This is exactly the resource-cleanup obligation that *Concurrent and Distributed Systems* drills into you for locks: every `acquire` needs a matching `release` on every path, including exceptional ones. Rust encodes the release as the `Drop` impl of a guard type ([smart pointers](15-smart-pointers.md) covers `MutexGuard`), so the lock is released the instant the guard leaves scope — no manual `unlock`, no path you forgot, no lock leak on early return.

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
- Drop runs exactly once at the owner's scope exit ⇒ deterministic RAII, no GC, no manual `free`.

The cost is that you cannot freely share an owned value the way you would pass a Java object reference around. The escape from "move everything" — using a value without consuming it — is [borrowing](03-references-and-borrowing.md): take a `&T` or `&mut T`, which lends access for a bounded region without transferring ownership. That is the next chapter, and it is what makes the ownership model ergonomic rather than merely safe.

## Mental-model recap

- Ownership is an **affine** discipline: each non-`Copy` value has one owner, may be moved (used) at most once, and is **dropped** deterministically when its owner leaves scope — RAII with compiler-synthesised, leak-free, zero-overhead destruction.
- A move is a **destructive `memcpy` of the handle** that statically invalidates the source. Unlike C++, there is no move constructor and no valid moved-from state; use-after-move is a *compile* error (`E0382`), never UB.
- `Copy` (implicit, cheap, bitwise, source stays valid) is for stack-only resource-free types; `Clone` (explicit, named, possibly a heap allocation) is the only way to deep-copy. Implicit duplication is therefore always cheap.
- `Copy` and `Drop` are mutually exclusive: a type with cleanup cannot be silently duplicated, or you'd double-free.
- Drop order is defined: **variables reverse-of-declaration, struct fields forward-of-declaration**. You can't call `.drop()` (`E0040`); force early cleanup with `std::mem::drop(x)`, which works purely by *moving* `x` in.

## Exercises

1. Write a function `fn first_word(s: String) -> usize` that returns the index of the first space (or the length if none). Call it, then try to print the original `String` afterwards. Read the `E0382` error carefully, then fix it *two* ways: (a) by cloning at the call site, (b) by changing the signature to borrow. Which is idiomatic and why?

2. Declare four `Loud` values (the noisy-`Drop` struct from this chapter) in a single scope, two of them inside a nested `{ }` block. Predict the exact print order *before* compiling, then verify. Explain the order using the reverse-declaration and nested-scope rules.

3. Define `struct Pair { a: Loud, b: Loud }` where `Loud` has a noisy `Drop`, and give `Pair` itself a noisy `Drop` too. Predict and then verify the order of all three messages when a `Pair` leaves scope. (★) Now swap the declaration order of `a` and `b` and explain what changed and what didn't.

4. Try to put `#[derive(Copy, Clone)]` on a struct that contains a `String` field. Read the error, and explain in one sentence — in terms of ownership of the heap buffer — why the compiler is right to reject it.

5. (★) Write a `Guard` struct holding a `&'static str` name with a noisy `Drop`. In `main`, create one, then use `std::mem::drop` to finalise it early, then attempt to use it again on the next line. Predict both the runtime output *and* the compile error you'll get from the post-drop use, and explain how the move rule guarantees the destructor runs exactly once.

6. Given `let v = vec![String::from("x"), String::from("y")];`, write a `for` loop that moves each `String` out and passes it to a consuming function. Then try to use `v` after the loop. Explain why `into_iter()` is what makes the per-element move legal and why the binding `v` is unusable afterwards.
