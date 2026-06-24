# 0. Orientation: the thesis of Rust

You already know how to write safe code in a managed runtime: Java with its garbage collector, Swift with ARC, Python with reference counting. You have probably also brushed against the other world — a raw pointer in C or C++ that you have to free by hand, with the constant low hum of "did I free this twice? is this pointer still valid?" The industry consensus you grew up with is that these are a trade-off: you pick manual memory management and accept use-after-free, double-free, and data races as the price of control; or you accept a managed runtime — a garbage collector that periodically traces the heap, or reference counts maintained behind your back — and pay with latency, throughput, footprint, and the loss of *deterministic* cleanup (you don't know exactly *when* an object dies). Rust's entire reason to exist is the claim that this trade-off is not fundamental. It is an artifact of weak type systems.

## The one-sentence thesis

**Rust achieves memory safety and data-race freedom at compile time, with no garbage collector and no runtime overhead, by enforcing single ownership of every value, compiler-checked references ("lifetimes" / "borrowing"), and abstractions that compile away to zero cost.**

Unpack each clause; the rest of this book is just these three ideas pursued relentlessly.

**Single ownership.** Every value has exactly one *owner*: the binding responsible for it. A value can be *used up* — handed off to someone else (a "move") — and once that happens, the original binding is dead and the compiler refuses to let you touch it. Think of Swift's value types or a Java reference where, after you pass it along, the language *forbids* you from reading the old variable. When the owner's binding leaves scope, the value is deterministically destroyed right then — not "eventually, when the GC gets around to it." This is the same idea as a C++ destructor running at the end of a scope, but promoted from a convention you opt into to a property the type system enforces on *every* value. And unlike C++'s move (which leaves behind a valid-but-empty husk you can still accidentally read), a moved-from variable in Rust is simply not usable — reading it is a compile error, not a subtle bug.

**Compiler-checked references.** You usually don't want to *use up* a value just to read it. So Rust lets you *borrow* it, producing a reference whose validity the compiler tracks across the program. The span over which a reference must stay valid is called its *lifetime*, and the compiler figures these out for you, using them purely for checking — never for allocation, never at runtime. The borrow checker enforces one invariant: at any point in the program, you may have either *one* mutable reference *or* any number of shared (read-only) references to a given value, never both at once. If that rule reminds you of a readers-writer lock — many readers OR one writer — that is exactly the right intuition, except Rust resolves it entirely at compile time, with zero runtime cost, and it is precisely what makes data races impossible. We develop this in [references](03-references-and-borrowing.md) and [lifetimes](04-lifetimes.md).

**Zero-cost abstractions.** Generics in Rust are *monomorphised*: when you write `Vec<T>` and use it with `i32` and with `String`, the compiler stamps out a separate specialised copy of the code for each concrete type — no boxing, no runtime type tags. This is the opposite of Java generics, which erase to `Object` and box; it is closer to what a C++ template does. Traits (Rust's interfaces) dispatch statically by default; you opt into dynamic dispatch — a vtable, like a Java interface call or a C++ virtual method — explicitly with `dyn`. Iterators, closures, `Option`, and `Result` compile down to the same loops and branches you'd write by hand. The slogan: *what you don't use, you don't pay for; what you do use, you couldn't hand-code better.*

> **🦀 From your toolbox →** The cleanest mental anchor is **value semantics plus a compiler that enforces who is responsible for what**. From **Swift**: structs are value types, copied or moved rather than aliased; Rust generalises that to almost everything. From **Java**: `interface` is `trait`, and dynamic dispatch through an interface reference is `dyn Trait`. From **Python**: a `list` is `Vec`, a `dict` is `HashMap`, and the duck-typing instinct ("if it has the method, it works") becomes a trait bound checked at compile time. The genuinely new thing is that the compiler *statically proves* the cleanup and aliasing rules that Java/Swift/Python enforce with a runtime (GC, ARC, refcounting) — Rust enforces them with no runtime at all. **Where the analogy breaks:** in Java/Swift/Python an object lives as long as something points at it, decided at run time; in Rust the *moment* of destruction is fixed at compile time, and using a value after it has been moved away is an error the compiler catches before the program ever runs.

## The central mental model: ownership is a tree, borrows point into it

Commit to this picture now; everything hangs off it.

Every value is owned by exactly one binding at a time. Composite values own their parts: a `Vec<String>` owns its heap buffer, which owns each `String`, which owns *its* heap buffer. Ownership therefore forms a **tree** (more precisely a forest rooted at stack frames). When a root goes out of scope, destruction recurses down the tree — deterministically, at a point fixed when you compiled, with no tracing GC pass and no reference-count bookkeeping.

References (`&T`, `&mut T`) are **non-owning** edges that point *into* this tree. The borrow checker's job is to prove that no reference outlives the part of the tree it points at, and that mutable access is exclusive. A program that type-checks comes with a guarantee from the compiler that it has no dangling pointers and no data races. That is the deal Rust offers: accept the discipline, receive the guarantee.

When the tree shape is genuinely insufficient — you need shared ownership, cycles, or a graph — you reach for [smart pointers](15-smart-pointers.md) (`Rc`, `RefCell`, `Arc`, `Mutex`) that move some checks to runtime, or, at the very edge, [`unsafe`](16-unsafe-and-ffi.md) where you take responsibility for the guarantee yourself. `Rc` and `Arc` are reference counting made explicit — the exact mechanism Swift's ARC and Python's runtime apply automatically, except here *you* choose where it happens, and only where the tree genuinely isn't enough. The tree is the default, and idiomatic Rust keeps it that way far longer than your instincts will predict.

## A glance at Hello, World

You will spend almost no time here; you've written `main` thousands of times.

```rust
fn main() {
    println!("Hello, world!");
}
```

Three things to register and move on:

- `fn main()` is the entry point, as in C or Java. No arguments needed; the OS-level `argc/argv` are reached through `std::env`.
- `println!` is a **macro**, not a function — the `!` is the tell. It is not C's `printf`: the format string is parsed and type-checked at compile time, so a mismatch between placeholders and arguments is a compile error, not a runtime format-string vulnerability. Macros are [their own chapter](18-macros.md); for now, `!` just means "expands to code."
- Statements end in `;`. Rust is expression-oriented (a block evaluates to its last expression), so the `;` is semantically meaningful — it discards a value, turning an expression into a statement. We make this precise in [the language delta](01-language-delta.md).

Compile directly with `rustc main.rs` (like invoking a C compiler by hand) and you get a standalone native binary — no interpreter, no VM, no `.jar`, statically linked against the Rust standard library by default. But nobody invokes `rustc` directly past this paragraph.

## The toolchain at expert speed

**`rustup`** is the toolchain multiplexer — conceptually `pyenv`/`nvm` for Rust, or `sdkman` if you've juggled JDK versions. It installs and switches between *toolchains* (a toolchain = a `rustc` + `cargo` + `std` of a given channel: `stable`, `beta`, `nightly`, or a pinned version like `1.96.0`). A directory can pin its toolchain via `rust-toolchain.toml`. `rustup update` pulls new releases (stable ships every six weeks); `rustup component add` fetches `clippy`, `rustfmt`, `rust-src`, etc.; `rustup doc` opens the *offline* copy of std and the book in your browser. Target cross-compilation (`rustup target add aarch64-unknown-linux-gnu`) lives here too.

**`cargo`** is build system *and* package manager in one — think Maven/Gradle for Java or pip+setuptools for Python, but unified, with conventions instead of configuration. A project (a *package*) has a `Cargo.toml` manifest (TOML: name, version, `edition`, `[dependencies]`) and a `Cargo.lock` (the resolved, exact-version dependency graph — commit it for binaries, conventionally not for libraries). Source lives in `src/`; build artifacts in `target/`. The commands you will actually live in:

| Command | What it does | Analogy |
|---|---|---|
| `cargo new <name>` | Scaffold package (`Cargo.toml`, `src/main.rs`, git init) | — |
| `cargo check` | Type-check + borrow-check, **emit no binary** | the fast inner loop |
| `cargo build` | Compile to `target/debug/` (unoptimised + debuginfo) | a plain compile |
| `cargo build --release` | Optimised build to `target/release/` | compile with `-O` |
| `cargo run` | `build` then execute (rebuilds only if stale) | — |
| `cargo test` | Build and run all tests (see [testing](20-testing-and-tooling.md)) | `mvn test` / `pytest` |
| `cargo clippy` | Lint: hundreds of idiom/correctness/perf lints | a Rust-aware linter |
| `cargo fmt` | Canonical formatting (rustfmt) | `gofmt`/`black`, but one true style |

`cargo check` is the command you run on every save: it does all the analysis that catches your bugs (type, borrow, lifetime) and *skips* codegen and linking, so it returns in a fraction of `build`'s time. Reserve `build`/`run` for when you actually want to execute.

> **🔧 In practice →** When you're working through a chapter of this book, you almost never type `cargo build`. You keep an editor open with `rust-analyzer` showing errors inline, and in a terminal you run `cargo check` (or just save and watch the editor). You're iterating on a borrow-checker error — say the compiler is rejecting a reference you're trying to return — and you want the *next* error message in under a second, not after a full optimising compile and link. Only once it type-checks cleanly do you reach for `cargo run` to actually watch it execute. That tight `edit → check → fix the error message → check` loop is how Rust development actually feels day to day.

> **⚙️ Under the hood →** The debug/release split is two *profiles*. Debug is `opt-level = 0` with debuginfo and overflow checks on, so it *panics on integer overflow* rather than wrapping. Release is `opt-level = 3` with overflow checks off (wrapping in release is defined two's-complement, not undefined behaviour). The semantics difference is deliberate: it surfaces bugs in development without costing a branch in production. Benchmark only release builds — a debug binary can be 10–50× slower and will mislead you completely.

### Editions: what they are and are not

`Cargo.toml` carries an `edition` key — `2015`, `2018`, `2021`, or `2024`. An **edition is an opt-in set of source-language changes** (new keywords, changed defaults, syntax) scoped per-crate. It is emphatically **not** a new compiler, not a new standard library version, and not a fork of the language.

The non-obvious property — and the whole point — is that editions **interoperate freely**: a 2024-edition crate can depend on a 2015-edition crate and vice versa, because the compiler understands all editions simultaneously and lowers them to one common internal representation before type-checking. Editions exist so Rust can make *otherwise-breaking* changes (e.g. promoting `async`/`await`/`dyn` to keywords, changing closure-capture rules) without a Python-2-to-3 ecosystem schism: your old crate keeps compiling on its declared edition forever, and `cargo fix --edition` mechanically migrates you when you choose. Stability is a first-class promise — code that compiles today compiles on every future stable `rustc`. Start new work on 2024 (the `cargo new` default on current toolchains) and don't think about it again.

> **🦀 From your toolbox →** You've lived the painful version of this in **Python** — the 2-to-3 split where a whole ecosystem stalled because old and new code couldn't coexist. Editions are Rust's answer to exactly that pain: think of an edition as a per-file *language level* you opt into, where the twist is that a file at the old level and a file at the new level still link and run together in one program. **Java's** `--release 8` flag is the closest familiar knob, but it sets a global bytecode target, not a per-file source dialect that freely intermixes. The "old code never breaks, new syntax is opt-in, everything still links together" combination is the genuinely novel bit, and it's why upgrading Rust never feels like the Python 3 migration did.

**`rust-analyzer`** is the LSP language server: go-to-definition, inline type hints (essential, since Rust infers most types), and the exact borrow-checker errors *inline as you type* rather than after a build. Install it before you write line one. **docs.rs** auto-builds and hosts API docs for every published crate from its doc-comments; **crates.io** is the registry `cargo add` pulls from; `cargo doc --open` builds the same docs for your own crate locally.

## How to read this book, and why it is ordered this way

This is a re-ordering of *The Rust Programming Language* for someone who already knows how to program and is comfortable with type systems — you. We front-load the one thing that is genuinely new and hard, then let everything else fall out of it.

The spine is the ownership model. [Ownership & moves](02-ownership-and-moves.md) → [references & borrowing](03-references-and-borrowing.md) → [lifetimes as spans](04-lifetimes.md) → [slices](05-slices-and-duality.md) form an unbroken argument; read them in order and do not skip ahead, because *every later chapter assumes you have internalised the borrow checker*. Collections, error handling, closures, and concurrency are not separate features — they are ownership applied to a domain. Fearless [concurrency](13-fearless-concurrency.md), Rust's headline result, is *literally just* the borrow-checker invariant (one writer xor many readers) reinterpreted across threads via the `Send`/`Sync` traits; it costs nothing extra to learn once ownership is solid.

After the spine come the abstraction tools ([structs](06-structs-and-methods.md), [enums & pattern matching](07-enums-and-pattern-matching.md), [generics & traits](08-generics-and-traits.md), [trait objects](09-trait-objects-and-oop.md)). Rust's enums with data will feel like Swift enums with associated values, and pattern matching like OCaml's `match` or Swift's `switch`; traits will feel like Java interfaces with sharper edges. Then come the standard library and the systems-level escape hatches ([smart pointers](15-smart-pointers.md), [unsafe & FFI](16-unsafe-and-ffi.md)), then [advanced traits](17-advanced-traits-and-types.md), [macros](18-macros.md), [project structure](19-project-structure.md), [tooling](20-testing-and-tooling.md), and two [capstones](21-capstones.md). Read the four callout types as signal, not decoration: **🦀 From your toolbox** anchors a new idea to something you already know (and tells you where the analogy lies to you); **🔧 In practice** is a concrete scenario where the feature earns its keep; **⚙️ Under the hood** is what `rustc` actually emits; **⚠️ Pitfall** is the exact compiler error you *will* hit and the idiomatic fix.

## Mental-model recap

- The thesis: memory safety and data-race freedom **at compile time, with no GC and no runtime cost**, via single ownership + compiler-checked borrowing + zero-cost abstractions. Memorise this sentence; the whole book is its proof.
- Ownership forms a **tree**; values are destroyed deterministically when the owning binding leaves scope (like a destructor at end of scope, but enforced by the type system on every value, not by convention). References are non-owning edges, and the borrow checker proves none of them dangle.
- The borrow-checker invariant — **one mutable xor many shared references** — is simultaneously what prevents use-after-free *and* what prevents data races. They are the same rule.
- `cargo check` is your inner loop (analysis, no codegen); `--release` is the only build worth benchmarking; debug builds panic on integer overflow by design.
- An **edition** is a per-crate, freely-interoperable source dialect — *not* a new compiler or std — so Rust evolves syntax without ever breaking old code.

## Exercises

1. You write a function that takes a `String` by value, and at the call site you keep using the variable afterward. The compiler rejects it. *Before* you've read the ownership chapter, predict what the error message complains about, and explain why a managed language like Java or Python would have let the same code through. (Hint: what happened to the original binding when you passed it in?)

2. Your colleague proposes committing `Cargo.lock` for both your application binary *and* your shared library crate "to be consistent." Make the design call: for which one is committing the lockfile the right default, and what concretely goes wrong for *users* of the other if you commit it? Tie your answer to who actually resolves dependency versions in each case.

3. Given the **⚙️ Under the hood** note on profiles, predict the behaviour of `let x: u8 = 255; let y = x + 1;` under `cargo run` versus `cargo run --release`, then answer the design question: a teammate argues "overflow checks should be ON in release too, for safety." Give one concrete reason Rust chose the opposite default, and one situation where you'd override it back on.

4. You have a 2015-edition crate that uses `async` as an ordinary variable name (`let async = ...;`), and you want to depend on it from a brand-new 2024-edition crate of your own. Predict: does the *old* crate still compile? Can your *new* crate use `async` as a variable name? Explain both answers using the single sentence "an edition is an opt-in set of source-language changes, and editions interoperate freely."

5. (★) Without writing any Rust, predict which of these bug categories the borrow checker rejects *at compile time*, and which it cannot catch on its own (needing a runtime smart pointer or `unsafe` even to express): (a) returning a reference to a local variable that's about to be destroyed; (b) two threads writing the same integer with no lock; (c) a shared-ownership cycle that never gets freed; (d) holding a reference into a `Vec` while another line pushes onto it and triggers a reallocation. For each, name the chapter in this book where the relevant machinery shows up — and for the *one* the compiler can't catch by itself, say why the ownership tree alone is insufficient.
