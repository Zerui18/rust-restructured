# 15. Smart Pointers & Interior Mutability

A *smart pointer* is a struct that owns the data it points to, behaves like a pointer (you dereference it with `*`), and runs custom code when it dies. The core idea is the same one Swift's ARC gives you for free: an object's storage is tied to a handle, and the handle's lifetime decides when the storage is released. In Java you'd lean on the garbage collector for the "release" half; here a stack object's scope drives cleanup deterministically, the way a destructor runs at the end of a scope in C++ — except the timing is exact and decided at compile time, not whenever a collector happens to run.

What is *genuinely new* in Rust is not the pattern but how cleanly it composes with the type system. A reference (`&T`, [borrowing](03-references-and-borrowing.md)) borrows; a smart pointer typically *owns*. Two traits make a struct "smart":

- `Deref` (and `DerefMut`) — lets `*p` and method calls see through the wrapper to the inner `T`. This is the autoderef machinery from [method resolution](06-structs-and-methods.md).
- `Drop` — the destructor, the deterministic cleanup hook from [ownership](02-ownership-and-moves.md).

This chapter covers the four you will actually reach for — `Box<T>`, `Rc<T>`, `RefCell<T>`, `Weak<T>` — and the pattern that ties them together: *interior mutability*, the controlled escape hatch from Rust's aliasing-XOR-mutability rule. The headline insight: **`RefCell` and `Rc` do not weaken the borrow rules; they move enforcement from compile time to run time.** You trade a static guarantee for a dynamic one — paying in panics and reference-count bumps, not in undefined behaviour.

> **🦀 From your toolbox →** Map these onto memory tools you already know. `Rc<T>` is Swift's ARC, made explicit: a value with a count of strong owners, freed the instant the count hits zero — except you can *see* the count (`Rc::strong_count`) and you choose when sharing happens (`Rc::clone`). `Arc<T>` is the same thing but safe to share across threads, like an `@Sendable`-safe reference. Java gives you sharing too, but a garbage collector decides *when* the memory goes away; Rust frees it deterministically at a known point. Python's `sys.getrefcount` object model is the closest mental match to `Rc`: an integer count that drops storage at zero. Where the analogy breaks down: Swift's ARC and Python's refcounting *also* let you mutate shared objects freely, because both languages accept the data-race risk; Rust splits sharing (`Rc`) from mutation (`RefCell`) so the aliasing rule survives. (A light C++ touch if you've seen it: `Rc`/`Arc`/`Weak` line up with `shared_ptr`/`shared_ptr`/`weak_ptr`.)

## `Box<T>`: one owner, on the heap

`Box<T>` is the minimal smart pointer: it heap-allocates a `T`, stores the pointer on the stack, owns the allocation, and frees it on drop. No reference count, no runtime check — just a single owned heap value with automatic cleanup.

```rust
let boxed: Box<i32> = Box::new(42);
println!("{}", *boxed); // 42 — deref reads through to the heap
// boxed dropped here: heap allocation freed
```

A bare `Box<i32>` is almost never useful — a plain `i32` lives on the stack and is faster. You reach for `Box<T>` for three reasons:

1. **Recursive types** that would otherwise have infinite size.
2. **Trait objects** — `Box<dyn Trait>`, owning a value by its behaviour rather than its concrete type (see [trait objects](09-trait-objects-and-oop.md)).
3. **Cheap moves of large values** — moving a `Box<[u8; 1_000_000]>` copies 8 bytes (the pointer), not a megabyte.

### Recursive types: the sizing problem

Consider a binary-tree node, the kind of variant type you'd write as an `enum` in Swift or a `type` in OCaml:

```rust
enum Tree {
    Leaf(i32),
    Branch(Tree, Tree), // does NOT compile
}
```

The compiler must compute `size_of::<Tree>()` at compile time to lay the type out. `Branch` contains two `Tree`s — the size equation `size(Tree) = size(i32) + 2 * size(Tree) + tag` has no finite solution.

> **⚠️ Pitfall →** This is `error[E0072]: recursive type \`Tree\` has infinite size`. rustc even suggests the fix: "insert some indirection (e.g., a `Box`, `Rc`, or `&`)". The point of indirection is that a pointer has a *fixed* size (8 bytes on a 64-bit target) regardless of what it points at. Box the recursive field:

```rust
enum Tree {
    Leaf(i32),
    Branch(Box<Tree>, Box<Tree>), // compiles: each Branch is i32-ish + two pointers
}

let t = Tree::Branch(
    Box::new(Tree::Leaf(1)),
    Box::new(Tree::Branch(Box::new(Tree::Leaf(2)), Box::new(Tree::Leaf(3)))),
);
```

> **⚙️ Under the hood →** In Swift, a class instance is always a reference (a pointer behind the scenes), so a recursive class never has a sizing problem — you never think about it. An `indirect enum` in Swift adds exactly the boxing Rust forces you to write by hand. Rust unboxes by default: a plain `enum` is laid out inline as a tag plus the largest variant, like a tagged union in C. That is why *you* must insert the indirection. `size_of::<Tree>()` becomes the size of the tag plus two pointer-sized fields; the actual subtrees live on the heap, "next to one another" rather than nested inside one another.

> **🔧 In practice →** You hit this the first time you model a real recursive shape — a JSON value, an expression tree for a small interpreter, a linked list. Say you're writing a tiny calculator and parse `2 * (3 + 4)` into an AST: `enum Expr { Num(f64), Add(Box<Expr>, Box<Expr>), Mul(Box<Expr>, Box<Expr>) }`. The `Box` on each operand is what lets `Mul` hold an `Add` hold two `Num`s without the type being infinitely large, and `Box::new` is where each subtree gets its own heap slot. Drop the root and the whole tree frees itself bottom-up.

> **🎓 Tripos link →** Foundations of Computer Science: the OCaml `type 'a tree = Leaf | Node of 'a tree * 'a * 'a tree` you wrote there is your `Box`ed enum — OCaml's runtime just does the boxing silently. Compiler Construction: the inability to size a recursive non-indirected type is exactly why every real compiler stores recursive ADTs as pointers in its intermediate representation. Rust makes that indirection a visible, named choice in the source instead of a runtime default.

## `Deref` and deref coercion: making a wrapper transparent

What makes `Box<T>` feel like a `T` rather than an opaque struct? The `Deref` trait. Implementing it lets you customise the unary `*` operator.

```rust
use std::ops::Deref;

struct Holder<T>(T);

impl<T> Deref for Holder<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.0
    }
}

let h = Holder(7);
assert_eq!(*h, 7);
```

`deref` returns a *reference* (`&T`), not the value — returning `T` by value would move the inner data out of `self`, which you almost never want. So `*h` expands to `*(h.deref())`: call `deref` to get `&T`, then apply the built-in dereference. The substitution fires exactly once per `*`, so it terminates — `*h` here lands on an `i32` and stops.

`Deref` matters far beyond `*` because of **deref coercion**: when you pass `&Wrapper` where the function wants `&Target`, the compiler silently inserts as many `deref` calls as needed to bridge the gap. The canonical case is `String → str`:

```rust
fn greet(name: &str) {
    println!("hi, {name}");
}

let owned = String::from("ada");
greet(&owned);  // &String coerced to &str via String: Deref<Target = str>
```

`&owned` is `&String`, but `String: Deref<Target = str>`, so `&String → &str` automatically. With a `Box<String>` it would chain twice: `&Box<String> → &String → &str`. This is why APIs take `&str` and `&[T]` (the borrowed [slice](05-slices-and-duality.md) forms) rather than `&String` and `&Vec<T>` — callers with the owned form get coerced for free, and callers with a literal `&str` pay nothing.

> **🦀 From your toolbox →** This is the autoderef chain from [method resolution](06-structs-and-methods.md), now generalised to function arguments. The closest thing you've used is Swift's implicit bridging — passing a `String` where an `NSString` is wanted, or the way a subclass passes where a superclass is expected — and Java's reference widening (a `String` flows into an `Object` parameter). The crucial difference is that Rust's coercion is deliberately narrow: it fires only across `Deref`/`DerefMut` impls, only on references, and only to reach a known target type — never to manufacture a brand-new value out of thin air, so it can't surprise you the way an implicit numeric conversion can.

There are exactly three coercion directions, and the asymmetry is load-bearing:

- `&T → &U` when `T: Deref<Target = U>`
- `&mut T → &mut U` when `T: DerefMut<Target = U>`
- `&mut T → &U` when `T: Deref<Target = U>`

The missing fourth case — `&U → &mut U` — is forbidden, not arbitrarily. A `&mut` is, by the borrow rules, the *unique* live reference to its data. Going `&mut → &` is always sound: you demote unique access to shared, dropping a capability. Going `& → &mut` would require *proving* the shared reference you hold is the only one, which a `&`'s type cannot guarantee. So the compiler refuses it.

> **⚙️ Under the hood →** Deref coercion is resolved entirely at compile time. The number of `deref` calls is fixed during type checking and inlined, so a coerced call is identical machine code to the hand-written `&(*x)[..]` you would otherwise type. Zero runtime penalty — coercion is pure ergonomics, not a dynamic dispatch.

## `Drop`: the destructor

`Drop` is the cleanup half of RAII. Implement one method, `drop(&mut self)`, and the compiler inserts a call to it when the value goes out of scope.

```rust
struct Connection {
    id: u32,
}

impl Drop for Connection {
    fn drop(&mut self) {
        println!("closing connection {}", self.id);
    }
}

fn main() {
    let _a = Connection { id: 1 };
    let _b = Connection { id: 2 };
} // prints "closing connection 2" then "closing connection 1"
```

Values drop in **reverse order of declaration** — `_b` before `_a` — because scopes nest like a stack. This is the deterministic, move-aware drop from [ownership](02-ownership-and-moves.md): a value is dropped exactly once, at the end of its final owner's scope, and a moved-from value is *not* dropped (the compiler statically tracks that its ownership left).

> **🔧 In practice →** `Drop` is how you write a "this resource is released when this variable dies" guard, which in Java you'd express with `try-with-resources` and in Swift with a manual `defer` or a `deinit`. A file handle, a database connection from a pool, a held lock — wrap it in a type with a `Drop` impl and the cleanup is automatic at scope exit, even on an early `return` or a `?`-propagated error. The standard library's `MutexGuard` works exactly this way: you `lock()`, use the data, and the lock releases the moment the guard leaves scope — no explicit `unlock()` to forget.

Two rules to watch for:

- **You cannot call `.drop()` manually.** `c.drop()` is `error[E0040]: explicit use of destructor method`. If Rust let you, it would still run the automatic drop at end of scope, producing a double-free — exactly the bug the ownership system exists to prevent.
- **To drop early, use the free function `std::mem::drop`** (in the prelude): `drop(c)`. It simply takes `c` *by value*, moving ownership in, and the value dies at the end of `drop`'s body. Because ownership moved, the compiler knows not to drop `c` again at end of scope.

```rust
let c = Connection { id: 3 };
drop(c);                      // closes now
// c is moved-out; not dropped again at scope end
```

> **🎓 Tripos link →** Programming in C and C++: this is RAII and the rule of "the destructor runs once, deterministically, at scope exit". The delta from C++ is that the *move* is type-checked — in C++ a moved-from object is left in a valid-but-unspecified state and its destructor still runs; in Rust the moved-from binding is statically dead and has no destructor call at all. There is no "valid but unspecified" leftover object to reason about.

## `Rc<T>`: shared ownership by reference counting

Single ownership is the common case, but sometimes a value is genuinely owned by several places at once — a node referenced by many edges in a graph, an immutable config shared across subsystems. You cannot express this with `Box` (one owner) or `&` (a borrow needs a single owner to outlive it, forcing lifetime gymnastics). `Rc<T>` — *reference counted* — keeps a count of owners and frees the data when the count hits zero.

```rust
use std::rc::Rc;

let a = Rc::new(vec![1, 2, 3]);
let b = Rc::clone(&a);          // count: 1 -> 2, no deep copy
let c = Rc::clone(&a);          // count: 2 -> 3
println!("count = {}", Rc::strong_count(&a)); // 3
// each Rc dropped decrements; allocation freed when count reaches 0
```

`Rc::clone` does **not** deep-copy the `Vec` — it bumps a `usize` counter and hands back another pointer to the same heap allocation. The idiom is to write `Rc::clone(&a)` rather than `a.clone()`, even though both work, precisely *because* the explicit form signals "this is a cheap refcount bump, not an expensive deep clone" to a reader scanning for performance problems.

There is no manual decrement — `Rc`'s own `Drop` impl decrements as each handle leaves scope. When `strong_count` reaches 0, the inner value is dropped and the heap freed.

> **🦀 From your toolbox →** This *is* Swift's ARC and Python's reference counting, with the bookkeeping made visible. In Swift you never write the retain/release calls; in Python the interpreter bumps the count on every assignment. `Rc` makes you say `Rc::clone` to bump it, which is the point — sharing becomes a deliberate, greppable act. One crucial difference from both: **`Rc`'s count is non-atomic**, so the compiler forbids moving it across threads (`Rc` is not `Send`/`Sync`). Swift's ARC and Python's GIL-protected count are always thread-safe-ish, paying that cost everywhere. Rust splits the choice: `Rc<T>` for single-threaded (a cheap increment), `Arc<T>` for cross-thread (an atomic increment). The API is identical; only the synchronisation cost differs.

> **🎓 Tripos link →** Concurrent and Distributed Systems: an atomic increment forces every core to agree on the new count, which under contention bounces a cache line between CPUs. `Rc` exists so single-threaded code does not pay that tax. We return to `Arc<T>` — the thread-safe twin — in [fearless concurrency](13-fearless-concurrency.md), where `Send + Sync` are the bounds that decide which one you may use.

The critical limitation: `Rc<T>` gives you **shared, immutable** access. `Rc<T>` derefs to `&T`, never `&mut T`. If you could get `&mut T` out of one of several `Rc` handles, you would have a mutable reference aliased by other handles — a data race in the making, exactly the situation the borrow rules forbid. To mutate shared data you must combine `Rc` with interior mutability, which is the next piece.

## Interior mutability: `RefCell<T>` and the runtime borrow check

Rust's defining rule, from [borrowing](03-references-and-borrowing.md): at any moment a piece of data has *either* one `&mut` *or* any number of `&`, never both — aliasing XOR mutability. The borrow checker proves this statically. But static analysis is necessarily conservative — no checker can accept *every* safe program — so some genuinely-safe programs are rejected.

*Interior mutability* is the pattern for those programs: a type that lets you mutate its contents through a shared `&`, with the aliasing rule still enforced — but **dynamically, at run time, by the type itself**. `RefCell<T>` is the workhorse.

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);

// no `mut` on `cell` — yet we mutate through a shared borrow:
cell.borrow_mut().push(4);

let len = cell.borrow().len();
println!("{len}"); // 4
```

`RefCell<T>` owns a single `T` (like `Box`, single owner — not multiple like `Rc`). Instead of `&` and `&mut`, you call:

- `borrow()` → `Ref<T>`, a smart pointer that derefs to `&T`. Increments an internal "shared borrows active" counter; decrements on drop.
- `borrow_mut()` → `RefMut<T>`, derefs to `&mut T`. Marks the cell exclusively borrowed.

The cell tracks these at run time and enforces the same XOR rule the compiler would: many `Ref`s, or one `RefMut`, never both. Violate it and you do not get UB — you get a **panic**.

```rust
let cell = RefCell::new(0);
let r1 = cell.borrow_mut();
let r2 = cell.borrow_mut(); // PANIC: already borrowed: BorrowMutError
```

> **⚠️ Pitfall →** This code *compiles* — the borrow checker cannot see the violation, since both `borrow_mut` calls are just method calls returning a guard. At run time the second one panics with `already borrowed: BorrowMutError`. The cure is to scope your guards tightly: let the `RefMut` from the first call drop (end its statement or open a `{ }` block) before requesting the second. The bug class here is "I'm holding a guard longer than I think" — the equivalent of forgetting to release a lock.

> **🦀 From your toolbox →** Think of `RefCell` as the disciplined version of mutating shared state in Swift or Python. In those languages a method on a shared object can quietly mutate it and the language never checks whether someone else is reading at the same time — you find out by debugging a corrupted value. `RefCell` gives you the same "mutate through a shared handle" power but verifies the aliasing-XOR-mutability rule *dynamically* and fails loudly with a panic the instant two accesses overlap. The cost is small: a couple of `usize` fields and a branch per borrow.

> **⚙️ Under the hood →** `RefCell<T>` is `{ borrow_flag: Cell<isize>, value: UnsafeCell<T> }`. `UnsafeCell<T>` is the *only* legitimate way in Rust to get a `*mut T` from a `&T` — it is the primitive that tells the compiler "do not assume the data behind this `&` is immutable." `borrow()`/`borrow_mut()` are a safe API wrapping `unsafe` code: they check the flag, hand out the pointer, and the `Ref`/`RefMut` guards restore the flag on drop. The "safe outside, unsafe inside" sandwich is exactly how all of `std`'s sound abstractions over raw memory are built (see [unsafe](16-unsafe-and-ffi.md)).

> **🔧 In practice →** The classic case is when an interface fixes a method to `&self` but your implementation needs to remember something. Picture writing a test double for a `Logger` trait: the trait says `fn log(&self, line: &str)` — you can't change it to `&mut self` to suit your one mock — yet your mock needs to record every line so the test can assert on it later. `RefCell` is the bridge.

```rust
use std::cell::RefCell;

trait Logger {
    fn log(&self, line: &str); // &self is fixed by the interface
}

struct Capture {
    lines: RefCell<Vec<String>>,
}

impl Logger for Capture {
    fn log(&self, line: &str) {
        self.lines.borrow_mut().push(line.to_owned()); // mutate through &self
    }
}
```

This is the standard pattern for test doubles, mocks, and observers in Rust: the interface is immutable, the implementation needs to mutate, `RefCell` bridges the two. The same shape shows up for memoization caches and lazy-initialised fields — anywhere a `&self` method legitimately needs to update hidden state.

### `Cell<T>`: the cheaper sibling

`Cell<T>` is interior mutability without handing out references at all. Instead of `borrow`/`borrow_mut`, you `get()` a copy out or `set()` a whole new value in (or `replace`/`take`). Because it never lends a reference into the interior, there is nothing to alias, so it needs **no runtime flag and never panics**.

```rust
use std::cell::Cell;

let counter = Cell::new(0u32);
counter.set(counter.get() + 1); // get/set, no borrow guards
```

Use `Cell<T>` for small `Copy` values (counters, flags) where copy-in/copy-out is fine; use `RefCell<T>` when you need a reference to a non-`Copy` interior (a `Vec`, a `String`) and are willing to pay for the runtime borrow tracking.

## The `Rc<RefCell<T>>` pattern: shared *and* mutable

Now combine the two. `Rc<T>` = multiple owners but immutable. `RefCell<T>` = single owner but mutable through `&`. Nest them — `Rc<RefCell<T>>` — and you get **multiple owners of mutable data**, the building block for graphs, adjacency lists, observer registries, and any shared-mutable structure. If you've built a graph in Java with shared object references, or in Swift with classes pointing at each other, this is the Rust spelling of that same shape.

```rust
use std::cell::RefCell;
use std::rc::Rc;

let shared = Rc::new(RefCell::new(0));

let owner_a = Rc::clone(&shared);
let owner_b = Rc::clone(&shared);

*owner_a.borrow_mut() += 10;   // mutate via one handle...
*owner_b.borrow_mut() += 5;    // ...and another

println!("{}", shared.borrow()); // 15 — all three see the same cell
```

`Rc` provides the shared ownership; `RefCell` provides the mutability the `Rc` alone would deny you; the runtime borrow check inside `RefCell` keeps the aliasing rule intact even though three handles point at the same data. Read the type inside-out: "a reference-counted, shared handle to a runtime-borrow-checked mutable `T`."

> **🎓 Tripos link →** Type safety, in plain terms: `Rc<RefCell<T>>` is a worked example of *trading a compile-time guarantee for a run-time check without losing safety*. The compiler normally proves "no aliased mutation" before the program runs; here that same fact is checked again at each `borrow`/`borrow_mut`, and the program aborts with a panic if the check fails. Nothing about safety is lost — still no undefined behaviour, still no data race — only the *moment* of checking (run time, not compile time) and the *failure mode* (a clean panic instead of a compile error) change.

## Reference cycles leak — and `Weak<T>` breaks them

Here is the one guarantee Rust does **not** make: it does not promise the absence of memory leaks. A leak is memory-*safe* (no dangling, no UB), so it is outside the ownership system's mandate. `Rc<RefCell<T>>` gives you the rope to hang yourself: build a cycle of `Rc`s pointing at each other and the strong counts never reach zero, so the destructors never run.

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Node {
    next: RefCell<Option<Rc<Node>>>,
}

let a = Rc::new(Node { next: RefCell::new(None) });
let b = Rc::new(Node { next: RefCell::new(None) });

*a.next.borrow_mut() = Some(Rc::clone(&b)); // a -> b, b's count = 2
*b.next.borrow_mut() = Some(Rc::clone(&a)); // b -> a, a's count = 2
// drop a and b at scope end: each count drops 2 -> 1, never 0. LEAKED.
```

When `a` and `b` go out of scope their *local* handles drop, taking each count from 2 to 1 — but each node still holds an `Rc` to the other, so neither hits zero, neither `Node` is freed, and the two allocations are stranded forever. This is the same trap that bites Swift programmers who create a strong reference cycle between two ARC objects, and for the same reason: plain reference counting cannot collect cycles. (A tracing garbage collector like Java's *can*, which is the one thing it buys you here.)

The fix is `Weak<T>` — a *non-owning* reference, the analogue of Swift's `weak` reference. A `Weak<T>` points into an `Rc`'s allocation but does **not** contribute to the strong count, so it does not keep the value alive. You create one with `Rc::downgrade(&rc)`. Because the pointee may already be dead, you cannot dereference a `Weak` directly: you call `.upgrade()`, which returns `Option<Rc<T>>` — `Some` if the value is alive (bumping the strong count while you hold the result), `None` if it was dropped. The `Option` is the type system forcing you to handle the dangling case; there is no path to a raw, possibly-dangling pointer.

The canonical use is a tree where children own nothing upward but need to find their parent. Ownership must point *down* (a parent owns its children; dropping the parent drops the subtree), so the upward edge must be weak:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,      // up: non-owning, breaks the cycle
    children: RefCell<Vec<Rc<Node>>>, // down: owning
}

let leaf = Rc::new(Node {
    value: 3,
    parent: RefCell::new(Weak::new()),    // no parent yet -> empty Weak
    children: RefCell::new(vec![]),
});

let branch = Rc::new(Node {
    value: 5,
    parent: RefCell::new(Weak::new()),
    children: RefCell::new(vec![Rc::clone(&leaf)]), // branch owns leaf
});

*leaf.parent.borrow_mut() = Rc::downgrade(&branch); // leaf points up, weakly

// leaf -> branch is weak, so:
//   branch: strong 1, weak 1
//   leaf:   strong 2 (held by `leaf` + branch.children), weak 0
assert_eq!(Rc::strong_count(&branch), 1);
assert_eq!(Rc::weak_count(&branch), 1);

// access the parent safely:
let parent_value = leaf.parent.borrow().upgrade().map(|p| p.value);
assert_eq!(parent_value, Some(5));
```

`Rc<T>` carries two counts: `strong_count` (owners — drives deallocation) and `weak_count` (outstanding `Weak`s — does *not*). When `branch` is dropped its strong count falls to 0 and its `Node` is freed *despite* `leaf.parent` still holding a `Weak` to it; afterwards `leaf.parent.borrow().upgrade()` simply returns `None`. No cycle, no leak, no dangling — the `Weak` cleanly observes its target's death.

> **🦀 From your toolbox →** This is exactly Swift's `weak` / `unowned` discipline. In Swift you mark the back-edge (child→parent, delegate→owner, observer→subject) `weak` so it doesn't pin the object alive, then unwrap the optional when you actually use it. `Weak<T>` is the same idea: make the back-edge a `Weak`, and `.upgrade()` it to a real owner only for the duration of use. The one improvement over Swift is that a Swift `weak var` silently becomes `nil` and it's easy to forget the unwrap; Rust's `upgrade()` returns `Option<Rc<T>>` and the compiler *forces* you to handle the `None` branch before you can touch the value.

> **⚙️ Under the hood →** An `Rc<T>` allocation is one heap block: `{ strong: Cell<usize>, weak: Cell<usize>, value: T }`. `strong` reaching 0 runs `T`'s destructor and the value's storage is logically gone; the *block itself* is freed only once `weak` also reaches 0 (any live `Weak` must keep the counts readable so `upgrade` can check them). `Arc<T>` is byte-identical except both counts are atomics. This is why a leaked cycle wastes the full `T` plus both counters — and why a `Weak`-only "leak" still costs you the two-`usize` header until the last `Weak` dies.

## When to reach for which

```text
Need              ┌────────────────────────────────────────────────────────┐
one owner,        │ Box<T>        heap; recursive types; trait objects;     │
move it cheaply   │               cheap large moves                         │
                  ├────────────────────────────────────────────────────────┤
many owners,      │ Rc<T>         refcount (non-atomic); single-threaded;   │
read-only         │               immutable access only                    │
                  ├────────────────────────────────────────────────────────┤
many owners,      │ Arc<T>        refcount (atomic); cross-thread           │
across threads    │               (see ch.13)                               │
                  ├────────────────────────────────────────────────────────┤
mutate through    │ RefCell<T>    runtime borrow check, panics on violation;│
a shared &,       │ Cell<T>       Cell for Copy values (no guards, no panic)│
single thread     │                                                         │
                  ├────────────────────────────────────────────────────────┤
shared + mutable  │ Rc<RefCell<T>>  the graph/observer building block       │
                  ├────────────────────────────────────────────────────────┤
break a cycle /   │ Weak<T>       non-owning back-edge; .upgrade() -> Option │
non-owning edge   │                                                         │
                  └────────────────────────────────────────────────────────┘
```

Default to plain ownership and `&`/`&mut`. Every entry below `Box` is a step *away* from compile-time guarantees toward runtime cost or runtime failure — reach for it only when the static model genuinely cannot express what you need, never to silence the borrow checker. A `RefCell` that panics in production is a borrow-rule violation you shipped; the compile error you "fixed" by wrapping in `RefCell` was telling you something true.

## Mental-model recap

- A smart pointer is a struct that **owns** its pointee and implements `Deref` (acts like a reference) and usually `Drop` (RAII cleanup). `Box`/`Rc`/`Arc`/`RefCell` are `std`'s safe abstractions over heap and ownership.
- `Box<T>` = single owner on the heap with a fixed pointer size — the indirection that makes recursive types sizable and trait objects/large moves cheap.
- `Deref` + deref coercion is the autoderef chain extended to function arguments; it is fully compile-time, hence why APIs take `&str`/`&[T]`. `&U → &mut U` is the one coercion that is forbidden, because a `&` cannot prove it is unique.
- **The headline:** `Rc` + `RefCell` move borrow-rule enforcement from compile time to run time. `RefCell` checks aliasing-XOR-mutability dynamically and *panics* rather than risking the silent corruption you'd get from mutating freely shared state in Swift or Python.
- Rust prevents UB, not leaks: `Rc` cycles leak because strong counts never hit zero. `Weak<T>` is the non-owning back-edge that breaks them; `upgrade()` returns `Option<Rc<T>>` so the dangling case is type-checked away.

## Exercises

1. **Predict the layout.** Given `enum Json { Null, Bool(bool), Num(f64), Arr(Vec<Json>), Obj(Vec<(String, Json)>) }`, decide *without compiling* whether it builds with no `Box` at all, then justify it: what is `size_of::<Vec<Json>>()` and where does the recursion's indirection physically live? Contrast this with why `enum Tree { Leaf(i32), Branch(Tree, Tree) }` does *not* build — what's different about a field being a `Vec<Json>` versus a bare `Json`?

2. **Which compiles, and why?** For each line below, predict compile vs. runtime-panic vs. clean run, and state the reason in one sentence:
   ```rust
   let cell = RefCell::new(vec![1, 2, 3]);
   let a = cell.borrow();
   let b = cell.borrow();          // (i)
   let c = cell.borrow_mut();      // (ii)
   ```
   Then reorder the three so that all of them succeed, changing nothing but where each binding's scope ends.

3. **Choose the tool.** For each scenario, pick exactly one of `Box`, `Rc`, `Rc<RefCell<_>>`, or `Weak`, and defend the choice in a sentence: (a) a config object read by five subsystems and never modified after startup; (b) the parent pointer of a tree node, where parents own children; (c) the single payload of a `Box<dyn Error>` you return from a function; (d) a counter that several closures in the same thread each need to increment.

4. **Adapt the mock.** Take the `Capture` logger from this chapter and add a method `fn count(&self) -> usize` that returns how many lines were logged, plus `fn clear(&self)` that empties the buffer — both must keep the `&self` signature. Which `RefCell` methods do you call inside each, and is there any ordering of calls to `log`, `count`, and `clear` in a single statement that would panic? Explain. (★)

5. **Find the leak by counting.** Sketch a doubly linked list of three nodes A↔B↔C where `next` is `Rc<Node>` and `prev` is `Weak<Node>`, then write out the strong count of B at the moment all three local bindings drop, and conclude whether B is freed. Now mentally switch `prev` to `Rc<Node>`: redo B's strong count and explain exactly why no node's `Drop` ever runs.
