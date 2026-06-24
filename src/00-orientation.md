# 0. Orientation: the thesis of Rust

You already know how to write fast code (C, C++) and how to write safe code (a GC'd runtime: Java, Swift's ARC, JavaScript). The industry consensus you grew up with is that these are a trade-off: you pick manual memory management and accept use-after-free, double-free, and data races as the price of control; or you accept a garbage collector — a runtime that periodically traces the heap — and pay with latency, throughput, footprint, and loss of deterministic destruction. Rust's entire reason to exist is the claim that this trade-off is not fundamental. It is an artifact of weak type systems.

## The one-sentence thesis

**Rust achieves memory safety and data-race freedom at compile time, with no garbage collector and no runtime overhead, by enforcing an affine type discipline ("ownership"), region-checked references ("lifetimes" / "borrowing"), and abstractions that monomorphise away to zero cost.**

Unpack each clause; the rest of this book is just these four ideas pursued relentlessly.

**Affine type discipline.** From [Logic and Proof](#) you know linear logic: a linear resource must be used exactly once. Rust uses the *affine* variant — a value may be used *at most* once before it is consumed (moved) or goes out of scope. Every value has exactly one *owner*. When the owner's binding leaves scope, the value is deterministically destroyed. This is RAII from C++, but promoted from a library convention you opt into (`unique_ptr`, destructors) to a property the type system enforces on *every* value. A `move` in Rust is the default for non-`Copy` types, and after a move the source binding is statically dead — the compiler refuses to let you read it. There is no `std::move`-leaves-a-valid-empty-husk subtlety as in C++; the moved-from variable is simply not usable.

**Region-checked references.** You usually don't want to *consume* a value to read it. So Rust lets you *borrow* it, producing a reference whose validity is bounded by a *region* — a lexical-ish span of the program — called a lifetime. From [Semantics of Programming Languages](#) this is exactly region-based memory management formalised by Tofte and Talpin, except Rust *infers* the regions and uses them only for static checking, not for allocation. The borrow checker enforces one invariant: at any program point, you may have either *one* mutable reference *or* any number of shared (immutable) references to a given value, never both. This is the readers-writers lock — but resolved at compile time, with zero runtime cost, and it is precisely what makes data races impossible. We develop this in [references](03-references-and-borrowing.md) and [lifetimes](04-lifetimes.md).

**Zero-cost abstractions.** Generics monomorphise (à la C++ templates and as you saw in [Compiler Construction](#)): `Vec<T>` generates specialised machine code per instantiation, no boxing, no type erasure. Traits dispatch statically by default; you opt into dynamic dispatch (a vtable) explicitly with `dyn`. Iterators, closures, `Option`, and `Result` compile down to the same loops and branches you'd write by hand. The slogan, due to Stroustrup and adopted wholesale by Rust: *what you don't use, you don't pay for; what you do use, you couldn't hand-code better.*

> **🦀 From your toolbox →** Think of Rust as "C++ with the parts you keep tripping over removed by the type checker." RAII, move semantics, monomorphised templates, no GC, deterministic destructors — all present and familiar. What's new is that the compiler *statically proves* the lifetime invariants you currently maintain in C++ by discipline, code review, and AddressSanitizer. **Where the analogy breaks:** in C++ a dangling reference or a data race is undefined behaviour the compiler happily emits; in safe Rust it is a compile error. And unlike OCaml's safety-via-GC, Rust pays no runtime price for that guarantee.

> **🎓 Tripos link →** *Programming in C and C++* taught you that `unique_ptr` models unique ownership and that a moved-from `unique_ptr` is null. Rust takes that exact model and makes it the language's default semantics for *all* values, then adds a static checker (the borrow checker) so that the C++ failure modes — using a moved-from object, holding a reference past the owner's destruction — become type errors rather than latent bugs.

## The central mental model: ownership is a tree, borrows are a proof

Commit to this picture now; everything hangs off it.

Every value is owned by exactly one binding at a time. Composite values own their parts: a `Vec<String>` owns its heap buffer, which owns each `String`, which owns *its* heap buffer. Ownership therefore forms a **tree** (more precisely a forest rooted at stack frames). When a root goes out of scope, destruction recurses down the tree — deterministically, at a statically known program point, with no tracing and no reference counting.

References (`&T`, `&mut T`) are **non-owning** edges that point *into* this tree. The borrow checker's job is to prove that no reference outlives the subtree it points at, and that mutable access is exclusive. A program that type-checks comes with a machine-verified proof that it has no dangling pointers and no data races. That is the deal Rust offers: accept the discipline, receive the proof.

When the tree shape is genuinely insufficient — you need shared ownership, cycles, or a graph — you reach for [smart pointers](15-smart-pointers.md) (`Rc`, `RefCell`, `Arc`, `Mutex`) that move some checks to runtime, or, at the very edge, [`unsafe`](16-unsafe-and-ffi.md) where you discharge the proof obligation yourself. But the tree is the default, and idiomatic Rust keeps it that way far longer than your C++ instincts will predict.

## A glance at Hello, World

You will spend almost no time here; you've written `main` thousands of times.

```rust
fn main() {
    println!("Hello, world!");
}
```

Three things to register and move on:

- `fn main()` is the entry point, as in C. No arguments needed; the OS-level `argc/argv` are reached through `std::env`.
- `println!` is a **macro**, not a function — the `!` is the tell. It is not C's `printf`: the format string is parsed and type-checked at compile time, so a mismatch between placeholders and arguments is a compile error, not a runtime format-string vulnerability. Macros are [their own chapter](18-macros.md); for now, `!` just means "expands to code."
- Statements end in `;`. Rust is expression-oriented (a block evaluates to its last expression), so the `;` is semantically meaningful — it discards a value, turning an expression into a statement. We make this precise in [the language delta](01-language-delta.md).

Compile directly with `rustc main.rs` (like invoking `clang` by hand) and you get a standalone native binary — no interpreter, no VM, no `.jar`, statically linked against the Rust standard library by default. But nobody invokes `rustc` directly past this paragraph.

## The toolchain at expert speed

**`rustup`** is the toolchain multiplexer — conceptually `nvm`/`pyenv`/`rbenv` for Rust. It installs and switches between *toolchains* (a toolchain = a `rustc` + `cargo` + `std` of a given channel: `stable`, `beta`, `nightly`, or a pinned version like `1.96.0`). A directory can pin its toolchain via `rust-toolchain.toml`. `rustup update` pulls new releases (stable ships every six weeks); `rustup component add` fetches `clippy`, `rustfmt`, `rust-src`, etc.; `rustup doc` opens the *offline* copy of std and the book in your browser. Target cross-compilation (`rustup target add aarch64-unknown-linux-gnu`) lives here too.

**`cargo`** is build system *and* package manager in one — `make` + a dependency resolver + a test runner, with conventions instead of configuration. A project (a *package*) has a `Cargo.toml` manifest (TOML: name, version, `edition`, `[dependencies]`) and a `Cargo.lock` (the resolved, exact-version dependency graph — commit it for binaries, conventionally not for libraries). Source lives in `src/`; build artifacts in `target/`. The commands you will actually live in:

| Command | What it does | Analogy |
|---|---|---|
| `cargo new <name>` | Scaffold package (`Cargo.toml`, `src/main.rs`, git init) | — |
| `cargo check` | Type-check + borrow-check, **emit no binary** | the fast inner loop |
| `cargo build` | Compile to `target/debug/` (unoptimised + debuginfo) | `make` |
| `cargo build --release` | Optimised build to `target/release/` | `make` with `-O` |
| `cargo run` | `build` then execute (rebuilds only if stale) | — |
| `cargo test` | Build and run all tests (see [testing](20-testing-and-tooling.md)) | — |
| `cargo clippy` | Lint: hundreds of idiom/correctness/perf lints | a Rust-aware linter |
| `cargo fmt` | Canonical formatting (rustfmt) | `clang-format`, but there is one true style |

`cargo check` is the command you run on every save: it does all the analysis that catches your bugs (type, borrow, lifetime) and *skips* codegen and linking, so it returns in a fraction of `build`'s time. Reserve `build`/`run` for when you actually want to execute.

> **⚙️ Under the hood →** The debug/release split is two *profiles*. Debug is `opt-level = 0` with debuginfo and overflow checks on, so it *panics on integer overflow* rather than wrapping. Release is `opt-level = 3` with overflow checks off (wrapping in release is defined two's-complement, not UB). The semantics difference is deliberate: it surfaces bugs in development without costing a branch in production. Benchmark only release builds — a debug binary can be 10–50× slower and will mislead you completely.

### Editions: what they are and are not

`Cargo.toml` carries an `edition` key — `2015`, `2018`, `2021`, or `2024`. An **edition is an opt-in set of source-language changes** (new keywords, changed defaults, syntax) scoped per-crate. It is emphatically **not** a new compiler, not a new standard library version, and not a fork of the language.

The non-obvious property — and the whole point — is that editions **interoperate freely**: a 2024-edition crate can depend on a 2015-edition crate and vice versa, because the compiler understands all editions simultaneously and lowers them to one common internal representation before type-checking. Editions exist so Rust can make *otherwise-breaking* changes (e.g. promoting `async`/`await`/`dyn` to keywords, changing closure-capture rules) without a Python-2-to-3 ecosystem schism: your old crate keeps compiling on its declared edition forever, and `cargo fix --edition` mechanically migrates you when you choose. Stability is a first-class promise — code that compiles today compiles on every future stable `rustc`. Start new work on 2024 (the `cargo new` default on current toolchains) and don't think about it again.

> **🦀 From your toolbox →** Editions are closest to a C/C++ language standard (`-std=c++17` vs `c++20`) *selected per translation unit but link-compatible across the whole program* — which C++ does **not** guarantee (mixing ABIs/standards across object files is fraught). Java has no analogue: a `--release 8` flag changes the bytecode target globally, not a per-file source dialect that freely intermixes. The "old code never breaks, new syntax is opt-in, everything still links" combination is the genuinely novel bit.

**`rust-analyzer`** is the LSP language server: go-to-definition, inline type hints (essential, since Rust infers most types), and the exact borrow-checker errors *inline as you type* rather than after a build. Install it before you write line one. **docs.rs** auto-builds and hosts API docs for every published crate from its doc-comments; **crates.io** is the registry `cargo add` pulls from; `cargo doc --open` builds the same docs for your own crate locally.

## How to read this book, and why it is ordered this way

This is a re-ordering of *The Rust Programming Language* for someone who already knows systems programming and type theory — you. We front-load the one thing that is genuinely new and hard, then let everything else fall out of it.

The spine is the ownership model. [Ownership & moves](02-ownership-and-moves.md) → [references & borrowing](03-references-and-borrowing.md) → [lifetimes as regions](04-lifetimes.md) → [slices](05-slices-and-duality.md) form an unbroken argument; read them in order and do not skip ahead, because *every later chapter assumes you have internalised the borrow checker*. Collections, error handling, closures, and concurrency are not separate features — they are ownership applied to a domain. Fearless [concurrency](13-fearless-concurrency.md), Rust's headline result, is *literally just* the borrow-checker invariant (one writer xor many readers) reinterpreted across threads via the `Send`/`Sync` traits; it costs nothing extra to learn once ownership is solid.

After the spine come the abstraction tools ([structs](06-structs-and-methods.md), [enums & pattern matching](07-enums-and-pattern-matching.md), [generics & traits](08-generics-and-traits.md), [trait objects](09-trait-objects-and-oop.md)), which will feel like OCaml sum types and Java interfaces with sharper edges, then the standard library and the systems-level escape hatches ([smart pointers](15-smart-pointers.md), [unsafe & FFI](16-unsafe-and-ffi.md)), then [advanced traits](17-advanced-traits-and-types.md), [macros](18-macros.md), [project structure](19-project-structure.md), [tooling](20-testing-and-tooling.md), and two [capstones](21-capstones.md). Read the four callout types as signal, not decoration: **🦀 From your toolbox** anchors a new idea to something you already know (and tells you where the analogy lies to you); **⚙️ Under the hood** is what `rustc` actually emits; **⚠️ Pitfall** is the exact compiler error you *will* hit and the idiomatic fix; **🎓 Tripos link** ties back to your courses.

## Mental-model recap

- The thesis: memory safety and data-race freedom **at compile time, with no GC and no runtime cost**, via affine ownership + region-checked borrowing + zero-cost abstractions. Memorise this sentence; the whole book is its proof.
- Ownership forms a **tree**; values are destroyed deterministically when the owning binding leaves scope (RAII, enforced by the type system, not by convention). References are non-owning edges, and the borrow checker proves none of them dangle.
- The borrow-checker invariant — **one mutable xor many shared references** — is simultaneously what prevents use-after-free *and* what prevents data races. They are the same rule.
- `cargo check` is your inner loop (analysis, no codegen); `--release` is the only build worth benchmarking; debug builds panic on integer overflow by design.
- An **edition** is a per-crate, freely-interoperable source dialect — *not* a new compiler or std — so Rust evolves syntax without ever breaking old code.

## Exercises

1. Run `cargo new orientation && cd orientation`, then `cargo check`, `cargo build`, and `cargo run`. Observe which produce output under `target/` and which don't; explain from the table above *why* `check` is faster than `build`.
2. Open `Cargo.toml`, change `edition = "2024"` to `edition = "2015"`, and add a binding named `dyn` (e.g. `let dyn = 5;`) in `main`. Build under each edition and explain the differing results in terms of "an edition is an opt-in set of source-language changes" — specifically, when did `dyn` become a reserved keyword?
3. Write a `main` that computes `let x: u8 = 255; let y = x + 1;` and run it with `cargo run` and then `cargo run --release`. Explain the two different behaviours using the **⚙️ Under the hood** note on profiles. Which behaviour would you *want* in a security-critical context, and why does Rust make development the strict one?
4. (★) Without writing any Rust, predict precisely which of these C++ bug categories the borrow checker rejects *at compile time* and which it cannot (and would need `unsafe` or a runtime smart pointer to even express): (a) returning a pointer to a local stack variable; (b) two threads writing the same `int` without a lock; (c) a reference-counted cycle that leaks; (d) iterator invalidation from `push_back` during a range-for. For each, name the chapter in this book where the relevant machinery appears.
5. (★) `rustup doc --std` opens the offline standard-library docs. Find the page for `std::mem::drop` and for `Drop`. From the signatures and prose alone, articulate how "destruction recurses down the ownership tree" is realised — what does `drop(x)` do that letting `x` fall out of scope does not, and why is it sound to call it early?
