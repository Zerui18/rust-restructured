# 3. References & Borrowing

[Ownership](02-ownership-and-moves.md) gave us a destructive default: passing a value by value *moves* it, and the source name is invalidated. That is exactly right for transferring responsibility for a resource, and exactly wrong for the overwhelmingly common case where a function just wants to *look at* — or *temporarily mutate* — a value it has no intention of keeping. Threading ownership in and then back out by return value (the tuple-juggling you saw at the end of module 02) is intolerable as a calling convention.

The fix is **borrowing**: creating a *reference* to a value without taking ownership of it. A reference is a non-owning pointer. Crucially, Rust attaches a static discipline to references — *aliasing XOR mutability* — that turns the reference type into a machine-checked proof of two distinct safety properties at once: freedom from data races and freedom from iterator invalidation. This is the chapter where Rust stops looking like a careful C++ and starts looking like nothing you have used before.

## References are pointers with a verb attached

You write `&x` to borrow `x` and `&mut x` to borrow it mutably. The types are `&T` and `&mut T`. The dual operation is dereference, `*r`, though method calls and field access auto-deref so you rarely write the star.

```rust
fn length(s: &String) -> usize {
    s.len() // auto-deref: (*s).len()
} // s drops here, but the String it points at does NOT — s never owned it

fn main() {
    let owner = String::from("borrow me");
    let n = length(&owner);          // lend, don't give
    println!("{owner} has length {n}"); // owner still valid: never moved
}
```

The function signature `s: &String` is a *contract*: "I will read your `String`, I will not keep it, give it back when I'm done." When `s` goes out of scope at the closing brace, only the reference is dropped — there is nothing to free, because the reference is not the owner. That single fact is what eliminates the return-the-value-back dance.

> **🦀 From your toolbox →** A `&T` is morally a C++ `const T&` and a `&mut T` is a `T&`. The analogy is *load-bearing only for the data layout* — both are just addresses, identical to a `const T*` / `T*` at the ABI level (a non-null, aligned, dereferenceable pointer). It breaks down completely on guarantees. A C++ `T&` promises nothing: you can have a dozen mutable references to the same object live at once, alias them through raw pointers, and read one while writing another. Rust's references carry a compiler-enforced exclusivity invariant that C++ references do not have, do not check, and cannot express. Equally: a `&T` is *not* a Swift `inout` (that is closer to `&mut`), and it is *not* an OCaml `ref` cell (that is a heap-allocated mutable box, i.e. interior mutability — see [smart pointers](15-smart-pointers.md)).

## The law: aliasing XOR mutability

Here is the entire rule, and you should memorise it verbatim:

> At any point in the program, for any given value, you may have **either** any number of shared references `&T` **or** exactly one mutable reference `&mut T` — **never both, and never two `&mut`.**

That is why `&T` is also called a *shared* reference and `&mut T` an *exclusive* reference; those names describe the invariant better than "immutable/mutable" do. The borrow checker rejects any program in which the regions of a `&mut` and *any* other live borrow of the same value overlap.

```rust
fn main() {
    let mut s = String::from("data");

    let a = &s;          // shared borrow #1 — fine
    let b = &s;          // shared borrow #2 — fine, readers don't conflict
    println!("{a} {b}"); // both used here

    let m = &mut s;      // exclusive borrow — fine, a and b are dead now
    m.push('!');
    println!("{m}");
}
```

Two attempts that violate the law, with the exact diagnostics:

```rust
let mut s = String::from("data");
let r1 = &mut s;
let r2 = &mut s;        // error[E0499]: cannot borrow `s` as mutable more than once at a time
println!("{r1} {r2}");
```

```rust
let mut s = String::from("data");
let r = &s;             // immutable borrow
let m = &mut s;         // error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
println!("{r} {m}");
```

> **🎓 Tripos link →** From *Concurrent and Distributed Systems*: a **data race** is concurrent access to a shared location where at least one access is a write and there is no synchronisation. The aliasing-XOR-mutability law is precisely the negation of that definition applied at every program point, including single-threaded ones. If only shared references are live, every access is a read — no race. If a `&mut` is live, the law guarantees it is the *only* path to that value — no aliasing, so no race. Rust lifts a runtime concurrency hazard into a static invariant of the type system. We will see in [fearless concurrency](13-fearless-concurrency.md) that this is not an analogy but the actual mechanism: the `Send`/`Sync` traits build directly on the fact that `&mut T` is unaliased, so "no data races at compile time" is a *theorem*, not a slogan.

The same law also kills **iterator invalidation** — a single-threaded bug that has nothing to do with concurrency. In C++, `v.push_back(x)` may reallocate the backing buffer and dangle every outstanding iterator and pointer into `v`; the standard says it's UB and the compiler says nothing. Rust forbids it structurally:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];   // shared borrow of v's contents
v.push(4);           // error[E0502]: push needs &mut v, but v is borrowed immutably
println!("{first}");
```

`Vec::push` has signature `fn push(&mut self, _)`, so calling it requires a `&mut v`. But `first` is a live `&v[0]`. Mutable-while-shared is illegal, so the program does not compile — the very same rule, the very same error code, defending against a completely different failure mode. *Aliasing XOR mutability* is the single invariant from which both data-race-freedom and pointer-stability fall out.

> **⚙️ Under the hood →** Because a live `&mut T` is provably the *only* live access path to its referent, rustc can hand LLVM the `noalias` attribute on `&mut` parameters — the equivalent of decorating every `&mut` argument with C's `restrict`, but soundly and automatically. The optimiser may keep the pointee in a register across calls, reorder loads/stores around it, and skip reloads, because no other pointer can touch that memory. Hand-written C earns `restrict` only where the programmer swears an aliasing oath the compiler cannot verify; Rust earns it everywhere `&mut` appears, for free, with a proof.

## Non-lexical lifetimes: a borrow ends at its last use

When does a borrow's region *end*? Not at the closing brace of the enclosing scope — at the **last point the reference is actually used**. The borrow checker is a dataflow analysis over the control-flow graph, and a borrow is "live" only along paths between its creation and its final use. This refinement is called **non-lexical lifetimes** (NLL), and it is why the first example above compiled.

Watch the difference. Lexically, `r` exists until the end of `main`, which would overlap the `&mut`. Under NLL, `r`'s region ends at the `println!`:

```rust
fn main() {
    let mut s = String::from("hi");
    let r = &s;
    println!("{r}");   // last use of r — its borrow region ENDS here
    let m = &mut s;    // OK: no live shared borrow overlaps this
    m.push('!');
    println!("{m}");
}
```

Reorder so the uses interleave and it breaks, because now the regions genuinely overlap:

```rust
fn main() {
    let mut s = String::from("hi");
    let r = &s;
    let m = &mut s;    // error[E0502]: r is still live...
    println!("{r} {m}"); // ...because r is used HERE, after m's creation
}
```

> **🎓 Tripos link →** This is exactly the dataflow framework from *Compiler Construction* and the *region* discipline from *Semantics of Programming Languages*. Each borrow is assigned a *region* (a set of CFG points) constrained so the borrow's region covers every path from its introduction to each use. The checker proves the program well-typed iff no two regions violate the aliasing-XOR-mutability constraint where they overlap. "Last use" is just liveness analysis: a borrow is dead once it is no longer live in the standard backward-flow sense. The earlier, purely lexical checker (pre-2018) was a strictly weaker approximation that rejected sound programs; NLL is the same soundness theorem with a tighter region inference.

## Shared references are `Copy`; `&mut` is not

A `&T` is `Copy` — duplicating an address is harmless precisely because all you can do through it is read, and unlimited readers are fine. So passing or assigning a `&T` copies the pointer and leaves the original usable.

A `&mut T` is emphatically **not** `Copy`. If it were, you could trivially manufacture two exclusive references to the same value, demolishing the law. Assigning or passing a `&mut` therefore *moves* it:

```rust
let mut x = 5;
let r = &mut x;
let r2 = r;          // MOVE, not copy
*r2 += 1;
println!("{}", *r);  // error[E0382]: borrow of moved value: `r`
```

This looks like a problem for the everyday case of threading a `&mut` through a chain of function calls — if every pass moved it away, you could only use it once. The escape hatch is **reborrowing**.

## Reborrowing: lending out a loan

When you pass a `&mut` somewhere, the compiler does not move your reference; it *reborrows* through it, implicitly forming `&mut *r` — a fresh, shorter-lived exclusive borrow derived from `r`. The reborrow is exclusive for *its* region, and while it is alive your original `r` is frozen (you cannot use `r` concurrently, which would alias). Once the reborrow's region ends, `r` is usable again.

```rust
fn bump(n: &mut i32) { *n += 1; }

fn main() {
    let mut x = 5;
    let r = &mut x;
    bump(r);   // reborrow: compiler passes &mut *r; r frozen during the call, alive after
    bump(r);   // r usable again — used twice despite &mut not being Copy
    println!("{}", *r);
}
```

The borrow stack discipline (original outlives reborrow, original frozen while reborrow lives) is what keeps reborrowing sound: at every instant there is still exactly one usable exclusive path. You almost never *write* `&mut *r` explicitly — coercion inserts it at call sites and assignments — but knowing it is happening explains why `&mut` "works like a copy in practice" without being `Copy`.

> **⚙️ Under the hood →** A reference is a single machine word (a thin pointer) for `Sized` referents; `&T` and `&mut T` have identical runtime representation — the distinction is purely static, erased before codegen. (References to unsized types like `&[T]` and `&str` are *fat* pointers carrying a length — see [slices](05-slices-and-duality.md).) "Reborrow" is not a runtime operation: no instruction is emitted, no pointer is copied beyond the ordinary argument pass. It is a bookkeeping entry in the borrow checker's region graph that says "this new borrow is nested inside that old one."

## Dangling references are unrepresentable

A reference, by definition of the law plus liveness, can never outlive its referent. The classic C footgun — return the address of a stack local, deallocate the frame, hold a pointer into freed memory — is rejected at the type level:

```rust
fn dangle() -> &String {   // error[E0106]: missing lifetime specifier
    let s = String::from("local");
    &s                     // s is dropped at the closing brace; this would dangle
}
```

The compiler's reasoning: the return type is a borrowed `&String`, but it can only borrow from something that outlives the call, and the only candidate is `s`, which dies when the function returns. There is no valid lender, so the program is ill-typed. The fix is to return the value by ownership — *move it out* — so the caller owns it and nothing is freed:

```rust
fn no_dangle() -> String {
    String::from("local")  // ownership moves to the caller; no deallocation
}
```

> **⚠️ Pitfall →** The error `E0106: missing lifetime specifier` is one of the first walls every newcomer hits, and the message is initially baffling because it points at lifetimes rather than at the dangling pointer. The compiler genuinely cannot tell *which* input the output should borrow from (here: none), so it demands you name the relationship explicitly. When you see E0106 on a function that returns a reference built from a local, the diagnosis is almost always "you meant to return an owned value" — drop the `&` from the return type. When you *did* mean to return a borrow of an input, you need to annotate the connection. Both roads lead to [lifetimes](04-lifetimes.md), which is the entire subject of the next module; for now, internalise only that dangling is *impossible*, not merely *discouraged*.

## `&self`, `&mut self`, `self`: the three receivers

Methods make the borrow choice part of the API. The receiver is written in shorthand, but it is just an ordinary parameter of one of three forms, and the choice tells callers exactly what the method does to the value:

| Receiver | Full form | Meaning | Caller cost |
|---|---|---|---|
| `&self` | `self: &Self` | reads the value | shared borrow, value stays usable |
| `&mut self` | `self: &mut Self` | mutates the value in place | exclusive borrow, value stays usable |
| `self` | `self: Self` | **consumes** the value | move; value invalidated at call site |

```rust
struct Counter { n: u32 }

impl Counter {
    fn value(&self) -> u32 { self.n }        // reads
    fn bump(&mut self) { self.n += 1; }      // mutates in place
    fn into_inner(self) -> u32 { self.n }    // consumes, returns the field
}

fn main() {
    let mut c = Counter { n: 0 };
    c.bump();                       // needs &mut c — c must be `mut`
    println!("{}", c.value());      // needs &c — fine after the &mut ended (NLL)
    let final_value = c.into_inner(); // moves c away
    // c is now gone; using it here would be E0382 use-after-move
    println!("{final_value}");
}
```

The same aliasing law governs receivers, which is why you cannot call a `&mut self` method on a value you are simultaneously borrowing immutably — that is the `Vec::push` case from earlier, dressed as method-call syntax. `self` (by-value) is reserved for builder-style consuming transforms and conversions (`into_*`); reach for it only when the operation logically destroys or transforms-away the original. We cover defining these in [structs and methods](06-structs-and-methods.md).

> **🦀 From your toolbox →** This is C++'s `const`/non-`const`/by-value member function distinction (`u32 value() const`, `void bump()`, `u32 into_inner() &&`), and the parallel is unusually tight — `&self` ≈ `const`-qualified, `self` ≈ rvalue-ref-qualified `&&` consuming method. Where it breaks: a C++ non-`const` method gives you no exclusivity, so two callers can interleave mutations through aliasing references with no diagnostic; and `const` in C++ is shallow and castable-away (`const_cast`), whereas `&self` exclusivity is enforced globally and cannot be subverted in safe code. Also note `self`-by-value *moves* and statically poisons the caller's binding — there is no C++ equivalent of the compiler then *forbidding* later use of the moved-from object.

## A subtlety: borrows can be disjoint

The law applies *per value*, and the borrow checker understands that distinct fields of a struct are distinct values. So you can hold two `&mut` into *different* fields simultaneously — there is no aliasing, hence no conflict:

```rust
struct Pair { left: i32, right: i32 }

fn main() {
    let mut p = Pair { left: 1, right: 2 };
    let a = &mut p.left;
    let b = &mut p.right;  // OK: disjoint fields, no overlap
    *a += 10;
    *b += 20;
    println!("{} {}", p.left, p.right);
}
```

This *disjoint-field* (or "split") borrowing is essential in practice and is why you structure data so that independently-mutated state lives in separate fields. The checker's field-sensitivity is limited, though: it does not look *through* function calls or indices, so splitting a slice into two mutable halves needs the explicit `split_at_mut` helper (covered in [slices](05-slices-and-duality.md)), and splitting across a method boundary sometimes forces a refactor. When the disjointness is real but the checker can't see it, that is the signal you may need such a helper — not a sign the code is unsound.

## Mental-model recap

- A reference is a non-owning pointer; `&T` shares (read-only, `Copy`), `&mut T` is exclusive (read-write, *moves* and reborrows). Dropping a reference frees nothing.
- The one law: at any program point a value has **either** N shared borrows **or** one exclusive borrow, never both. Memorise it as *aliasing XOR mutability*.
- That single invariant statically proves *both* data-race-freedom (no aliased writes) *and* iterator-invalidation-freedom (no mutation under an outstanding read) — and lets rustc emit `noalias` on every `&mut`.
- Borrows are non-lexical: a borrow's region runs from creation to its **last use**, computed by dataflow/liveness over the CFG, not by lexical scope.
- A reference can never outlive its referent, so dangling is unrepresentable; returning a borrow of a local is the E0106 error, fixed by returning an owned value (or annotating a [lifetime](04-lifetimes.md)).

## Exercises

1. Predict the exact error code before compiling, then check:
   ```rust
   let mut s = String::from("x");
   let r = &s;
   s.push('y');
   println!("{r}");
   ```
   Explain in one sentence why `push` is the offending call, naming its receiver type.

2. The following *fails* lexically but you can make it compile by moving exactly one line, changing nothing else. Do it, and state which NLL property your edit relies on.
   ```rust
   let mut v = vec![10, 20];
   let head = &v[0];
   v.push(30);
   println!("{head}");
   ```

3. Write a function `swap_fields(p: &mut Pair)` for the `Pair` struct above that swaps `left` and `right` in place. Then explain why you do *not* need two `&mut` borrows to do it, and what would happen to the disjoint-field example if you tried to call `swap_fields(&mut p)` while `a` (a `&mut p.left`) was still live.

4. (★) `&mut T` is not `Copy`, yet this compiles and prints `7`. Explain precisely, in terms of reborrowing and which binding is "frozen" when, why it is sound:
   ```rust
   fn add(n: &mut i32, k: i32) { *n += k; }
   let mut x = 0;
   let r = &mut x;
   add(r, 3);
   add(r, 4);
   println!("{}", *r);
   ```
   Then change the second call to `let r2 = r; add(r2, 4);` and explain why *that* moves while the call form did not.

5. (★) Without using `unsafe`, write a function that *attempts* to return a reference to a `String` it computes internally, observe the E0106 error, and then produce **two** different fixes: one that returns an owned `String`, and one that takes the storage as a `&mut` out-parameter and returns `()`. Argue which design you would ship and why, referencing the cost model from [ownership](02-ownership-and-moves.md).
