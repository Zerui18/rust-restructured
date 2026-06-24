# 13. Fearless Concurrency

You have written concurrent C with `pthread_mutex_t`, Java with `synchronized`, and argued about CSP in Concurrent & Distributed Systems. You know the central pain: the lock and the data it protects are two separate objects, linked only by a convention living in a comment, in your head, or in a wiki nobody reads. Forget the convention once — touch the data without the lock, or hand a raw pointer to another thread — and you get a data race: undefined behaviour, a heisenbug that survives review and reproduces only under load in production.

Rust's thesis for this chapter is bold and, once seen, obvious: **the ownership and borrowing rules that gave you memory safety in [chapter 2](02-ownership-and-moves.md) and [chapter 3](03-references-and-borrowing.md) give you data-race freedom *for the same price*.** There is no separate concurrency-safety subsystem. The borrow checker that rejected `&mut` aliasing in single-threaded code is the *same* checker that rejects sharing-without-synchronisation across threads. Two marker traits, `Send` and `Sync`, thread that guarantee through the type system. The community calls the result *fearless concurrency*: not because concurrency becomes easy, but because the entire class of bug that made it terrifying — the data race — is now a compile error.

One honest caveat up front, because it is the most common over-claim: **Rust prevents data races, not deadlocks, not race conditions in the general sense, not livelock.** You can still write two threads that lock A-then-B and B-then-A and wedge forever. The compiler proves the absence of *unsynchronised concurrent access*; it does not prove your synchronisation *logic* is correct.

## A data race, precisely

Borrowing the definition from Concurrent & Distributed Systems: a **data race** occurs when two or more threads access the same memory location concurrently, at least one access is a write, and there is no synchronisation (happens-before edge) ordering them. All three conditions are necessary. Remove any one — make all accesses reads, or serialise them behind a lock, or never share the location — and the race is gone.

Now compare to the single-threaded rule you already internalised. Rust's borrow checker enforces **aliasing XOR mutability**: at any program point a value has either any number of shared references `&T`, *or* exactly one exclusive reference `&mut T`, never both. Stare at that next to the data-race definition:

- "at least one access is a write" ⟺ you need `&mut T`,
- "two or more threads access the same location" ⟺ you need aliasing,
- and aliasing-XOR-mutability says you cannot have both at once.

The single-threaded rule that prevents iterator invalidation and use-after-free is *structurally the same* rule that prevents data races. The only missing piece is teaching the type system *which* types may cross a thread boundary at all, and which references may be shared across one. That is exactly what `Send` and `Sync` encode, and we get to them at the end.

> **🎓 Tripos link →** In CDS a data race is undefined behaviour at the memory-model level; the standard mitigations (locks, message passing) are *disciplines* the programmer must apply correctly. Rust reframes the discipline as a *typing judgement*. "This program has no data races" becomes a theorem the type checker discharges — a soundness property in the sense of Semantics of Programming Languages. The aliasing-XOR-mutability invariant is the lemma; data-race freedom is a corollary.

## Threads: `spawn`, `JoinHandle`, and why `'static`

Rust's standard library uses a 1:1 threading model — one OS thread per `std::thread`, the same processes-and-threads picture from Computer Architecture & OS. You create one by handing `thread::spawn` a closure:

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        let mut total = 0u64;
        for i in 1..=1_000 { total += i; }
        total
    });

    // ... main thread does its own work here ...

    let sum = handle.join().unwrap(); // block until the thread finishes
    println!("worker computed {sum}");
}
```

`spawn` returns a `JoinHandle<T>`, where `T` is the closure's return type. It is an owned handle to the running thread. Calling `.join()` blocks the current thread until the spawned one terminates and returns a `Result<T, _>` — `Err` if that thread panicked. Note that `join` returns the closure's value, so threads are not just fire-and-forget side-effect engines; they are computations you await.

If you never join, the handle is dropped and the thread is detached — and crucially, **when `main` returns, the process exits and all surviving spawned threads are killed mid-flight**, finished or not. The `JoinHandle` is your only lever to wait.

### The `'static` bound and why it must be there

Here is the signature that governs everything (slightly simplified):

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

Two bounds on the closure `F`: `Send` (we will return to it) and `'static`. The `'static` bound is the one that bites first, and the reason is a pure lifetime argument from [chapter 4](04-lifetimes.md). The spawned thread may run for *any* duration — it can outlive the stack frame that spawned it. So any reference the closure captures must be valid for the rest of the program, i.e. its region must be `'static`. A borrow of a local variable has a region that ends when that local goes out of scope; the compiler cannot prove the thread finishes first, so it refuses.

Watch it fail:

```rust
use std::thread;

fn main() {
    let labels = vec!["alpha", "beta", "gamma"];
    let handle = thread::spawn(|| {
        println!("{labels:?}"); // borrows `labels`
    });
    handle.join().unwrap();
}
```

```text
error[E0373]: closure may outlive the current function, but it borrows `labels`,
              which is owned by the current function
 --> src/main.rs:5:32
  |
5 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `labels`
6 |         println!("{labels:?}");
  |                    ------ `labels` is borrowed here
  |
note: function requires argument type to outlive `'static`
help: to force the closure to take ownership of `labels` ... use the `move` keyword
```

This is not pedantry about a case that "obviously" works because we join immediately. The danger is concrete. Imagine the main thread dropped `labels` right after spawning:

```rust
let labels = vec!["alpha", "beta", "gamma"];
let handle = thread::spawn(|| println!("{labels:?}"));
drop(labels);            // freed here
handle.join().unwrap();  // thread may only now read it — use-after-free
```

The OS scheduler decides when the spawned thread actually runs. It might not touch `labels` until after `drop` has freed the backing allocation. That is a use-after-free across threads — precisely the bug C lets you ship. Rust forbids the *borrow* in the first place, so the dangerous version cannot be written.

### `move`: transferring ownership across the boundary

The fix the compiler suggests is `move`. As in [chapter 12](12-closures-and-iterators.md), `move` forces the closure to capture by value rather than by reference. For `Vec<String>` that means the heap-owning struct is *moved* into the closure, and the closure (now owning `'static` data) satisfies the bound:

```rust
let labels = vec!["alpha", "beta", "gamma"];
let handle = thread::spawn(move || {
    println!("{labels:?}");
});
// labels is no longer usable here — it was moved into the new thread.
handle.join().unwrap();
```

Ownership has physically crossed the thread boundary. The main thread can no longer name `labels`, so the use-after-free is impossible *by construction*: there is no second owner to free it early.

> **🦀 From your toolbox →** `move` is C++'s `std::move` applied to a lambda's capture list — `[v = std::move(v)]() { ... }`. The difference is enforcement. In C++ nothing stops you capturing by reference (`[&]`) into a `std::thread` that outlives the referent; you get a dangling reference and a data race, and the compiler is silent. Rust's `'static` bound makes the C++ footgun a type error. The analogy breaks down on detach semantics too: a C++ `std::thread` destructor *terminates the program* if you forget to `join`/`detach`; Rust's `JoinHandle` quietly detaches on drop.

> **⚙️ Under the hood →** A `move` closure capturing a `Vec` is a compiler-synthesised struct holding the vector's three words (pointer, length, capacity) by value — no boxing for the capture itself. `spawn` then `Box`es that closure onto the heap and hands the box to the OS thread-creation call (`pthread_create` on this platform), because the new thread needs a stable, owned entry point. The `JoinHandle` wraps the OS thread id plus a channel for the return value. `join` is a `pthread_join` plus reading that value out.

## Message passing: channels as CSP

The first of Rust's two concurrency idioms is message passing, captured by the Go-inflected slogan "do not communicate by sharing memory; share memory by communicating." This is the channel — directly the channel of CSP from Concurrent & Distributed Systems, and the substrate of the actor model.

`std::sync::mpsc` gives you a channel. The name is the design: **multiple producer, single consumer.** `channel()` returns a `(Sender<T>, Receiver<T>)` pair.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let payload = String::from("ping");
        tx.send(payload).unwrap();
        // payload is gone here — ownership left through the channel.
    });

    let got = rx.recv().unwrap();
    println!("received: {got}");
}
```

`tx.send(v)` enqueues `v`; it returns `Result<(), SendError<T>>`, erroring only if the receiver has already been dropped (the channel is *closed* when either half drops). `rx.recv()` blocks until a value arrives or the channel closes, returning `Result<T, RecvError>`. There is also `try_recv` for a non-blocking poll, useful when the consumer has other work between checks.

The deep point is what the type system does to `send`. Its signature is `fn send(&self, t: T) -> Result<...>` — it takes `T` **by value**. So sending *moves ownership of the payload into the channel*, and from there to the receiver. The classic CSP intuition "the message is now theirs, not mine" is enforced as an ownership transfer. Try to keep using a sent value and you get the same E0382 that any move-after-use produces:

```rust
thread::spawn(move || {
    let payload = String::from("ping");
    tx.send(payload).unwrap();
    println!("still mine? {payload}"); // ERROR: borrow of moved value
});
```

```text
error[E0382]: borrow of moved value: `payload`
  |
  | tx.send(payload).unwrap();
  |         ------- value moved here
  | println!("still mine? {payload}");
  |                        ^^^^^^^ value borrowed here after move
```

There is *no separate rule* for channels. "After you send a value you may not touch it" — the CSP correctness condition you would otherwise enforce by discipline — falls straight out of move semantics. Single ownership *is* the channel discipline.

`Receiver<T>` is an iterator: `for msg in rx` blocks for each message and ends cleanly when the channel closes. For multiple producers, **clone the sender** — each clone is an independent handle into the same queue:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    for worker in 0..3 {
        let tx = tx.clone();           // one sender per producer thread
        thread::spawn(move || {
            tx.send(format!("done by worker {worker}")).unwrap();
        });
    }
    drop(tx); // drop the original so the channel can close once workers finish

    for result in rx {                 // ends when all senders are dropped
        println!("{result}");
    }
}
```

The `drop(tx)` is the subtle bit a beginner misses and you should not: the `for result in rx` loop terminates only when *every* `Sender` is dropped. The original `tx` is still alive in `main`'s scope, so without the explicit `drop` the loop would block forever waiting on a producer that will never send. This is a *liveness* bug, not a safety bug — exactly the kind Rust does **not** catch for you.

> **🎓 Tripos link →** This is CSP made concrete. A `Sender`/`Receiver` pair is a typed channel `c : chan T`; `send` is `c!v` and `recv` is `c?x`. The `mpsc` restriction — many writers, one reader — mirrors a process algebra where output capability is shared but input is owned by a single process. The actor model is the natural specialisation: give each actor an `rx`, hand out clones of its `tx`, and its `recv` loop is the actor's behaviour. Because the message is *moved*, there is no aliasing between sender and receiver after the handover — the no-shared-state property actors rely on for race-freedom is, again, just ownership.

## Shared state: `Mutex<T>` owns its data

Message passing models single ownership flowing through pipes. Sometimes you genuinely want **multiple threads touching one piece of state** — a shared counter, a cache. That is multiple ownership plus mutation, and it is where the lock-versus-data separation in C and Java does its damage.

Rust's answer is `Mutex<T>`, and the design decision that makes it different is in the type itself: **the mutex *contains* the data.** You do not have a `Mutex` over here and an `i32` over there bound by a comment. You have a `Mutex<i32>`, and the only way to reach the `i32` is through the lock.

```rust
use std::sync::Mutex;

fn main() {
    let cell = Mutex::new(0);
    {
        let mut guard = cell.lock().unwrap(); // acquire; blocks until held
        *guard += 10;                         // guard derefs to &mut i32
    } // guard dropped here -> lock released automatically
    println!("{cell:?}"); // Mutex { data: 10, .. }
}
```

Three things to read carefully:

1. **You cannot reach the data without locking.** The type is `Mutex<i32>`, not `i32`. There is no method to grab the inner value except `lock`. The "always lock before you touch the data" rule is not a convention you can forget — it is the only API that exists. This is the single most important sentence in the chapter: in C and Java, lock and data are separate and the link is *discipline*; in Rust the link is *the type*.

2. **`lock()` returns a `MutexGuard<T>`, an RAII handle.** `guard` derefs (via `Deref`/`DerefMut`) to `&mut i32`, so you mutate through it. When `guard` goes out of scope its `Drop` impl releases the lock. You cannot forget to unlock for the same reason you cannot forget to free a `Box` — release is destruction, and destruction is automatic. This is precisely the RAII / scope-bound resource management from Programming in C and C++, applied to a lock instead of a heap allocation.

3. **`lock()` returns a `Result`.** If a thread panics while holding the guard, the mutex becomes *poisoned* — subsequent `lock()` calls return `Err(PoisonError)`. The rationale: a panic mid-critical-section may have left the protected invariant broken, so the lock refuses to silently hand out possibly-corrupt state. `.unwrap()` propagates that as a panic; in production you might inspect the error and call `.into_inner()` to recover the data deliberately.

> **🦀 From your toolbox →** `MutexGuard` is `std::lock_guard`/`std::unique_lock` from C++ RAII — lock on construction, unlock on destruction. But C++'s `std::mutex` guards *nothing*; it is a bare lock and you pair it with separate data by convention, exactly the error-prone separation Rust removes. Java's `synchronized(obj)` and `ReentrantLock` are worse on this axis: the lock and the fields it "protects" are related only by programmer intent, with zero compiler enforcement, and Java mutexes are reentrant whereas `std::sync::Mutex` is *not* — locking it twice on one thread deadlocks. The win Rust adds over both is that *the type forbids unsynchronised access at all*, not merely automates the unlock.

> **⚠️ Pitfall →** A `MutexGuard` holds the lock for its *entire lifetime*, which is until end of scope, not until last use. So `let total = *m.lock().unwrap() + other.lock().unwrap()...` or holding a guard across an `.await` or a long computation keeps everyone else blocked. Worse, locking the *same* mutex twice in one expression deadlocks instantly: `*m.lock().unwrap() += *m.lock().unwrap();` acquires, then tries to re-acquire a non-reentrant lock the same thread already holds. Fix: bind the guard to a name, scope it tightly with an explicit `{ }` block, and copy the value out (`let n = *m.lock().unwrap();`) so the guard drops before you do further work.

### Sharing the mutex itself: `Arc<T>`

`Mutex<T>` lets many threads *mutate* the data — but how do many threads each *own* a reference to the same mutex, given the `'static` move bound on `spawn`? You cannot move one `Mutex` into ten closures; it has a single owner. From [chapter 15](15-smart-pointers.md) you might reach for `Rc<T>`, the shared-ownership reference-counted pointer. Try it and the compiler stops you with a different, deeper error:

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let mut handles = vec![];
    for _ in 0..10 {
        let counter = Rc::clone(&counter);
        handles.push(thread::spawn(move || {       // ERROR
            *counter.lock().unwrap() += 1;
        }));
    }
    for h in handles { h.join().unwrap(); }
    println!("{}", *counter.lock().unwrap());
}
```

```text
error[E0277]: `Rc<Mutex<i32>>` cannot be sent between threads safely
   |
   | handles.push(thread::spawn(move || {
   |              ------------- ^ `Rc<Mutex<i32>>` cannot be sent between threads safely
   |
   = help: the trait `Send` is not implemented for `Rc<Mutex<i32>>`
note: required by a bound in `spawn`
```

`Rc<T>` updates its reference count with **plain, non-atomic** increments and decrements — fast, because single-threaded code needs no synchronisation. But two threads doing `clone`/`drop` concurrently would race on that count: a lost update could free the value while a clone still points at it (use-after-free) or leak it forever. So `Rc<T>` is deliberately `!Send`, and `spawn`'s `Send` bound rejects it at compile time. **The data race on the refcount is caught before the program runs**, even though you never wrote the racing code yourself — you only tried to share a type that *could* race.

The fix is `Arc<T>` — *atomically* reference-counted. Identical API to `Rc`, but the count is maintained with atomic fetch-add/fetch-sub, so concurrent clone/drop is safe. You pay for the atomics (memory fences, contended cache lines) only when you opt in:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    for _ in 0..10 {
        let counter = Arc::clone(&counter);     // bump the atomic refcount
        handles.push(thread::spawn(move || {
            let mut n = counter.lock().unwrap();
            *n += 1;
        }));
    }
    for h in handles { h.join().unwrap(); }
    println!("Result: {}", *counter.lock().unwrap()); // Result: 10
}
```

The idiom to memorise is **`Arc<Mutex<T>>`**: `Arc` provides shared *ownership* across threads, `Mutex` provides synchronised *mutation* of the shared data. They compose because they solve orthogonal problems. (Compare the single-threaded analogue from [chapter 15](15-smart-pointers.md): `Rc<RefCell<T>>` — shared ownership plus interior mutability, but with runtime borrow-checking instead of locking, and no thread safety.)

> **⚙️ Under the hood →** `Arc<T>`'s heap allocation holds two counts (strong and weak) as `AtomicUsize` plus the `T`. `clone` is a `fetch_add(1, Relaxed)`; `drop` is a `fetch_sub(1, Release)` and, when it hits zero, an `Acquire` fence before running `T`'s destructor — the release/acquire pair establishes the happens-before edge so the last dropper sees all prior writes. `Mutex<T>` on this platform wraps an OS futex (a `pthread_mutex_t`-equivalent) alongside the `T` and a poison flag. `Mutex::new` is not `const`-allocated separately; the lock and data sit in one contiguous allocation, which is *why* the lock can guard the data without indirection.

> **🦀 From your toolbox →** `Arc<T>` is C++'s `std::shared_ptr<T>` with atomic refcounting — but `shared_ptr` only makes the *control block* thread-safe, never the pointee. Two threads writing through one `shared_ptr<vector<int>>` is a data race C++ permits silently. Rust splits the two guarantees cleanly: `Arc` makes *ownership* shareable, and you must additionally wrap the data in `Mutex`/`RwLock`/an atomic to make *access* shareable — and the type system refuses to let you skip that step.

### `RwLock<T>` and atomics, briefly

When reads vastly outnumber writes, `RwLock<T>` lets *many* readers (`.read()`) share access simultaneously while writers (`.write()`) get exclusivity — the classic readers-writer lock from CDS, with the same aliasing-XOR-mutability shape: many `&T` or one `&mut T`, now enforced at runtime by the lock. For a single integer or flag, skip the mutex entirely and use `std::sync::atomic` types like `AtomicUsize` or `AtomicBool`, which expose lock-free `fetch_add`, `compare_exchange`, etc., parameterised by the memory `Ordering` (`Relaxed`/`Acquire`/`Release`/`SeqCst`) you studied in the memory-consistency part of CDS.

## `Send` and `Sync`: the proof, encoded as types

Everything above is enforced by two marker traits in `std::marker`. They have **no methods** — they are pure type-level predicates, like a refinement the type carries, and they are *auto-derived*: a type is `Send`/`Sync` iff all its fields are.

- **`Send`** — "ownership of a `T` may be transferred to another thread." `spawn` requires its closure (hence everything captured) to be `Send`.
- **`Sync`** — "a `&T` may be shared across threads," defined precisely as: `T: Sync` ⟺ `&T: Send`. If you can safely send a shared reference, the type is safe to share.

Almost everything is `Send + Sync` because almost everything is built from `Send + Sync` parts. The interesting cases are the *negative* ones, and each negative encodes exactly one would-be data race:

- `Rc<T>` is `!Send` and `!Sync` — its non-atomic refcount races, as we saw.
- `RefCell<T>` (and `Cell<T>`) is `!Sync` — its borrow flag is a non-atomic runtime check that two threads could corrupt. It *is* `Send` (you may move it to one other thread), just not `Sync` (you may not let two threads share `&RefCell`).
- Raw pointers `*const T`/`*mut T` are neither, because the compiler knows nothing about what they alias.
- `Arc<T>` is `Send + Sync` *provided* `T: Send + Sync`; `Mutex<T>` is `Sync` whenever `T: Send`, because the lock turns any `Send` payload into something safely shareable.

This is the whole proof, and it is genuinely a proof in the Logic and Proof / Curry-Howard sense. Read the trait bounds as propositions: `spawn`'s `F: Send` is the hypothesis "the captured state may cross a thread." The compiler discharges it by structural induction — `F` is `Send` because each field is, down to primitives which are axiomatically `Send`. When you wrote `Rc<Mutex<i32>>`, the induction *failed* at the `Rc` node: there is no derivation of `Rc: Send`, so there is no derivation of `closure: Send`, so the program does not type-check. The data-race-freedom theorem is *constructive* — its absence is a concrete missing implication the compiler points at.

> **🎓 Tripos link →** `Send`/`Sync` are the soundness machinery from Semantics of Programming Languages, specialised to concurrency. The metatheorem is roughly: *a well-typed program using only safe Rust has no data races.* `Send`/`Sync` are the typing-judgement side conditions that make the proof go through; they are auto-derived so the programmer rarely writes them, exactly as type inference (Foundations of CS) spares you from writing types. Implementing them by hand requires `unsafe` (see [chapter 16](16-unsafe-and-ffi.md)) — that is you stepping *outside* the checked fragment and assuming a proof obligation the compiler can no longer discharge for you.

> **⚙️ Under the hood →** `Send`/`Sync` are zero-cost: marker traits compile to nothing, carry no data, and have no vtable. They exist only during type-checking. After monomorphisation (Compiler Construction) they vanish entirely — there is no runtime representation of "this `Vec` is `Send`." The guarantee is paid for once, at compile time, and the generated machine code is identical to what an equivalent correct C program would emit.

## Beyond the standard library: scoped threads and rayon

Two practical escapes from boilerplate, both stable. **Scoped threads** (`std::thread::scope`) relax the `'static` bound: threads spawned inside a `scope` closure are guaranteed to be joined before `scope` returns, so the compiler *can* prove they do not outlive borrowed locals — meaning you may pass `&data` (not just `move`d owned data) into them without `Arc`:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4];
    thread::scope(|s| {
        s.spawn(|| println!("borrowed read: {:?}", &data[..2]));
        s.spawn(|| println!("borrowed read: {:?}", &data[2..]));
    }); // all scoped threads joined here; only now may `data` be moved/dropped
    println!("back in main, still own: {data:?}");
}
```

The scope's join-before-return guarantee *is* the lifetime argument that makes the borrows sound — it turns "may outlive" into "provably does not."

For data parallelism, the **`rayon`** crate is the community default: it turns `iter()` into `par_iter()` and parallelises the work across a thread pool with work-stealing, while the same `Send`/`Sync` bounds keep it data-race-free. A sequential `.map().sum()` becomes a parallel one by changing one method call — fearless concurrency at the iterator level from [chapter 12](12-closures-and-iterators.md).

## Mental-model recap

- A data race needs aliasing + a writer + no synchronisation. Aliasing-XOR-mutability — the *same* borrow rule that gives single-threaded memory safety — kills the first two simultaneously, so data-race freedom is not a new feature, it is a corollary of ownership.
- `thread::spawn` demands `Send + 'static` on its closure: `'static` because the thread may outlive its spawner (so no borrows of locals), `Send` because the captured state must be safe to move across threads. `move` transfers ownership in; `JoinHandle::join` waits and retrieves the result. Forgetting to join detaches, and `main` exiting kills detached threads.
- Channels (`mpsc`) make CSP/actor message passing into ownership transfer: `send` moves the value, so "don't touch it after sending" is just move-after-use. `Mutex<T>` makes lock-before-access into a *type* — the lock owns the data, the `MutexGuard` is RAII, release is automatic, and the lock can be poisoned by a panicking holder.
- `Arc<Mutex<T>>` is the canonical shared-mutable-state idiom: `Arc` = thread-safe shared ownership (atomic refcount, vs `Rc`'s racy non-atomic one), `Mutex` = synchronised mutation. They compose because they solve orthogonal problems.
- `Send`/`Sync` are auto-derived, method-less marker traits that *are* the data-race-freedom proof, discharged by structural induction over fields. `Rc`/`RefCell`/raw pointers are the instructive `!Send`/`!Sync` cases. Rust prevents races, **not** deadlocks or liveness bugs — your synchronisation logic is still your responsibility.

## Exercises

1. Write a program that spawns a thread capturing a `String` by reference (no `move`) and `drop`s the string in `main` before joining. Read the E0373/E0382 error. Now add `move`. Then *also* try to use the string in `main` after the `move` and explain, citing the lifetime argument, why exactly one of "borrow it" and "use it after moving" can ever succeed but never both.

2. Build a worker pool: spawn 4 threads, each cloning a shared `Sender`, sending its thread index down the channel. In `main`, collect all 4 via `for x in rx`. Run it — it hangs. Diagnose why (which `Sender` is still alive?) and fix it with a single `drop`. Explain whether the bug was a *safety* bug or a *liveness* bug, and why the compiler did not catch it.

3. (★) Take the `Arc<Mutex<i32>>` counter example and deliberately introduce a deadlock using *two* mutexes `a` and `b`, where one thread locks `a` then `b` and another locks `b` then `a`. Confirm it hangs. Then prove to yourself the borrow checker never complained — and write one sentence explaining precisely which guarantee Rust gives here and which it does not.

4. Try to send a `Rc<i32>` down an `mpsc` channel into a spawned thread. Predict the error trait (`Send`? `Sync`?) before compiling, then verify. Now wrap it in `Arc` instead and explain at the field level *why* the auto-derivation flips from failing to succeeding.

5. Hold a `MutexGuard` across a `thread::sleep` inside the critical section, with several threads contending. Measure (informally, via timestamps) how serialised they become. Then refactor to copy the protected value out and drop the guard before sleeping. Articulate the rule: a guard is held for its *scope*, not until its last use.

6. (★) Define `struct Counter(*mut i32);` and try to move it into a spawned thread. Observe that raw pointers are `!Send` so it fails. Then (reading ahead to [chapter 16](16-unsafe-and-ffi.md)) write `unsafe impl Send for Counter {}` and make it compile. Argue what proof obligation you have just taken on from the compiler, and construct an input where your `unsafe impl` causes a genuine data race — demonstrating that `unsafe` is you promising a theorem the checker can no longer verify.
