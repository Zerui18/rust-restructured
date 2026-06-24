# 14. Asynchronous Rust

In [Fearless Concurrency](13-fearless-concurrency.md) you got *preemptive* concurrency: `std::thread::spawn` hands the OS a stack and a closure, and the kernel scheduler decides — at arbitrary instruction boundaries, driven by a timer interrupt — when to suspend one thread and resume another. Correct and, in Rust, statically race-free, but with a fixed cost: each thread carries its own kernel stack (megabytes of reserved virtual address space plus a kernel control block), and every switch crosses into the kernel and trashes the cache working set (CompArch/OS: TLB, cache). Spawn ten thousand of them to hold ten thousand mostly-idle sockets and you pay for ten thousand stacks to do almost nothing.

Async is the *cooperative* alternative. One OS thread, or a small pool, runs an in-process scheduler multiplexing thousands of logical tasks. A task yields only at points *you* marked, and the whole machine is built so the compiler still proves memory- and data-race-safety statically. This is the tool for **I/O-bound, high-connection-count** work — the multithreaded web server in the [capstones](21-capstones.md) chapter, or any network service holding thousands of slow, mostly-waiting TCP connections (Computer Networking). It is *not* a speedup for CPU-bound work; for that, use threads, or both together.

> **🎓 Tripos link →** This is the cooperative-vs-preemptive scheduling distinction from Concurrent and Distributed Systems, lifted into a language feature. OS threads are preemptively scheduled by the kernel; async tasks are cooperatively scheduled by a user-space runtime. Cooperative scheduling has lower switching cost (no syscall, no kernel stack) but demands that tasks *voluntarily* yield — a task that never yields starves all others sharing its thread, exactly the failure mode of cooperative kernels of the 1990s.

## The core mechanism: `async`/`.await` compiles to a state machine

The single most important fact in this chapter, and what makes Rust's async model different from almost every other language's: an `async fn` or `async` block does not *run* anything. It compiles to an anonymous struct — a **state machine** — implementing the `Future` trait. Calling the function just *constructs* that struct. Nothing executes until something polls it.

```rust
use std::future::Future;

async fn fetch_len(url: &str) -> usize {
    let body = download(url).await;   // suspension point 1
    let processed = transform(body).await; // suspension point 2
    processed.len()
}
```

`fetch_len` desugars to roughly a function returning `impl Future<Output = usize>`:

```rust
fn fetch_len(url: &str) -> impl Future<Output = usize> {
    async move {
        let body = download(url).await;
        let processed = transform(body).await;
        processed.len()
    }
}
```

The `async move { ... }` block is itself compiled into a hidden enum-like state machine. Conceptually:

```rust
// Sketch of what rustc generates — you never write this.
enum FetchLen<'a> {
    Start { url: &'a str },
    AwaitingDownload { fut: DownloadFut },          // paused at .await #1
    AwaitingTransform { fut: TransformFut },         // paused at .await #2
    Done,
}
```

Each `.await` is a possible suspension point. By liveness analysis across those points, the compiler computes exactly which locals are still live at each one and stores precisely those in the corresponding variant. This is a **continuation-passing-style transform**: it carves the linear body into segments delimited by `.await`, and the variant you are "in" *is* the saved continuation.

> **🎓 Tripos link →** This is the CPS transformation from Compiler Construction, materialised as a data structure instead of a closure. Where a CPS compiler heap-allocates a closure capturing the live environment at each yield, rustc lays out a *flat enum* whose largest variant bounds the whole future's size — no per-suspension allocation. The liveness analysis deciding what each variant stores is the same dataflow analysis you saw for register allocation, run over `.await` boundaries instead of basic blocks.

### The `Future` trait

A future is anything implementing this:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

`poll` is the heart of it. The runtime calls `poll`; the state machine runs forward from its current state until it hits an `.await` whose inner future returns `Pending`, whereupon this future *also* returns `Pending`, recording which state it stopped in. The next `poll` resumes from that state. When it runs off the end, it returns `Ready(value)`.

`.await` itself desugars, in spirit, to a poll-or-yield loop:

```text
loop {
    match inner_future.poll(cx) {
        Ready(v)  => break v,          // got the value, continue past the .await
        Pending   => return Pending,   // yield: bubble Pending up to OUR caller
    }
}
```

The crucial line is `Pending => return Pending`. When an inner future isn't ready, control doesn't block — it returns all the way up through every enclosing future to the runtime, which is then free to poll a *different* task. That bubbling-up is the cooperative yield. Everything *between* two `.await` points runs straight through, synchronously.

> **🦀 From your toolbox →** A `Future` is morally a reified continuation — like CPS-converting an OCaml function by hand, except the compiler does it and packs the environment into a struct. The `poll` signature `Poll<T>` from `&mut self` should also remind you of `Iterator::next(&mut self) -> Option<T>`: both pull-based, both driven by repeated calls, both threading state through `&mut self`. `for` desugars to `Iterator`; `.await`/`async` desugars to `Future`. The analogy breaks at *who drives*: you drive an iterator with a loop you wrote; a runtime drives a future, and only re-polls when told it *might* now progress (the waker, below) — it does not spin.

### Futures are inert until polled — unlike JS Promises

If you come from TypeScript, internalise this now. A JavaScript `Promise` is **eager**: calling an `async function` starts its body executing up to the first `await`, scheduling work on the event loop *whether or not anyone ever `.then`s it*. A Rust future is **lazy**: calling an `async fn` runs *zero* of its body, handing you an inert state machine in its `Start` state. Nothing happens until a runtime polls it — because you `.await`ed it, or handed it to the runtime to drive.

```rust
let fut = fetch_len("https://example.com"); // NOTHING has happened. No request sent.
// ... fut is just a value. You could drop it here and no I/O ever occurs.
let n = fut.await;                            // NOW it runs (when the surrounding task is polled).
```

The compiler enforces awareness of this: an unused future raises a `#[must_use]` warning, because constructing a future and never awaiting it is almost always a bug — the dead giveaway of a TS habit applied to Rust.

> **⚠️ Pitfall →** `warning: unused implementer of `Future` that must be used` — `note: futures do nothing unless you `.await` or poll them`. You wrote `something_async();` expecting a fire-and-forget side effect, as you would in JS. In Rust that line constructs a future and immediately drops it, doing nothing. The fix: `.await` it, or hand it to the runtime with `spawn` (below) if you genuinely want it to run independently.

> **⚙️ Under the hood →** Laziness is what makes Rust async zero-cost and allocation-free in the common case. A future is a value of statically known size (its largest state-machine variant), so an `async fn` awaiting other `async fn`s composes their state machines *by inlining the inner enum as a field of an outer variant*. A whole leaf-to-root call tree can collapse into one flat struct on the caller's stack — no heap, no per-`await` allocation. Eager models like JS *must* heap-allocate and schedule a promise object per call; Rust pays nothing until you opt into spawning.

## Runtimes: Rust ships none

The standard library defines `Future`, `Poll`, `Context`, `Waker`, and `Pin`, but *no executor*: no built-in event loop, no `tokio`-equivalent in `std`, and `fn main` cannot be `async`:

```rust
async fn main() { /* ... */ }
// error[E0752]: `main` function is not allowed to be `async`
```

This is deliberate. A runtime makes hard, workload-specific tradeoffs: number of OS threads, work-stealing vs single-threaded, timer granularity, I/O backend (epoll/kqueue/io_uring), whether it allocates at all. A 64-core web server and a heap-less microcontroller want opposite runtimes, so Rust makes the executor *pluggable*: you pick a crate. [`tokio`](https://tokio.rs) dominates server-side; `smol` is minimal; `embassy` targets embedded. You bridge sync `main` to async with `block_on`:

```rust
fn main() {
    // The #[tokio::main] macro expands to roughly this:
    let runtime = tokio::runtime::Runtime::new().unwrap();
    runtime.block_on(async {
        let n = fetch_len("https://example.com").await;
        println!("{n}");
    });
}
```

`block_on` is the boundary between the synchronous and asynchronous worlds: it parks the current OS thread and drives the given future (plus any tasks spawned onto the runtime) to completion. Everything *inside* avoids blocking; the thread calling `block_on` *does* block, by design — the one place you choose to bridge worlds.

> **⚙️ Under the hood →** When `poll` returns `Pending`, how does the runtime know when to retry? It does *not* busy-poll. The `Context` passed to `poll` carries a `Waker`. A leaf future (say, a socket read) stashes a clone of the waker with the OS event source: "wake me when this fd is readable" registered with epoll/kqueue. On kernel readiness the reactor calls `waker.wake()`, pushing the owning task back onto the run queue. The loop: poll → `Pending` + waker registered → thread sleeps in `epoll_wait` → readiness → wake → re-poll. The waker is what makes laziness efficient rather than a spin-loop.

## Composing futures: `join`, `select`, and spawning tasks

Several ways to make multiple futures progress, differing in *who owns the concurrency* and *what "done" means*.

**`join` / `join!` — wait for all.** Produces a single future that polls each child and completes when *all* complete, yielding a tuple of outputs.

```rust
use tokio::join;
async fn run() {
    let (a, b) = join!(fetch_len("https://a.com"), fetch_len("https://b.com"));
    println!("{a} {b}");
}
```

Both children run *concurrently within the same task* — when one returns `Pending`, the join polls the other. No threads, no spawning. Use `join_all` for a runtime-sized `Vec` of futures (with a Pin caveat, below).

**`select!` — race; take the first.** Polls multiple futures and completes as soon as *one* is ready, returning its result and (typically) dropping the others.

```rust
use tokio::select;
async fn race() {
    select! {
        r = fetch_len("https://fast.com")  => println!("fast won: {r}"),
        r = fetch_len("https://slow.com")  => println!("slow won: {r}"),
    }
}
```

`select` builds **timeouts** with no special machinery — race the real future against a sleep:

```rust
async fn timeout<F: Future>(work: F, limit: Duration) -> Result<F::Output, Duration> {
    select! {
        out = work               => Ok(out),
        _   = tokio::time::sleep(limit) => Err(limit),
    }
}
```

A timeout is *just composition*: two futures, first to finish wins. That is the payoff of futures being first-class values — `timeout`, `retry`, `throttle` are ordinary functions over futures, not built-in keywords.

**`spawn` — hand a task to the runtime to drive independently.** `join!` and `select!` keep everything in *one* task; the futures share that task's poll budget and cannot outlive the `join!`. `tokio::spawn` is different: it submits a future as a *new top-level task* the executor schedules independently, possibly on another OS thread (tokio's default runtime is multithreaded and **work-stealing**). It returns a `JoinHandle`, itself a future you `.await` for the result.

```rust
async fn run() {
    let handle = tokio::spawn(async {
        do_background_work().await
    });
    // ... do other things concurrently ...
    let result = handle.await.unwrap(); // JoinHandle::Output is Result<T, JoinError>
}
```

> **🦀 From your toolbox →** `join!` is like interleaving two coroutines on the *same* thread yourself — structured, scoped, no shared-thread escape. `tokio::spawn` is like `std::thread::spawn` from [chapter 13](13-fearless-concurrency.md): fire-it-and-get-a-handle, may land on a different OS thread, hence requires `Send` (next section). The analogy to threads breaks on *cost and number*: spawning a task is a small heap box plus a queue push (microseconds, kilobytes), so 100,000 tasks is routine where 100,000 threads is not.

### Tasks vs threads vs futures: the M:N picture

Three nested units of concurrency:

- A **future** is the finest grain — one state machine, one unit of *cooperative* progress, possibly containing a tree of child futures. Concurrency *within* a task lives here, via `join!`/`select!`.
- A **task** is a top-level future the runtime *owns and schedules* — the unit on the executor's run queue. Concurrency *between* tasks lives here.
- A **thread** is an OS-scheduled execution resource; the runtime runs a pool and multiplexes many tasks onto few threads.

This is an **M:N model**: M tasks scheduled onto N OS threads (N ≈ core count). Work-stealing lets an idle worker steal tasks from a busy worker's queue, balancing load transparently. A task can migrate between threads at `.await` points — precisely why spawned futures must be `Send`.

> **🎓 Tripos link →** M:N user-level threading is the green-threads / user-level-scheduling design from Concurrent and Distributed Systems. The classic green-thread problem — one blocking syscall stalls the whole scheduler thread — is dodged because async futures *never* make a blocking syscall at an `.await`; they register a waker and yield. For genuinely blocking work, `spawn_blocking` moves it to a dedicated thread pool so it can't stall the cooperative scheduler.

## The starvation trap: only `.await` yields

Cooperative scheduling has a sharp edge. **A task yields control *only* at an `.await` point.** Everything between `.await`s runs to completion with no preemption. So a future doing a long CPU-bound stretch — or worse, calling a *blocking* synchronous API like `std::thread::sleep` or a synchronous file read — without awaiting will **starve** every other task sharing its thread. The runtime cannot preempt it; there is no timer interrupt in user space.

```rust
async fn greedy() {
    for chunk in huge_dataset() {
        crunch(chunk);            // CPU-bound, no .await — NOTHING ELSE RUNS on this thread
    }
}
```

To stay cooperative inside a long loop, hand control back explicitly with `tokio::task::yield_now().await` after each chunk — a future that just yields once:

```rust
async fn polite() {
    for chunk in huge_dataset() {
        crunch(chunk);
        tokio::task::yield_now().await;  // "nothing to wait for, but let others run"
    }
}
```

`yield_now` returns `Pending` exactly once (re-queuing its waker immediately) then `Ready` — a pure yield with no waiting, unlike `sleep`. But yielding isn't free, and on a multithreaded work-stealing runtime a CPU loop on one worker doesn't block *other* workers. The honest rule: don't do unbounded blocking or CPU work inside an `async fn`. Move CPU-bound work to threads (`spawn_blocking` or a `rayon` pool) and reserve async for I/O-bound waiting.

> **⚠️ Pitfall →** The most common real bug: calling `std::thread::sleep(d)` instead of `tokio::time::sleep(d).await` inside async code. `std::thread::sleep` blocks the *OS thread*, freezing every task multiplexed onto it — and there's no compiler error, just mysterious latency. Worse on a single-threaded runtime, where it deadlocks anything the frozen tasks were supposed to feed. Fix: use the runtime's async timer (`tokio::time::sleep`) which yields; for unavoidable blocking calls (legacy sync libraries, synchronous file I/O), wrap them in `tokio::task::spawn_blocking`.

## `Send` across `.await` points

A spawned task may move between worker threads, parking its live state — the state-machine struct — across `.await`. So everything held *across* an `.await` must be `Send`, since it may be resumed on a different thread. This is the same `Send`/`Sync` story from [Fearless Concurrency](13-fearless-concurrency.md), now applied to whatever the compiler stored in the future's variants.

```rust
use std::rc::Rc;
async fn bad() {
    let local = Rc::new(5);          // Rc is !Send
    do_io().await;                   // <-- `local` is live across this .await
    println!("{}", local);
}
// tokio::spawn(bad()) =>
// error: future cannot be sent between threads safely
//   the trait `Send` is not implemented for `Rc<i32>`
//   note: future is not `Send` as this value is used across an await
```

The compiler tracks, per `.await`, exactly which locals are live (the liveness analysis from the desugaring). If a `!Send` value is live across an `.await`, the whole future is `!Send` and can't be spawned on a multithreaded runtime. The fix is usually to make the value `Send` (`Arc` not `Rc`) — or, tellingly, to *not hold it across the await*: drop it in an inner scope first, and the future stays `Send`.

```rust
async fn good() {
    {
        let local = Rc::new(5);
        println!("{}", local);       // used and dropped BEFORE the await
    }                                // `local` no longer live here
    do_io().await;                   // nothing !Send is live across this point
}
```

> **⚙️ Under the hood →** "Held across an `.await`" is a dataflow notion, not a lexical one. The compiler asks: at each suspension point, is this variable in the live set? A value constructed, used, and dropped *between* two awaits never appears in any variant, so it never affects `Send`-ness. That is why the inner-scope trick works: ending the scope kills the liveness, so the variable isn't stored in the variant straddling the next `.await`.

## Streams: asynchronous iterators

A `Future` resolves to *one* value. Many async sources produce a *sequence* over time — incoming packets, lines from an async read, channel messages. That is a **`Stream`**: the async analogue of `Iterator`, and exactly the merge of the two traits you know:

```rust
// From Iterator: a sequence, next() -> Option<Item>.
// From Future:   readiness over time, poll() -> Poll<Output>.
// Stream merges them:
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

`poll_next` returns `Poll<Option<Item>>`: the outer `Poll` says "ready or come back later" (the `Future` part); the inner `Option` says "another item, or end-of-stream `None`" (the `Iterator` part). As of this writing `Stream` lives in the `futures` crate, not `std` (stabilisation is in progress), but the ecosystem agrees on this shape.

You rarely call `poll_next` directly. Just as the `Iterator` combinators live on `Iterator`, the async combinators (`.next()`, `.map`, `.filter`, `.take`) live on a separate **`StreamExt`** extension trait — `Ext` is the community idiom for "convenience layer over a minimal core trait." You must bring it into scope:

```rust
use tokio_stream::StreamExt;   // <-- forgetting this is the usual first error

async fn consume(mut stream: impl Stream<Item = i32> + Unpin) {
    while let Some(value) = stream.next().await {  // async pull
        println!("{value}");
    }
}
```

Note the loop shape: `while let Some(v) = stream.next().await`, not `for v in stream`. Rust has no `for await` syntax (yet); `while let` over `.next().await` is the idiom. Each iteration hits an `.await`, so each is a cooperative yield point — a stream consumer interleaves naturally with other tasks.

> **⚠️ Pitfall →** `error[E0599]: no method named `next` found for ...`, `StreamExt is not in scope`. You have a `Stream` but called `.next()` without `use futures::StreamExt;` (or the runtime's re-export, `tokio_stream::StreamExt`). The method-providing trait must be imported — the same in-scope rule as any extension trait, in spirit identical to needing `Iterator` in scope.

## Pin and Unpin: the self-referential-future problem

This is the part everyone finds strange, so let's be precise about *why* it has to exist. Recall the state machine. Between two `.await` points your code can take a reference into a local that is *also* stored in the future:

```rust
async fn selfref() {
    let buf = [0u8; 1024];
    let slice = &buf[..];      // `slice` borrows `buf`
    do_io(slice).await;        // both `buf` AND `slice` are live across this .await
}
```

Both `buf` and `slice` must be saved in the state-machine variant straddling that `.await`, but `slice` points *into* `buf`, and both now live in the *same* struct. The future is **self-referential**: a field holds a pointer to another field of the same struct.

Self-referential structs are catastrophic to *move*. A move in Rust is a bitwise copy of bytes to a new address (recall [moves](02-ownership-and-moves.md)), and Rust does **not** fix up internal pointers. After the move, `slice` still holds the *old* address of `buf`, now garbage — dereferencing it is a use-after-free. This is exactly why the borrow checker normally forbids moving a value while a reference into it is live; but here the reference is *inside* the value being moved, so the usual rule can't apply.

The fix is **`Pin`**. `Pin<P>` wraps a pointer `P` (`&mut T`, `Box<T>`, …) and statically guarantees the *pointee* never moves again until dropped. Look back at the trait: `poll` takes `self: Pin<&mut Self>`, not `&mut self`. That is the whole point — to poll a future (and drive its self-references forward) you must first pin it, promising it won't move out from under its own pointers. A pinned future has a stable address, so its self-references stay valid. Pinning the *pointer* is fine: you can move a `Pin<Box<Fut>>` freely, because moving the `Box` doesn't move the heap allocation the `Fut` lives in. What's pinned is the pointee at its address.

**`Unpin`** is the escape hatch: an auto-trait (like `Send`) the compiler implements for every type *safe to move even when pinned* — i.e. without internal self-references. `i32`, `String`, `Vec<T>`, ordinary structs: all `Unpin`. For an `Unpin` type, `Pin` is a no-op wrapper — you can freely get `&mut T` back out. The only `!Unpin` types are ones that genuinely can't move after pinning: compiler-generated self-referential futures, and a few hand-built types opting out with `PhantomPinned`. Mental rule: **`Unpin` is the normal case; `!Unpin` is the rare self-referential case; pinning only constrains you when the type is `!Unpin`.**

Why doesn't `.await` force `Pin` everywhere? Because awaiting a future *pins it implicitly* — the `.await` machinery pins it in place (on the current stack frame) before polling. You only meet `Pin`/`Unpin` in errors when you handle a future *as a value* without awaiting it directly:

```rust
let futures: Vec<Box<dyn Future<Output = ()>>> = vec![Box::new(a), Box::new(b)];
trpl::join_all(futures).await;
// error[E0277]: `dyn Future<Output = ()>` cannot be unpinned
//   the trait `Unpin` is not implemented for `dyn Future<Output = ()>`
//   = note: consider using the `pin!` macro / `Box::pin`
```

`join_all` needs to poll each future, so it needs each pinned. A bare `dyn Future` isn't `Unpin`, so you must pin them first — `Box::pin(a)` gives `Pin<Box<dyn Future>>`, or the `std::pin::pin!` macro pins on the stack:

```rust
use std::pin::pin;
let a = pin!(async { /* ... */ });
let b = pin!(async { /* ... */ });
let futures: Vec<Pin<&mut dyn Future<Output = ()>>> = vec![a, b];
trpl::join_all(futures).await;
```

> **🎓 Tripos link →** `Pin` is a *type-system encoding of an address-stability invariant* — Curry-Howard flavour from Logic and Proof. The invariant "this value's address never changes after pinning" can't be checked at runtime, so it's lifted into a type whose API makes violating it impossible in safe code (you can't get `&mut T` out of `Pin<&mut T>` unless `T: Unpin`). The proof obligation discharged when you *do* go through `unsafe` (a manual `Future` impl) is precisely "I uphold the pin contract."

> **⚙️ Under the hood →** Why not have the compiler patch internal pointers on every move, like a moving GC? Because that taxes *every* move of *every* type to fix a problem only self-referential types have, and Rust's moves are meant to be trivial `memcpy`s with no hidden code. `Pin` instead makes the *minority* (self-referential futures) bear the cost via a restrictive API, while `Unpin` exempts the overwhelming majority at zero cost.

## `async` in traits: the current status

Until recently you could not write `async fn` in a trait at all: the desugared return type is an *anonymous, per-implementation* future, with no way to name it in the trait. As of Rust 1.75, **`async fn` in traits ("AFIT") is stable** for the common case:

```rust
trait Fetcher {
    async fn fetch(&self, url: &str) -> String;   // stable since 1.75
}
```

This desugars to `fn fetch(&self, url: &str) -> impl Future<Output = String>` — return-position `impl Trait` in a trait. The remaining sharp edge: there is no clean built-in way to require, *from the trait definition*, that the returned future be `Send`, which matters when implementors must be usable with `tokio::spawn`. The community workaround for public, dyn-compatible, `Send`-requiring async traits is the [`async-trait`](https://docs.rs/async-trait) macro, which rewrites each method to return a boxed `Pin<Box<dyn Future + Send>>` (one heap allocation per call), sidestepping both naming and dyn-compatibility — at that allocation cost. For internal traits where you control the types, native AFIT is preferred. This area is still in motion; check current docs.

> **⚠️ Pitfall →** Native `async fn` in traits is *not yet `dyn`-compatible* in general — you cannot always make `dyn Fetcher` from a trait with `async fn` methods, because the return type is an opaque `impl Future` whose size isn't known for the vtable (recall the object-safety constraints in [trait objects](09-trait-objects-and-oop.md)). If you need dynamic dispatch over an async trait, reach for `#[async_trait]`, which boxes the future to give the vtable a uniform pointer-sized return type.

## When to use async — an honest map

Async is not free and not always the answer:

- **I/O-bound, many concurrent waits** (servers, proxies, scrapers, anything holding thousands of mostly-idle connections): async wins decisively — one small state machine per connection instead of one OS stack.
- **CPU-bound, parallelisable** (image processing, numerical kernels): use threads (`rayon`, a thread pool). Async gives you *nothing* here — it structures concurrency, it is not a parallelism primitive, and CPU work in an `async fn` just starves the scheduler.
- **Both**: combine them. Encode video on a dedicated thread pool, notify the async network layer via a channel. `tokio::spawn_blocking` exists precisely to bridge blocking/CPU work into an async program without stalling the executor.

The costs of going async: a steeper type-level curve (`Pin`, `Send`-across-await, lifetimes in futures), longer compiles and larger binaries (every distinct future is a monomorphised state machine), worse backtraces (the stack is a state machine, not a real stack), and the ever-present starvation footgun. The reward, when the workload fits, is C10k-and-beyond on a handful of threads with statically-guaranteed data-race freedom — something thread-per-connection cannot afford.

## Mental-model recap

- `async fn`/`async {}` compile to an anonymous **state machine** implementing `Future`; `.await` carves the body at suspension points (a compiler CPS transform). `.await` = "poll the inner future; if `Pending`, register the waker and yield `Pending` up to the runtime."
- Futures are **lazy/inert**: calling an `async fn` runs none of its body and allocates nothing; only polling runs it. The opposite of eager JS Promises, and the source of zero-cost composition.
- Rust ships **no runtime** — the executor is a pluggable crate (`tokio` etc.) matched to the workload, from 64-core servers to heapless microcontrollers. `block_on` is where you cross from sync to async.
- Three nested units in an **M:N** model: futures (concurrency *within* a task via `join!`/`select!`), tasks (scheduled by the runtime, `spawn`), threads (the few OS threads tasks multiplex onto, work-stealing). Async yields **only** at `.await`, so blocking/CPU code between awaits *starves* the scheduler.
- `Pin`/`Unpin` exist because state machines can become **self-referential** and Rust moves don't fix internal pointers; `Pin` guarantees address stability so `poll` is sound, `Unpin` (the normal case) exempts non-self-referential types. Spawning needs `Send`, decided by what's live *across* an `.await`.

## Exercises

1. Write an `async fn` that constructs a future with `let f = some_async_call();`, never `.await`s it, and returns. Predict the compiler diagnostic, then confirm it. Explain in one sentence why this is a warning and not an error, and why the equivalent JS code *would* perform the side effect.

2. Build a `retry` combinator: `async fn retry<F, Fut, T, E>(mut make: F, attempts: u32) -> Result<T, E>` where `F: FnMut() -> Fut` and `Fut: Future<Output = Result<T, E>>`. It calls `make()` to get a fresh future each attempt (why must you re-call `make` rather than `.await` the same future twice — what does the note about polling-after-`Ready` say?), awaits it, and retries on `Err` up to `attempts` times.

3. Take the `bad()`/`good()` `Rc`-across-await example. Without changing `Rc` to `Arc`, make the future `Send`-spawnable purely by restructuring scopes. Then explain, in terms of the state-machine variant straddling the `.await`, exactly *why* your version is `Send` and the original is not.

4. You have a `tokio` runtime and inside an `async fn` you write `std::thread::sleep(Duration::from_secs(1));` in a loop that's supposed to run concurrently with another task via `join!`. Describe the observed behaviour, why there is *no* compiler error, and give two distinct correct fixes (one if the sleep is a real wait, one general one for an unavoidable blocking call).

5. (★) Hand-write a type `Countdown { remaining: u32 }` that implements `Future<Output = ()>` directly (no `async`): each `poll` decrements `remaining`, and returns `Pending` (after re-waking via `cx.waker().wake_by_ref()`) until it hits zero, then `Ready(())`. Then explain why `Countdown` is `Unpin` even though it implements `Future` by hand, whereas a compiler-generated `async {}` future generally is not.

6. (★) `trpl::join_all(vec![Box::new(fut_a), Box::new(fut_b)])` fails with `the trait Unpin is not implemented for dyn Future`. Explain the error in terms of what `join_all` must do to each future, why `await`ing a single future directly does *not* trigger this, and fix it two ways: with `Box::pin` and with the `std::pin::pin!` macro. State the difference in *where* each pins the future (heap vs stack) and when you'd prefer each.
