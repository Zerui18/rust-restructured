# 13. Fearless Concurrency

You have written concurrent Java with `synchronized` and `ReentrantLock`, touched threads in C, and argued about CSP in Concurrent & Distributed Systems. You know the central pain: the lock and the data it protects are two separate objects, linked only by a convention living in a comment, in your head, or in a wiki nobody reads. Forget the convention once — touch the data without the lock, or hand a raw pointer to another thread — and you get a data race: undefined behaviour, a heisenbug that survives review and reproduces only under load in production.

Rust's thesis for this chapter is bold and, once seen, obvious: **the ownership and borrowing rules that gave you memory safety in [chapter 2](02-ownership-and-moves.md) and [chapter 3](03-references-and-borrowing.md) give you data-race freedom *for the same price*.** There is no separate concurrency-safety subsystem. The borrow checker that rejected `&mut` aliasing in single-threaded code is the *same* checker that rejects sharing-without-synchronisation across threads. Two marker traits, `Send` and `Sync`, thread that guarantee through the type system. The community calls the result *fearless concurrency*: not because concurrency becomes easy, but because the entire class of bug that made it terrifying — the data race — is now a compile error.

One honest caveat up front, because it is the most common over-claim: **Rust prevents data races, not deadlocks, not race conditions in the general sense, not livelock.** You can still write two threads that lock A-then-B and B-then-A and wedge forever. The compiler proves the absence of *unsynchronised concurrent access*; it does not prove your synchronisation *logic* is correct.

## A data race, precisely

Borrowing the definition from Concurrent & Distributed Systems: a **data race** occurs when two or more threads access the same memory location concurrently, at least one access is a write, and there is no synchronisation ordering them. All three conditions are necessary. Remove any one — make all accesses reads, or serialise them behind a lock, or never share the location — and the race is gone.

Now compare to the single-threaded rule you already internalised. Rust's borrow checker enforces **aliasing XOR mutability**: at any program point a value has either any number of shared references `&T`, *or* exactly one exclusive reference `&mut T`, never both. Stare at that next to the data-race definition:

- "at least one access is a write" ⟺ you need `&mut T`,
- "two or more threads access the same location" ⟺ you need aliasing,
- and aliasing-XOR-mutability says you cannot have both at once.

The single-threaded rule that prevents iterator invalidation and use-after-free is *structurally the same* rule that prevents data races. The only missing piece is teaching the type system *which* types may cross a thread boundary at all, and which references may be shared across one. That is exactly what `Send` and `Sync` encode, and we get to them at the end.

> **🎓 Tripos link →** In Concurrent & Distributed Systems a data race is undefined behaviour at the memory-model level, and the standard mitigations (locks, message passing) are *disciplines* the programmer must apply correctly and can always forget. Rust's twist is to move that discipline from your head into the type checker: instead of you remembering to lock, the compiler refuses to compile any program that could race. The aliasing-XOR-mutability rule is the single idea doing all the work, and data-race freedom falls out of it as a consequence.

## Threads: `spawn`, `JoinHandle`, and why `'static`

Rust's standard library uses a 1:1 threading model — one OS thread per `std::thread`, the same processes-and-threads picture from Computer Architecture & Operating Systems. You create one by handing `thread::spawn` a closure:

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

> **🦀 From your toolbox →** Think of Swift capturing a value explicitly in a closure rather than capturing `self` weakly/strongly by reference — `move` says "this closure takes its own copy of the captured bindings." The crucial difference from the languages you know is *enforcement of the dangerous case*. In Java you can hand a `Runnable` a reference to an object and let it outlive everything that cared about it; the GC keeps the object alive, so you never get a dangling pointer — but nothing tells you the thread is now reading state you thought was done with. Rust's `'static` bound turns "this thread might read freed or stale local state" into a compile error. (Light C++ touch: it is the same hazard as capturing a stack variable by reference into a thread that outlives the frame — a dangling pointer the C++ compiler stays silent about.) Where the analogy breaks down: a Java thread keeps captured objects alive via GC, whereas in Rust ownership has genuinely *moved* — the original binding is gone, not merely still-referenced.

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

> **🔧 In practice →** Channels shine when you have a stream of *jobs* to hand to background workers. Picture an image-thumbnail service: the request handler should not block while images are resized. You spawn a pool of worker threads up front, each owning a clone of `rx` (via a shared, locked receiver) or — more idiomatically — own one receiver and fan jobs out. The producer side just moves the job in and forgets it:
> ```rust
> // request handler thread
> let job = ResizeJob { path, target_px };
> tx.send(job).unwrap();   // ownership of the job leaves; handler returns immediately
> // worker thread: `for job in rx { resize(job); }`
> ```
> Because `send` *moves* the job, there is zero shared mutable state between handler and worker, so no lock and no possibility of the handler accidentally mutating a job the worker is already processing. This is the pattern behind work queues, log/metrics pipelines, and actor-style event loops.

> **🎓 Tripos link →** This is the CSP channel from Concurrent & Distributed Systems made concrete: a `Sender`/`Receiver` pair is exactly a typed channel, `send` is the output `c!v` and `recv` is the input `c?x`. The `mpsc` restriction — many writers, one reader — matches the actor model, where many threads can post to an actor's mailbox but only the actor reads it. Because the message is *moved* rather than copied or shared, there is no aliasing between sender and receiver after the handover, which is precisely the no-shared-state property actors rely on to stay race-free.

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

2. **`lock()` returns a `MutexGuard<T>`, an RAII handle.** `guard` derefs (via `Deref`/`DerefMut`) to `&mut i32`, so you mutate through it. When `guard` goes out of scope its `Drop` impl releases the lock. You cannot forget to unlock for the same reason you cannot forget to free a `Box` — release is destruction, and destruction is automatic. This is the same scope-bound resource cleanup you get from Swift's `deinit` running at end of life, or a C++ destructor running at end of scope, applied to a lock instead of a heap allocation.

3. **`lock()` returns a `Result`.** If a thread panics while holding the guard, the mutex becomes *poisoned* — subsequent `lock()` calls return `Err(PoisonError)`. The rationale: a panic mid-critical-section may have left the protected invariant broken, so the lock refuses to silently hand out possibly-corrupt state. `.unwrap()` propagates that as a panic; in production you might inspect the error and call `.into_inner()` to recover the data deliberately.

> **🔧 In practice →** The textbook real-world use is a shared in-memory cache or counter behind a web server. Say every request handler thread needs to bump a hit counter and occasionally read a shared config map. You wrap the state once and every handler reaches it the same way:
> ```rust
> let hits = Arc::new(Mutex::new(0u64));
> // inside each request handler thread:
> {
>     let mut n = hits.lock().unwrap();
>     *n += 1;
> } // guard dropped here — lock held for two instructions, not the whole request
> ```
> The win over Java's `synchronized(counterLock) { ... }` is that there is no separate `counterLock` you might forget to take before touching `hits`: the *only* path to the `u64` is through `.lock()`. Note the tight `{ }` block — you deliberately drop the guard before doing slow per-request work so you do not serialise every handler behind one lock.

> **🦀 From your toolbox →** The closest things you have used are Java's `ReentrantLock` / `synchronized` and the lock you pair with a critical section by hand. The difference is what the lock *guards*. In Java the lock object and the fields it "protects" are related only by your intent — the compiler never checks that you held the lock before touching the fields. Rust's `Mutex<T>` makes the data physically reachable *only* through the lock, so unsynchronised access is not a discipline you maintain, it is a program you cannot write. One sharp edge from the Java world: `synchronized` and `ReentrantLock` are reentrant (one thread can re-lock what it already holds), but `std::sync::Mutex` is **not** — locking it twice on the same thread deadlocks.

> **⚠️ Pitfall →** A `MutexGuard` holds the lock for its *entire lifetime*, which is until end of scope, not until last use. So `let total = *m.lock().unwrap() + other.lock().unwrap()...` or holding a guard across a long computation keeps everyone else blocked. Worse, locking the *same* mutex twice in one expression deadlocks instantly: `*m.lock().unwrap() += *m.lock().unwrap();` acquires, then tries to re-acquire a non-reentrant lock the same thread already holds. Fix: bind the guard to a name, scope it tightly with an explicit `{ }` block, and copy the value out (`let n = *m.lock().unwrap();`) so the guard drops before you do further work.

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

> **⚙️ Under the hood →** `Arc<T>`'s heap allocation holds two counts (strong and weak) as `AtomicUsize` plus the `T`. `clone` is a `fetch_add(1, Relaxed)`; `drop` is a `fetch_sub(1, Release)` and, when it hits zero, an `Acquire` fence before running `T`'s destructor — the release/acquire pair establishes the ordering so the last dropper sees all prior writes. `Mutex<T>` on this platform wraps an OS futex (a `pthread_mutex_t`-equivalent) alongside the `T` and a poison flag. The lock and data sit in one contiguous allocation, which is *why* the lock can guard the data without indirection.

> **🦀 From your toolbox →** `Arc<T>` is exactly Swift's ARC or Python's reference counting, but explicit and thread-safe: every `Arc::clone` bumps a count, every drop decrements it, and the value is freed when it hits zero — deterministically, no GC pause, just like Swift's and Python's refcounts. The difference from both is that Swift and Python make *everything* refcounted automatically and quietly, whereas in Rust you opt into sharing by writing `Arc`, and you pay for atomics only there. The other half of the story is the one Swift and Python do not force on you: making the *data* safe to touch from two threads. Swift will happily let two threads mutate the same class instance and corrupt it; Python's GIL papers over it at a cost. Rust makes `Arc` share *ownership* but still requires you to wrap the data in `Mutex`/`RwLock`/an atomic to share *access* — and the type system refuses to let you skip that step.

### `RwLock<T>` and atomics, briefly

When reads vastly outnumber writes, `RwLock<T>` lets *many* readers (`.read()`) share access simultaneously while writers (`.write()`) get exclusivity — the classic readers-writer lock from Concurrent & Distributed Systems, with the same aliasing-XOR-mutability shape: many `&T` or one `&mut T`, now enforced at runtime by the lock. For a single integer or flag, skip the mutex entirely and use `std::sync::atomic` types like `AtomicUsize` or `AtomicBool`, which expose lock-free `fetch_add`, `compare_exchange`, etc., parameterised by the memory `Ordering` (`Relaxed`/`Acquire`/`Release`/`SeqCst`) you studied in the memory-consistency part of Concurrent & Distributed Systems.

## `Send` and `Sync`: the guarantee, encoded as types

Everything above is enforced by two marker traits in `std::marker`. They have **no methods** — they are pure type-level flags, and they are *auto-derived*: a type is `Send`/`Sync` iff all its fields are.

- **`Send`** — "ownership of a `T` may be transferred to another thread." `spawn` requires its closure (hence everything captured) to be `Send`.
- **`Sync`** — "a `&T` may be shared across threads," defined precisely as: `T: Sync` ⟺ `&T: Send`. If you can safely send a shared reference, the type is safe to share.

Almost everything is `Send + Sync` because almost everything is built from `Send + Sync` parts. The interesting cases are the *negative* ones, and each negative encodes exactly one would-be data race:

- `Rc<T>` is `!Send` and `!Sync` — its non-atomic refcount races, as we saw.
- `RefCell<T>` (and `Cell<T>`) is `!Sync` — its borrow flag is a non-atomic runtime check that two threads could corrupt. It *is* `Send` (you may move it to one other thread), just not `Sync` (you may not let two threads share `&RefCell`).
- Raw pointers `*const T`/`*mut T` are neither, because the compiler knows nothing about what they alias.
- `Arc<T>` is `Send + Sync` *provided* `T: Send + Sync`; `Mutex<T>` is `Sync` whenever `T: Send`, because the lock turns any `Send` payload into something safely shareable.

This is how the data-race-freedom guarantee is actually carried. `spawn`'s `F: Send` bound says "the captured state must be safe to move to another thread," and the compiler checks it the obvious way: `F` is `Send` because each of its fields is, all the way down to primitives which are `Send` by definition. When you wrote `Rc<Mutex<i32>>`, that check *failed* at the `Rc` field: `Rc` is not `Send`, so the closure capturing it is not `Send`, so the program does not compile. The error is concrete — the compiler points at the exact field that breaks the chain.

> **🎓 Tripos link →** The promise here is the same kind of "well-typed programs don't do the bad thing" property you meet informally in Semantics of Programming Languages: a program written entirely in safe Rust cannot have a data race, and `Send`/`Sync` are the type-level conditions that make that hold. They are auto-derived, so you almost never write them yourself — the compiler infers them from a type's fields, the same way type inference (Foundations of CS) spares you from spelling out most types. Implementing them by hand requires `unsafe` (see [chapter 16](16-unsafe-and-ffi.md)): that is you stepping outside the part of the language the compiler can check, and personally promising the safety the compiler can no longer verify for you.

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

- A data race needs aliasing + a writer + no synchronisation. Aliasing-XOR-mutability — the *same* borrow rule that gives single-threaded memory safety — kills the first two simultaneously, so data-race freedom is not a new feature, it is a consequence of ownership.
- `thread::spawn` demands `Send + 'static` on its closure: `'static` because the thread may outlive its spawner (so no borrows of locals), `Send` because the captured state must be safe to move across threads. `move` transfers ownership in; `JoinHandle::join` waits and retrieves the result. Forgetting to join detaches, and `main` exiting kills detached threads.
- Channels (`mpsc`) make CSP/actor message passing into ownership transfer: `send` moves the value, so "don't touch it after sending" is just move-after-use. `Mutex<T>` makes lock-before-access into a *type* — the lock owns the data, the `MutexGuard` is RAII, release is automatic, and the lock can be poisoned by a panicking holder.
- `Arc<Mutex<T>>` is the canonical shared-mutable-state idiom: `Arc` = thread-safe shared ownership (atomic refcount, vs `Rc`'s racy non-atomic one), `Mutex` = synchronised mutation. They compose because they solve orthogonal problems.
- `Send`/`Sync` are auto-derived, method-less marker traits that carry the data-race-freedom guarantee, checked field by field. `Rc`/`RefCell`/raw pointers are the instructive `!Send`/`!Sync` cases. Rust prevents races, **not** deadlocks or liveness bugs — your synchronisation logic is still your responsibility.

## Exercises

1. **Predict the bound that bites.** Here are three closures handed to `thread::spawn`: (a) one that captures and prints an `i32` by value, (b) one that captures `&local_vec` by reference, (c) one that captures an owned `Rc<String>` by `move`. For each, predict whether it compiles, and if not, predict whether the failing requirement is `'static` or `Send` — they fail for *different* reasons. Then explain in one sentence why (b) and (c) are genuinely different hazards even though both are rejected.

2. **Choose the tool.** You are designing two features. Feature A: a request handler must push completed jobs to a background writer that persists them to disk. Feature B: many handlers must increment a shared hit counter and read it back. For each, decide whether you would reach for a channel (`mpsc`) or `Arc<Mutex<T>>`, and justify the choice in terms of "ownership flowing through a pipe" versus "many owners touching one cell." When would Feature B instead be better served by a single `AtomicU64`?

3. **Adapt the worker-pool fix.** In the multi-producer example, the loop hangs unless you `drop(tx)`. Now suppose you refactor so the producer clones are created inside a helper function `fn spawn_workers(tx: Sender<String>)` that takes `tx` by value. Does the original `drop(tx)` in `main` still type-check, and does the hang go away or come back? Predict, then explain what owning-vs-borrowing the original sender has to do with when the channel closes.

4. **Which compiles, and why.** Of these, predict which build and which are rejected, naming the trait at fault: (a) sending an `Rc<i32>` down an `mpsc` channel into a spawned thread; (b) moving a `RefCell<i32>` into *one* spawned thread and mutating it there; (c) sharing `&RefCell<i32>` across two scoped threads that both read it. Case (b) and (c) hinge on the `Send`-but-not-`Sync` distinction — use them to articulate the difference between "send ownership to one thread" and "share a reference across threads."

5. (★) **Take on the proof obligation.** Define `struct Wrapper(*mut i32);` and try to move it into a spawned thread; observe that raw pointers are `!Send` so it fails. Then add `unsafe impl Send for Wrapper {}` to make it compile. Construct an input where two threads sharing one `Wrapper` produce a genuine data race on the `i32`, and state in one sentence what you promised the compiler when you wrote `unsafe impl` — i.e. which check you switched off and who is now responsible for it.
