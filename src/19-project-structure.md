# 19. Crates, Modules & Workspaces

You already know several answers to "how do I split a program into pieces and control what's visible." C++ gives you headers, translation units, the One-Definition Rule, `#include`, and namespaces — orthogonal mechanisms you juggle simultaneously. Java fuses the compilation unit, the namespace, and the directory layout into one concept (the package), with `public`/`protected`/`private`/package-private as a separate access lattice. OCaml is cleaner: a module is a structure, and the `.ml`/`.mli` split separates implementation from interface.

Rust's answer is its own, and the headline facts trip up everyone arriving from C++: **there is no preprocessor, no textual inclusion, and no linking object files by name.** A crate compiles as one unit from a single root file. Modules are a purely *logical* namespace tree inside that crate — the mapping to files is a compiler convention, not a fundamental. And visibility is **module-scoped, not type-scoped**: privacy is the boundary of a `mod`, the single biggest delta from Java's per-class `private`.

This chapter builds the vocabulary — package, crate, module, path, visibility, `use` — then scales up to multi-crate workspaces.

## Three nested concepts: package, crate, module

These three words are used loosely in conversation and precisely by the compiler. Get them straight:

- A **crate** is the unit of compilation. It is *one tree of modules* that `rustc` compiles in a single invocation, producing either one executable (a **binary crate**, must have `fn main`) or one library artifact (a **library crate**, no `main`). When people say "the `serde` crate" or "a crate on crates.io," they almost always mean a library crate.
- A **package** is the unit Cargo manages: a directory with a `Cargo.toml`. A package bundles **one or more crates**: at most one library crate, plus any number of binary crates. It is the thing you `cargo new`, version, and publish.
- A **module** is a namespace *inside* a crate. Modules nest, forming the **module tree**, and they are the unit of *privacy*.

> **🦀 From your toolbox →** A C++ *translation unit* is the nearest analogue to a crate, but the analogy breaks on scale and linkage. A TU is typically one `.cpp`; a crate is potentially hundreds of `.rs` files compiled together. There is no ODR across crates because there is no textual `#include` duplicating definitions into many TUs — each crate is compiled once and depended upon by name. A Rust **module** is closest to a C++ `namespace` or an OCaml `module`, a logical naming scope — but unlike a C++ namespace (open, reopenable anywhere) a Rust module's contents are fixed at its definition site.

The file conventions Cargo uses:

```text
my_package/
├── Cargo.toml          # makes this a package
└── src/
    ├── main.rs         # crate root of a binary crate named `my_package`
    ├── lib.rs          # crate root of a library crate named `my_package`
    └── bin/
        └── tool.rs     # crate root of an extra binary crate named `tool`
```

Nothing in `Cargo.toml` names `src/main.rs` or `src/lib.rs`. Cargo *assumes* them by convention: `src/main.rs` is a binary crate root, `src/lib.rs` is a library crate root, and each file under `src/bin/` is its own additional binary crate. A package with both `main.rs` and `lib.rs` has two crates that share the package's name.

> **🦀 From your toolbox →** This is the opposite of C/C++'s explicit build description, where every TU is listed in a `Makefile`/`CMakeLists.txt`. Rust trades configurability for convention: you almost never enumerate source files, because `mod` declarations (below) thread them into the tree automatically. It is closer to how `dune` discovers OCaml modules, or how Java maps `com.example.Foo` to `com/example/Foo.java` — but Rust's mapping is far more flexible than Java's rigid package-equals-directory rule.

## The module tree

The crate root file (`lib.rs` or `main.rs`) is itself an implicit module called `crate`. Every other module hangs off it. You introduce a module with `mod`, and you may nest inline with braces:

```rust
// src/lib.rs — crate root, implicitly module `crate`
mod billing {
    pub mod invoices {
        pub struct Invoice { pub total_cents: u64 }
        pub fn issue() -> Invoice { Invoice { total_cents: 0 } }
    }
    mod ledger {              // private to `billing`
        fn post() {}
    }
}
```

This produces the tree:

```text
crate
└── billing
    ├── invoices
    │   ├── Invoice
    │   └── issue
    └── ledger
        └── post
```

The vocabulary mirrors a filesystem and is worth fixing: `invoices` and `ledger` are **siblings** (defined in the same parent); `invoices` is a **child** of `billing`; `billing` is the **parent** of `invoices`. The whole tree is rooted at `crate`.

> **🎓 Tripos link →** In *Foundations of Computer Science* you used OCaml modules to bundle a type with the operations over it (the classic `Stack` / `Queue` ADT presentation). A Rust module serves the identical *organisational* role, but the encapsulation mechanism differs: OCaml hides representation by *omitting* it from an `.mli` signature, whereas Rust hides items by *default privacy* and selectively *exposes* with `pub`. Same goal — abstraction boundaries the compiler enforces — opposite default (OCaml: list it to export with a signature; Rust: everything private until annotated).

## Modules are not files

This is where C++ intuition misleads you. In C++, `#include "foo.h"` is *textual substitution* — the preprocessor pastes the file's bytes into the current TU; the file *is* the unit. In Rust, **the module tree is logical and exists independently of how it is spread across files.** The line

```rust
mod billing;   // note: semicolon, not a brace block
```

is not an include. It declares: *"a module named `billing` exists here in the tree; go find its body."* The compiler then looks, in order, in:

- `src/billing.rs` (modern, idiomatic), or
- `src/billing/mod.rs` (older style, still supported).

For a *submodule* declared inside `billing` — say `mod invoices;` written in `src/billing.rs` — the compiler looks in `src/billing/invoices.rs` or `src/billing/invoices/mod.rs`. The directory nesting tracks the module nesting.

Two rules that follow, and that catch newcomers:

1. **You declare a file into the tree exactly once,** with one `mod` statement, at the position in the tree where you want it. Other code then refers to its contents *by path*, never by re-declaring `mod` again. Writing `mod billing;` in two places does not "include it twice" — it tries to create two modules and is usually an error.
2. **`use` does not pull files into the build.** Only `mod` does. `use` merely creates a local shorthand for an already-existing path (more below). A file with no `mod` declaration pointing at it is simply not part of the crate.

> **⚠️ Pitfall →** Coming from C, you write `use crate::billing;` expecting it to "import the module" the way `#include` makes a file's contents available — but you never wrote `mod billing;`. The error is `error[E0432]: unresolved import` or `failed to resolve: use of unresolved module`. The fix is to add `mod billing;` in the crate root (or appropriate parent) to *attach the file to the tree*; `use` only shortens a path that already resolves.

> **⚙️ Under the hood →** Because there is no textual inclusion, a generic definition in module `a` and its use site in module `b` are seen by `rustc` in one pass over one crate. Monomorphisation (the machinery from *Compiler Construction*) runs with full visibility of every instantiation — none of the "template lives in the header, use lives in another TU" fragility. Cross-crate generics work because the library crate ships its generic bodies in metadata, not by re-parsing headers.

Whether to use inline `mod foo { ... }` or file-based `mod foo;` is purely stylistic — the resulting tree is identical. Inline is convenient for small or test modules; files for anything that grows.

## Paths: naming an item

To refer to an item across the tree you write a **path**, segments separated by `::`. There are two flavours:

- **Absolute**, starting from a crate root: `crate::billing::invoices::issue` for the current crate, or `some_dep::module::item` beginning with an *external crate's name*.
- **Relative**, starting from the current module: a bare identifier, or `self::` (this module) or `super::` (parent module — exactly like `..` in a shell path).

```rust
mod billing {
    pub mod invoices {
        pub fn issue() {}
    }
    pub fn run_books() {
        // relative, via a child
        invoices::issue();
        // relative, explicit self
        self::invoices::issue();
        // absolute, from the root
        crate::billing::invoices::issue();
    }
}

mod reporting {
    pub fn summarise() {
        // `super` = parent of `reporting` = crate root
        super::billing::run_books();
    }
}
```

Which to prefer? **Default to absolute (`crate::…`).** When you later move the *calling* code, an absolute path keeps resolving (the target didn't move); when you move the *definition*, you fix it either way. Since refactors move callers more often than definitions, absolute paths break less. Reach for `super::` only when a child is tightly coupled to its parent and you expect them to move together.

> **🦀 From your toolbox →** `crate::` is Rust's analogue of a leading `/` (filesystem root) or a fully-qualified Java name `com.example.billing.Invoices`. `super::` is `..`. The break from Java: there is no implicit "same package, no qualification needed" — siblings still need a path segment, and crucially the path is checked against *privacy* at every segment, which Java's package-private does not do hierarchically.

## Visibility: private by default, and module-scoped

Every item — function, struct, enum, module, constant, method — is **private by default**, meaning *visible only within its defining module and that module's descendants*. The exact rule, and it is the one to memorise:

- A **child module can see everything in its ancestors,** including private items. (A child sees the context it was defined in.)
- A **parent cannot see the private contents of its children.** (Children hide their internals.)

`pub` flips an item to public. But here is the subtlety that produces a two-stage compiler error in practice: making a *module* `pub` only lets ancestors *name* it; it does **not** make the items inside it public. You must mark each exposed item `pub` as well.

```rust
mod billing {
    pub mod invoices {       // module is reachable...
        pub fn issue() {}    // ...and this item is reachable
        fn audit() {}        // ...but this stays private to `invoices`
    }
}

pub fn run() {
    billing::invoices::issue();   // OK
    // billing::invoices::audit(); // E0603: function `audit` is private
}
```

> **⚠️ Pitfall →** You add `pub` to `mod invoices` and re-run, expecting success, and instead get `error[E0603]: function `issue` is private` pointing at the *function*. The mental bug is treating a module like a Java class where one access modifier on the container governs members. In Rust, `pub mod` and `pub fn` are independent gates — the path must be public *at every segment*. Fix: annotate each item you intend to export, not just the module.

For structs, `pub struct` exposes the type but **each field stays private unless individually `pub`** — so you can publish a type whose representation is sealed, exactly the OCaml abstract-type pattern. Enums are the exception: `pub enum` makes **all variants public**, because an enum with hidden variants is almost useless to a caller (you cannot pattern-match on what you cannot name).

```rust
pub struct Account {
    pub name: String,   // callers may read/write
    balance_cents: i64,  // sealed: only this module touches it
}

impl Account {
    pub fn open(name: String) -> Account {
        Account { name, balance_cents: 0 }   // must be built in-module
    }
    pub fn balance(&self) -> i64 { self.balance_cents }
}
```

Because `balance_cents` is private, external code cannot construct an `Account` with a struct literal nor mutate the balance directly — it must go through `open` and methods. Encapsulation is enforced *by the module boundary*, which is why the constructor lives where the private field lives.

> **🎓 Tripos link →** This is the same encapsulation guarantee you reasoned about informally in *Further Java*, but with a sharper formal property. In *Semantics of Programming Languages* terms, module privacy is a *static* representation-hiding discipline: a well-typed external client provably cannot observe or depend on private state, so you may change `balance_cents` (its type, its representation, even split it into two fields) without breaking any compiling client. That is a soundness-style invariant the compiler maintains, not a runtime convention.

### Finer-grained visibility

`pub` means "public to anyone who can reach the path," including other crates. Often you want narrower scopes. Rust offers a small lattice:

```rust
pub(crate) fn internal_helper() {}     // visible anywhere in THIS crate, never exported
pub(super) fn for_my_parent() {}       // visible to the parent module
pub(in crate::billing) fn scoped() {}  // visible within a named ancestor module path
```

`pub(crate)` is the workhorse: it is how you share an item freely across your own crate's modules while keeping it out of your *public API*. Reach for it constantly when building a library — most of your "internal but cross-module" functions want `pub(crate)`, not bare `pub`.

> **🦀 From your toolbox →** Java's four levels (`public`, `protected`, package-private, `private`) map roughly but not cleanly: `private` ≈ Rust default *at the module level* (not the type level!), package-private ≈ `pub(crate)` in spirit, `public` ≈ `pub`. There is no `protected` because Rust has no inheritance (see [trait objects & OOP](09-trait-objects-and-oop.md)). The decisive difference: in Java `private` is per-*class*, so two classes in the same file cannot see each other's privates; in Rust privacy is per-*module*, so two `struct`s in the same module freely access each other's private fields. This makes the "module = friend group" pattern idiomatic where C++ would reach for `friend`.

## `use`: shortcuts into scope

Writing `crate::billing::invoices::issue()` everywhere is noise. `use` binds a path to a short name *in the current scope*:

```rust
use crate::billing::invoices;

pub fn run() {
    invoices::issue();   // `invoices` now names that module here
}
```

A `use` is like a symbolic link, and it is **scoped to the block it appears in** — it does not leak into child modules. It is also subject to privacy: `use` cannot reach a private item any more than a full path can.

**Idiomatic granularity.** The community convention — and it is a real convention worth following for readability, not a rule the compiler cares about:

- For **functions**, `use` the *parent module* and call `parent::func()`. Seeing `invoices::issue()` tells the reader at the call site that `issue` is not local. `use …::issue;` and a bare `issue()` hides that.
- For **structs, enums, traits, and other types**, `use` the full path: `use std::collections::HashMap;` then `HashMap`. Types are typically named often and unambiguously, so the extra segment is just noise.

```rust
use std::collections::HashMap;   // type: full path
use crate::billing::invoices;    // function's parent module
```

**Name clashes.** Two items with the same final name cannot both be `use`d bare into one scope. Two idiomatic fixes:

```rust
// (a) keep the parent modules and disambiguate at the call site
use std::fmt;
use std::io;
fn f() -> fmt::Result { Ok(()) }
fn g() -> io::Result<()> { Ok(()) }

// (b) rename one with `as`
use std::fmt::Result;
use std::io::Result as IoResult;
fn h() -> Result { Ok(()) }
fn k() -> IoResult<()> { Ok(()) }
```

**Nested paths** collapse repetitive `use` lists, and `self` lets a module and one of its children share a line:

```rust
use std::{cmp::Ordering, io};        // std::cmp::Ordering AND std::io
use std::io::{self, Write};          // std::io AND std::io::Write
```

**The glob `*`** pulls in *all* public items of a path:

```rust
use std::collections::*;
```

Use it sparingly. It obscures where names came from, and it makes you fragile to upstream additions: if a dependency later adds a public name that collides with one of yours, your build breaks on upgrade. The two sanctioned uses are (1) `use super::*;` inside a `#[cfg(test)]` test module to grab everything under test, and (2) deliberate *prelude* modules designed to be glob-imported (below).

> **🦀 From your toolbox →** `use` is *not* `#include` and *not* Java's `import` in effect on the build. Java's `import` is also just a naming shorthand (the class is found via the classpath regardless), so that analogy holds — but in C++, omitting an `#include` means the symbol genuinely does not exist in the TU. In Rust, the item exists in the crate as soon as `mod` attaches it; `use` only abbreviates. Forgetting a `use` is a *spelling* fix (write the full path), never a *linkage* error.

## Re-exports: `pub use` shapes your public API

By default a `use` binding is private — it is your local convenience. Prefix it with `pub` and you **re-export**: the name becomes part of *your* module's public surface, as if defined there. This decouples your *internal* module layout from the API your callers see.

```rust
// src/lib.rs
mod billing {
    pub mod invoices {
        pub fn issue() {}
    }
}

// Internally organised under billing::invoices,
// but presented to the world at the crate root:
pub use crate::billing::invoices;
```

A caller now writes `my_crate::invoices::issue()` instead of `my_crate::billing::invoices::issue()` and never learns `billing` exists. This is the standard technique for a library that wants a flat, curated front door over a deep internal hierarchy — your contributors navigate `billing::invoices`, your users see a tidy top-level API. It is also how a crate provides a single import point: re-export the handful of types and traits most users need from the root.

> **⚙️ Under the hood →** `pub use` re-exports also flow into generated documentation: `cargo doc` lists a re-exported item at the re-export site, so your published docs reflect the curated structure, not the raw module tree. This is why mature crates (`tokio`, `serde`) feel coherent despite large internal module trees — the public path geometry is hand-shaped with `pub use`, independent of where things actually live.

## External crates and the prelude

Adding a dependency is two steps. First, declare it in `Cargo.toml`:

```toml
[dependencies]
rand = "0.8.5"
```

Cargo resolves it from crates.io (or a registry/git/path source), downloads it, and pins the exact resolved version in `Cargo.lock`. Second, refer to it by crate name in paths — an external crate's name is the first segment of an absolute path to its items:

```rust
use rand::Rng;                 // bring the trait into scope
let n = rand::thread_rng().gen_range(1..=100);
```

The standard library `std` is special only in that it ships with the toolchain, so it needs no `Cargo.toml` entry — but it is otherwise an ordinary external crate: you still write `use std::collections::HashMap;`, an absolute path rooted at the crate `std`.

**The prelude** is why you never `use std::option::Option;`. The standard library defines a small set of ubiquitous items — `Option`, `Result`, `Vec`, `String`, `Box`, `Clone`, `Iterator`, `drop`, and so on — in `std::prelude`, and the compiler glob-imports it into *every* module automatically: the prelude pattern applied by the language itself. Crates can ship their own opt-in prelude (`use some_crate::prelude::*;`) for the same purpose — one glob bringing the crate's most-used traits into scope, which matters because trait methods are only callable when the trait is in scope (see [generics & traits](08-generics-and-traits.md)).

> **🎓 Tripos link →** The automatic prelude is the language pre-populating the typing context Γ that you formalised in *Semantics of Programming Languages*. Every module begins not with an empty environment but with the prelude's bindings already in Γ, which is why bare `Option` type-checks. A crate prelude is the same move at library granularity — extending the ambient context so the common case needs no explicit `use`.

## Workspaces: many crates, one build

A single library crate is the right granularity for a long time. You split into multiple crates when you have a concrete reason:

- **API boundaries.** A separate crate has a hard public/private wall (only `pub` items cross it), which enforces layering far more firmly than module privacy within one crate.
- **Compile times.** A crate is the unit of *incremental recompilation* and the unit of *parallelism* in Cargo's build graph. Editing one crate need not rebuild the others (only re-link/re-check dependents). Splitting a monolith into crates lets `rustc` rebuild and the build parallelise across the unchanged ones.
- **Independent reuse or publishing.** Each crate publishes to crates.io separately.

A **workspace** is Cargo's mechanism for developing several related packages together. They share **one `Cargo.lock`** and **one `target/` output directory**. The workspace root has a `Cargo.toml` with a `[workspace]` table and *no* `[package]`:

```toml
# Cargo.toml at the workspace root
[workspace]
resolver = "3"
members = ["app", "domain", "storage"]
```

A plausible layout: a binary `app` depending on two libraries, `domain` and `storage`. Crucially, **a workspace does not assume its members depend on each other** — you must state intra-workspace dependencies explicitly with a path:

```toml
# app/Cargo.toml
[dependencies]
domain  = { path = "../domain" }
storage = { path = "../storage" }
```

```rust
// app/src/main.rs
fn main() {
    let order = domain::Order::new();
    storage::persist(&order);
}
```

Build the whole graph with `cargo build` from the root; target a single member with `-p`:

```bash
cargo build              # build every member
cargo run -p app         # run the binary in package `app`
cargo test               # test every member (unit, integration, doc tests)
cargo test -p domain     # test only `domain`
```

> **⚙️ Under the hood →** The single shared `target/` is the load-bearing design choice. If each member had its own `target/`, then `app` depending on `domain` would force a *separate* compilation of `domain` into `app`'s target — every dependent re-compiling every dependency. One shared output directory means `domain` is built **once** and its artifact reused by all dependents, which is exactly the build-graph deduplication that makes large workspaces tractable.

The shared **`Cargo.lock`** is the other half: every member resolves against *one* unified dependency graph, so if both `domain` and `storage` depend on `rand`, they get the *same* resolved version, recorded once, downloaded once. The members are therefore mutually compatible — you cannot accidentally link two copies of a shared dependency at incompatible versions (Cargo resolves to as few versions as semver allows). A frequent surprise on the flip side: a dependency declared in `domain/Cargo.toml` is **not** usable from `storage` — each member lists its own `[dependencies]`. The lockfile unifies *versions*, not *visibility*.

> **🎓 Tripos link →** A workspace's build is a DAG of crates, each a node compiled once after its dependencies — precisely the dependency-ordered compilation you analysed in *Compiler Construction*, and the topological scheduling Cargo performs is the same problem as instruction/phase scheduling over a DAG. The single `target/` is content-addressed caching over that DAG: a node is recompiled only when its inputs (source or upstream artifacts) change.

## Mental-model recap

- **Crate = compilation unit** (one module tree → one binary or one library). **Package = Cargo unit** (a `Cargo.toml` bundling ≤1 library + N binaries). **Module = in-crate namespace and the privacy boundary.** Keep the three distinct.
- **`mod` builds the tree; `use` abbreviates a path; neither is `#include`.** A file joins the crate only via a single `mod` declaration; `use` merely shortens an already-resolving path. Forgetting `use` is a spelling fix, forgetting `mod` is a structural one.
- **Privacy is module-scoped and private-by-default.** Children see ancestors; parents cannot see children's privates. `pub mod` exposes the *name* only — each item still needs its own `pub`. Reach for `pub(crate)` for cross-module-but-not-public.
- **`pub use` decouples internal layout from public API** — curate your front door independently of where code lives; it shapes `cargo doc` too.
- **Split into crates for API walls, compile times, and reuse; use a workspace** so members share one `Cargo.lock` and one `target/`. Intra-workspace deps and per-crate `[dependencies]` are still explicit.

## Exercises

1. Create a binary package with `src/main.rs` that declares `mod config;` and calls `config::load()`. Write `config.rs` containing a *private* `fn load()`. Predict the exact error code before compiling, then fix it with the minimal change. Now move `config.rs` to `src/config/mod.rs` and confirm the build still works — explain why the module path is unchanged.

2. In one file, write `mod a { pub(super) fn helper() {} }` at the crate root and try to call `a::helper()` from `main`. Does it compile? Now nest `a` inside `mod outer { mod a { ... } }` and call `helper` from `outer`. Explain what `pub(super)` resolves to in each case.

3. Define `pub struct Meter { value: f64 }` in a module `units` with a private field, and a function `meters(x: f64) -> Meter`. From `main`, try (a) `units::Meter { value: 3.0 }` and (b) `units::meters(3.0).value`. Both fail — give the two distinct error codes and explain why the private field forbids each independently.

4. You have `use std::fmt::Display;` and `use crate::ui::Display;` (your own type) in the same module. State the error, then resolve it two different ways — once with `as`, once by keeping a parent-module path on one of them.

5. (★) Build a library crate with internal modules `net::tcp` and `net::http`, where `http` has a `pub(crate)` function that `tcp` must call but no external user should reach. Then add a `pub use` at the crate root that re-exports `http::Client` as `my_crate::Client`. Write a doc comment and run `cargo doc --open`; confirm `Client` appears at the root and the `pub(crate)` function appears nowhere in the docs. Explain the two different visibility decisions you made and why bare `pub` would have been wrong for the helper.

6. (★) Create a workspace with members `core` (lib) and `cli` (bin) where `cli` depends on `core` via a path dependency. Add `rand = "0.8"` to `core` only. Now write `use rand;` in `cli/src/main.rs` and predict the error. Fix it the *correct* way (not by removing the line). Then run `cargo build`, inspect the top-level `Cargo.lock`, and confirm `rand` appears exactly once even though two members now depend on it — explain what guarantee that single entry provides.
