# 3. References & Borrowing

[Ownership](02-ownership-and-moves.md) gave us a destructive default: passing a value by value *moves* it, and the source name is invalidated. That is exactly right for transferring responsibility for a resource, and exactly wrong for the overwhelmingly common case where a function just wants to *look at* — or *temporarily mutate* — a value it has no intention of keeping. Threading ownership in and then back out by return value (the tuple-juggling you saw at the end of module 02) is intolerable as a calling convention.

The fix is **borrowing**: creating a *reference* to a value without taking ownership of it. A reference is a non-owning pointer. Crucially, Rust attaches a static discipline to references — *aliasing XOR mutability* — that turns the reference type into a machine-checked guarantee of two distinct safety properties at once: freedom from data races and freedom from iterator invalidation. This is the chapter where Rust stops looking like a careful C++ and starts looking like nothing you have used before.

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

> **🦀 From your toolbox →** The closest bridge is Swift's `inout`, but it lines up with `&mut`, not `&`. When you write `func bump(_ n: inout Int)` and call `bump(&count)`, Swift passes a temporary writable handle to `count`, and the caller keeps the variable afterward — that is essentially a `&mut`. A plain `&T` (shared, read-only) has no direct Swift counterpart, because Swift's value types are copied when you pass them and its reference types (classes) are freely aliasable. In Java and Python, *every* object you pass is already passed by a shared, aliasable reference — so a `&T` feels familiar, but the analogy breaks down hard on the next rule: Java and Python place no limit on how many writable references to one object can be live at once, and Rust does. Hold that thought.

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

> **🎓 Tripos link →** From *Concurrent and Distributed Systems*: a **data race** is concurrent access to a shared location where at least one access is a write and there is no synchronisation. The aliasing-XOR-mutability law is precisely the negation of that situation, enforced at every program point — even single-threaded ones. If only shared references are live, every access is a read, so there is nothing to race. If a `&mut` is live, the law guarantees it is the *only* path to that value, so there is no second accessor to race against. Rust takes a hazard you normally only catch at runtime (or never) and turns it into something the compiler refuses to let you write. We will see in [fearless concurrency](13-fearless-concurrency.md) that this is not just an analogy: the `Send`/`Sync` machinery builds directly on the fact that a live `&mut T` is unaliased, which is how "no data races" becomes a compile-time guarantee rather than a hope.

The same law also kills **iterator invalidation** — a single-threaded bug that has nothing to do with concurrency. In C++, `v.push_back(x)` may reallocate the backing buffer and dangle every outstanding iterator and pointer into `v`; the standard says it's undefined behaviour and the compiler says nothing. (Java and Python catch the analogous "modified the list while iterating it" bug, but only at runtime — `ConcurrentModificationException` or surprising skipped elements.) Rust forbids it structurally, at compile time:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];   // shared borrow of v's contents
v.push(4);           // error[E0502]: push needs &mut v, but v is borrowed immutably
println!("{first}");
```

`Vec::push` has signature `fn push(&mut self, _)`, so calling it requires a `&mut v`. But `first` is a live `&v[0]`. Mutable-while-shared is illegal, so the program does not compile — the very same rule, the very same error code, defending against a completely different failure mode. *Aliasing XOR mutability* is the single invariant from which both data-race-freedom and pointer-stability fall out.

> **🔧 In practice →** This is the rule you will trip over the very first time you write a loop that edits a collection. Say you are scanning a `Vec<Job>` and want to add a retry job whenever you find a failed one:
> ```rust
> for job in &jobs {            // &jobs is a live shared borrow for the whole loop
>     if job.failed {
>         jobs.push(retry_of(job)); // E0502: needs &mut jobs while &jobs is live
>     }
> }
> ```
> The borrow checker stops you because growing `jobs` could reallocate and dangle the very `job` reference you're holding — exactly the iterator-invalidation crash that bites you at runtime in Java or Python. The idiomatic fix is to collect the new work first, then apply it once the read-only borrow has ended:
> ```rust
> let retries: Vec<Job> = jobs.iter().filter(|j| j.failed).map(retry_of).collect();
> jobs.extend(retries);        // no live borrow of jobs here — fine
> ```
> You will reach for this "gather-then-mutate" shape constantly; the compiler is naming a real bug, not nagging.

> **⚙️ Under the hood →** Because a live `&mut T` is provably the *only* live access path to its referent, rustc can hand LLVM the `noalias` attribute on `&mut` parameters — the equivalent of decorating every `&mut` argument with C's `restrict`, but soundly and automatically. The optimiser may keep the pointee in a register across calls, reorder loads/stores around it, and skip reloads, because no other pointer can touch that memory. Hand-written C earns `restrict` only where the programmer swears an aliasing oath the compiler cannot verify; Rust earns it everywhere `&mut` appears, for free.

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

> **🎓 Tripos link →** The "when does a borrow end" question is exactly the **liveness analysis** you meet in *Compiler Construction*: a variable (here, a borrow) is *live* at a program point if its value will still be used on some path forward, and *dead* once its last use is behind it. NLL is that backward dataflow applied to borrows — each borrow gets a region covering every path from where it's created to each place it's used, and the checker only complains where two such regions overlap in a forbidden way. The earlier, purely lexical checker (pre-2018) was a cruder approximation that ended every borrow at the closing brace, and so rejected perfectly safe programs that NLL now accepts.

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

The compiler's reasoning: the return type is a borrowed `&String`, but it can only borrow from something that outlives the call, and the only candidate is `s`, which dies when the function returns. There is no valid lender, so the program is rejected. The fix is to return the value by ownership — *move it out* — so the caller owns it and nothing is freed:

```rust
fn no_dangle() -> String {
    String::from("local")  // ownership moves to the caller; no deallocation
}
```

> **🔧 In practice →** This is the single most reassuring guarantee Rust gives you, and it pays off the moment you port habits from another language. In Swift, ARC keeps an object alive as long as *some* reference points at it; in Java, the garbage collector does the same; in Python, reference counting plus a cycle collector. All three keep your program from reading freed memory — but at the cost of a runtime collector and non-deterministic cleanup timing. Rust reaches the same safety with *zero* runtime machinery: the borrow checker proves at compile time that no reference can outlive what it points at, so cleanup happens at a known point (the end of the owner's scope) and a dangling pointer is simply not a program you can write. Concretely, if you find yourself writing a helper like `fn first_word(&self) -> &str` that hands back a slice into your own buffer, the compiler guarantees the caller can't hold that slice past the buffer's life — so you can expose internal views safely instead of defensively copying the way you might in Java or Swift.

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

> **🦀 From your toolbox →** In Java you signal "this method only reads" by convention or by `final` fields; in Swift you mark a method `mutating` when it changes a value-type's stored properties, and leave it unmarked when it only reads. Rust's `&self` vs `&mut self` makes that distinction *explicit in the type and enforced*: a `&self` method is Swift's non-`mutating`, a `&mut self` method is Swift's `mutating`, and Swift's compiler even forbids calling a `mutating` method through a `let` — which is almost exactly Rust forbidding `c.bump()` when `c` isn't `mut`. Where Rust goes further than all three: `self`-by-value methods (like `into_inner`) *consume* the receiver, and the compiler then forbids any later use of the moved-from binding. Java and Python have no equivalent — the object lives on, garbage-collected whenever — and Swift's value semantics copy rather than poison the original. The C++ touch, if you want one: `&self` is like a `const` method and `self`-by-value is like a method that's allowed to gut its receiver, but unlike C++ `const`, Rust's read-only promise can't be cast away.

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

1. **Predict the error, name the receiver.** Before compiling, say which line fails and with which error code, then explain in one sentence *why* — naming the receiver type that `push` requires:
   ```rust
   let mut s = String::from("x");
   let r = &s;
   s.push('y');
   println!("{r}");
   ```
   Now: would deleting the final `println!("{r}")` line make it compile? Answer using the NLL rule about when `r`'s borrow ends.

2. **Which compile, and why?** Three sibling functions claim to read a `Counter` and return its value. Decide which compile and which don't, *without* running them, and give the one-line reason for each:
   ```rust
   fn a(c: &Counter) -> u32 { c.value() }
   fn b(c: &Counter) -> u32 { c.bump(); c.value() }     // bump is &mut self
   fn d(c: Counter)  -> u32 { let v = c.value(); v }
   ```
   For the one that fails, state the minimal change to its *signature* that fixes it, and what that change forces on every caller.

3. **Design decision: `&mut self` vs `self`.** You are writing a `StringBuilder` with an `append(...)` method. Sketch the two receiver choices — `fn append(&mut self, part: &str)` versus `fn append(self, part: &str) -> Self` — and decide which you would ship. State one concrete calling pattern that each style makes pleasant and the other makes awkward. (Hint: think about chaining `b.append("a").append("b")` versus reusing `b` in a loop.)

4. **Adapt the disjoint-borrow rule.** The two-field split borrow compiles. Predict what happens if you change `Pair` to hold its data in an array, `struct Pair { vals: [i32; 2] }`, and try:
   ```rust
   let a = &mut p.vals[0];
   let b = &mut p.vals[1];
   ```
   Will the borrow checker accept it? Explain *why* the array case differs from the named-field case in terms of what the checker can and cannot "see," and name the helper you'd reach for to split the array soundly.

5. (★) **Two fixes, one shipping decision.** Write a function that tries to return a reference to a `String` it builds internally, observe the E0106 error, then produce **two** working designs: one that returns an owned `String`, and one that takes a `&mut String` out-parameter and writes into it (returning `()`). Argue which you would ship, referencing the cost model from [ownership](02-ownership-and-moves.md) — and name one realistic situation (e.g. filling a reused buffer inside a hot loop) where the *less obvious* design actually wins.
