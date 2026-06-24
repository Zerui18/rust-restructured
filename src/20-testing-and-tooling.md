# 20. Testing & Tooling

Every language you already know bolts testing on from outside: JUnit, GoogleTest, Catch2, OUnit, XCTest, Jest. The framework is a third-party citizen — it ships its own test runner, its own assertion vocabulary, its own discovery rules, and it has *no* privileged relationship with the compiler. Rust inverts this. Testing is a feature of `rustc` and `cargo`, not a library you add. `#[test]` is a compiler-recognised attribute; `cargo test` is a first-class subcommand that compiles a *separate* binary in a special configuration and runs it. There is nothing to install and nothing to configure to get a parallel test runner with output capture, filtering, and a results summary.

This chapter covers two things the brief groups together because they are the daily reality of working in Rust: the built-in test machinery (unit, integration, and the unique *doc tests*), and the broader toolchain that surrounds `cargo build` — `clippy`, `rustfmt`, `rust-analyzer`, `rustdoc`, release profiles, features, publishing, and extending Cargo itself. Treat the second half as a checklist you return to.

## Anatomy of a test

A test is a function annotated `#[test]` that takes no arguments. The convention — and `cargo new --lib` generates it for you — is to gather them in a child module gated by `#[cfg(test)]`:

```rust
pub fn checksum(bytes: &[u8]) -> u32 {
    bytes.iter().fold(0u32, |acc, &b| acc.wrapping_add(b as u32))
}

#[cfg(test)]
mod tests {
    use super::*; // pull the parent module's items into scope

    #[test]
    fn empty_slice_is_zero() {
        assert_eq!(checksum(&[]), 0);
    }

    #[test]
    fn sums_bytes() {
        assert_eq!(checksum(&[1, 2, 3]), 6);
    }
}
```

A test *passes* if the function returns normally and *fails* if it panics. That is the whole protocol. Every assertion macro is just a conditional `panic!`. `cargo test` compiles a dedicated test-runner binary that calls each `#[test]` function, catches the panic (each test runs on its own thread, so the runner observes the thread dying), and tallies the results.

`#[cfg(test)]` is the load-bearing attribute. `cfg` is conditional compilation: the `tests` module — and any helper functions inside it — is compiled *only* when the `test` cfg flag is set, which Cargo does for `cargo test` but not for `cargo build`. So your test code, and any test-only dependencies, are entirely absent from the shipped artifact. This is compile-time dead-code elimination by configuration, not by the optimiser.

> **🦀 From your toolbox →** Think JUnit, but with the visibility headache removed. In Java a `@Test` method in `src/test/java` cannot see a `private` field without reflection hacks, so unit tests get pushed onto the public surface; in Python you can reach in, but only because nothing is truly private. Rust's `tests` module is a *child* module of the code under test, and a child module can name its ancestors' private items — so you test privates directly, with no ceremony. The closest thing you've used is Swift's `@testable import`, which lets a test target see `internal` members of another module; the difference is that Swift's version is an opt-in linker mode you switch on per import, whereas in Rust it falls out of the ordinary module-visibility rules with nothing to enable. Where the analogy breaks down: there is no separate "test target" project here at all — the test module lives in the same file and is simply compiled out of release builds.

### Assertion macros

Three macros, all from `std`, all just panic on failure:

- `assert!(cond)` — fails if `cond` is `false`. Use for arbitrary booleans.
- `assert_eq!(a, b)` / `assert_ne!(a, b)` — fail if the two are (un)equal, and on failure *print both values*. This is why you reach for them over `assert!(a == b)`: the diagnostic shows `left` and `right`, not just "assertion failed".

Note the **trait obligations** baked into the equality macros. To compare, the operands need `PartialEq`; to *print* them on failure, they need `Debug`. For a primitive both are free. For your own type you must opt in:

```rust
#[derive(Debug, PartialEq)]
struct Version { major: u16, minor: u16 }

#[test]
fn parses_version() {
    assert_eq!(parse("1.4"), Version { major: 1, minor: 4 });
}
```

Omit `Debug` and the compiler rejects the *test*, not the type:

> **⚠️ Pitfall →** `assert_eq!` on a type without `Debug` gives `error[E0277]: Version doesn't implement Debug` pointing at the macro call. The fix is `#[derive(Debug)]`. The reason the bound exists: the macro's failure path unconditionally formats both arguments with `{:?}`, and Rust resolves that requirement statically at the macro expansion site — there is no runtime "does this support printing" fallback as in Java's `Object.toString` or Python's `repr`. Likewise, comparing requires `PartialEq`; deriving it gives the obvious field-wise structural equality.

Any extra arguments after the required ones become a `format!` string for a custom failure message:

```rust
assert!(ratio <= 1.0, "ratio out of bounds: got {ratio}");
```

### Testing that code panics

For functions whose contract is "panic on bad input", assert the panic with `#[should_panic]`:

```rust
pub fn percentage(n: u8) -> u8 {
    assert!(n <= 100, "percentage must be 0..=100, got {n}");
    n
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "must be 0..=100")]
    fn rejects_over_100() {
        percentage(200);
    }
}
```

The test passes iff the body panics. The bare `#[should_panic]` is dangerously coarse — it would pass if the function panicked for *any* reason, including an unrelated bug. The `expected = "..."` argument tightens it: the panic message must *contain* that substring. Always supply it.

Two contracts are at work here, and they're worth separating. The type system already guarantees the *always-true* parts — `percentage` can only ever be handed a `u8`, so "it's a number, not a string" needs no test. What it can't express is "the number must be `0..=100`": that's a runtime promise the function enforces with a panic. `#[should_panic(expected = ...)]` tests exactly that promise — it checks the function rejects an out-of-range input at the right boundary. So the types cover what's provable at compile time, and `#[should_panic]` covers the runtime edge the types can't reach.

### Returning `Result` from tests

A test may instead return `Result<(), E>`. It passes on `Ok(())` and fails on `Err`. The payoff is that `?` now works inside the test body, so a chain of fallible operations short-circuits to a failure instead of needing `.unwrap()` at every step:

```rust
#[test]
fn round_trips() -> Result<(), Box<dyn std::error::Error>> {
    let parsed: i32 = "42".parse()?;   // ? bubbles a parse failure as test failure
    let back = parsed.to_string();
    assert_eq!(back, "42");
    Ok(())
}
```

One sharp edge: you cannot combine `#[should_panic]` with a `Result`-returning test. To assert a fallible call returns `Err`, do *not* use `?` on it — write `assert!(thing().is_err())`, otherwise the `?` turns the expected `Err` into a test failure. See [error handling](11-error-handling.md) for `?`, `Box<dyn Error>`, and the `Result` machinery this leans on.

> **🔧 In practice →** You're testing a round-trip: open a temp file, write some bytes, read them back, parse them, and check the result. Every one of those steps can fail, and in a `()`-returning test each would need its own `.unwrap()`, turning the body into noise. Returning `Result` lets you `?` straight through:
>
> ```rust
> #[test]
> fn config_round_trips() -> Result<(), Box<dyn std::error::Error>> {
>     let dir = std::env::temp_dir();
>     let path = dir.join("cfg_round_trips.toml");
>     std::fs::write(&path, "port = 8080\n")?;          // ? on io::Error
>     let text = std::fs::read_to_string(&path)?;       // ? on io::Error
>     let port: u16 = text.trim_start_matches("port = ").trim().parse()?; // ? on ParseIntError
>     assert_eq!(port, 8080);
>     Ok(())
> }
> ```
>
> Any failing step short-circuits and the runner reports it as a test failure with the actual error attached — which is far more useful than a generic "unwrap on a None/Err" backtrace. This is the everyday shape for integration tests that talk to the filesystem, a parser, or a network client.

## Running tests: `cargo test` mechanics

`cargo test` compiles in test mode and runs the resulting binary. Arguments before `--` go to Cargo; arguments after `--` go to the test runner. `cargo test --help` and `cargo test -- --help` document the two halves separately.

**Parallel by default.** Tests run concurrently on a thread pool. This is fast but it imposes a discipline: tests must not share mutable external state — no two tests writing the same file, mutating the same env var, or binding the same fixed port — or they race. The fix is either to make each test self-contained (unique temp paths, ephemeral ports) or to serialise:

```bash
cargo test -- --test-threads=1   # one at a time; deterministic, slower
```

> **🎓 Tripos link →** This is exactly the *Concurrent & Distributed Systems* shared-mutable-state hazard. The test runner gives you N threads with no implicit synchronisation; two tests touching `./fixture.db` is a data race at the filesystem level even though the Rust borrow checker, which only reasons about in-process memory, says nothing about it. The idiomatic resolution mirrors the course's advice: eliminate the sharing (give each test its own resource) rather than reach for a global lock. A global lock reintroduces serialisation and can deadlock if a test panics while holding it.

**Output capture.** By default the runner captures `stdout`/`stderr` of *passing* tests and shows them only for failures (so a green run is quiet). To see prints from passing tests too:

```bash
cargo test -- --show-output      # show captured stdout of passing tests
cargo test -- --nocapture        # don't capture at all (interleaves with parallel runs)
```

**Filtering by name.** A bare argument is a substring filter against the fully-qualified test name (module path included):

```bash
cargo test parse              # every test whose name contains "parse"
cargo test tests::round_trips # an exact path
cargo test --test wire        # only the integration-test file tests/wire.rs
```

Only the first positional filter is honoured; you cannot list several names, but a shared substring covers a group.

**Ignoring expensive tests.** Annotate slow tests so the default run skips them:

```rust
#[test]
#[ignore = "hits the network; run in CI only"]
fn fetches_remote() { /* ... */ }
```

Then `cargo test -- --ignored` runs *only* the ignored ones, and `cargo test -- --include-ignored` runs everything.

> **⚙️ Under the hood →** `cargo test` can produce *several* binaries: one for the library's/binary's unit tests, one per file in `tests/`, plus the doc-test harness. It runs them in sequence. Crucially, if any one stage fails, the later stages are not run — a failing unit test means you see no integration or doc-test output that run. Each test binary embeds `libtest` (the harness `#[test]` compiles against); the per-test thread spawning and the panic-as-failure protocol live there, which is why a panic *during* a test is caught while a `panic = "abort"` profile (see below) would abort the whole runner instead.

## Test organisation: three tiers

Rust's community splits tests into three kinds, distinguished by *where they live* and *what they can see*.

### 1. Unit tests — in-module, white-box

In the same file as the code, in a `#[cfg(test)] mod tests`. They can reach private functions and fields. Use them to check internal invariants and helpers:

```rust
fn normalise(path: &str) -> String { /* private */ todo!() }

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn collapses_dot_segments() {
        assert_eq!(normalise("a/./b"), "a/b"); // testing a non-pub fn — fine
    }
}
```

### 2. Integration tests — `tests/`, black-box

Files placed in a top-level `tests/` directory (sibling of `src/`). Cargo compiles *each file as its own separate crate* that links your library as an external dependency. Consequences:

- They see only your **public API** — exactly what a downstream user sees. They cannot touch privates.
- They need no `#[cfg(test)]`; Cargo only builds `tests/` for `cargo test`.
- They must `use your_crate::...` to bring items in, because they are a different crate.

```rust
// tests/api.rs
use mylib::checksum;   // only public items are reachable

#[test]
fn checksum_is_order_independent() {
    assert_eq!(checksum(&[1, 2]), checksum(&[2, 1]));
}
```

Shared helpers across integration files are a known wrinkle. A plain `tests/common.rs` would itself be compiled as a test crate and show up as an empty "running 0 tests" section. The fix is the directory form `tests/common/mod.rs`: files in *subdirectories* of `tests/` are not treated as test crates, so they become an ordinary module you pull in with `mod common;`.

> **⚠️ Pitfall →** Integration tests only work against a **library** crate. A pure binary (only `src/main.rs`, no `src/lib.rs`) exposes nothing — `use my_bin::...` fails with `error[E0432]: unresolved import`, because binaries are not linkable libraries. The idiom, which you will see in the [capstones](21-capstones.md), is to keep `main.rs` a thin shell that calls into `lib.rs`, then integration-test the library. The few lines left in `main` are not worth testing.

### 3. Doc tests — examples that must compile and run

This tier has no equivalent in your toolbox. Code fences inside `///` documentation comments are extracted, compiled, and *executed* by `cargo test`. Your examples cannot rot:

```rust
/// Returns the larger of two values.
///
/// # Examples
///
/// ```
/// use mylib::max;
/// assert_eq!(max(2, 9), 9);
/// ```
pub fn max(a: i32, b: i32) -> i32 {
    if a > b { a } else { b }
}
```

`cargo test` reports these under a `Doc-tests` section. If you rename `max`, or change its signature so the example stops compiling, or break its behaviour so the `assert_eq!` fails, the doc test fails — the example and the code are kept in lockstep by the build. This is the single biggest reason Rust documentation tends to have working examples.

Doc-comment mechanics worth knowing:

- `///` documents the *following* item; `//!` documents the *enclosing* item (use it at the top of `lib.rs` to describe the crate, or inside a module).
- Both accept Markdown. Conventional sections are `# Examples`, `# Panics`, `# Errors`, and `# Safety` (the last mandatory for `unsafe fn` — see [unsafe](16-unsafe-and-ffi.md)).
- A doc-test that should fail to compile is marked ```` ```compile_fail ````; one that shouldn't be run, ```` ```no_run ````; one whose boilerplate you want hidden from the rendered docs but still compiled is prefixed with `# ` inside the fence.

> **⚙️ Under the hood →** `rustdoc` lifts each fenced block into a standalone source file. Unless the block already has a `fn main`, rustdoc wraps it in one and inserts an implicit `extern crate yourcrate;`, then compiles and runs it as a tiny program. That is why the example must `use` your public items by their external path: it is genuinely a separate compilation unit, the same as an integration test. This also means doc tests do *not* see private items — they are black-box by construction.

> **🔧 In practice →** You ship a library with a function whose doc comment shows a usage example. Six months later you change the signature — say `parse(s: &str)` becomes `parse(s: &str, strict: bool)`. With any other language, the example in the docs silently goes stale and the next user copies code that no longer compiles. In Rust, `cargo test` runs that example as a real test, so the stale doc *fails the build*:
>
> ```rust
> /// Parses a port number from text.
> ///
> /// ```
> /// # use mylib::parse_port;
> /// assert_eq!(parse_port("8080"), Some(8080));
> /// ```
> pub fn parse_port(s: &str) -> Option<u16> {
>     s.parse().ok()
> }
> ```
>
> The `# ` prefix on the `use` line hides that boilerplate from the rendered HTML while still compiling it. The payoff is concrete: your README and API docs are guaranteed-correct example code, checked on every CI run, with no extra test file to maintain. This is why publishing a crate to crates.io almost always means writing doc tests first.

## The wider toolchain

Testing is one node in a graph of `cargo`/`rustup` tooling. The rest of this chapter is the checklist.

### `cargo check` vs `cargo build`

`cargo check` runs the front end — parsing, name resolution, type and borrow checking — and stops *before* codegen and linking. It produces no runnable artifact but emits every error and warning. Because LLVM codegen and linking dominate compile time, `check` is several times faster than `build`, and it is what your editor runs on every keystroke. Use `cargo check` in your edit loop; `cargo build` only when you actually need to run.

### `rustfmt` and `clippy`

```bash
cargo fmt              # rewrite all source to canonical style
cargo fmt --check      # fail (CI) if anything is unformatted
cargo clippy           # ~750 extra lints beyond rustc
cargo clippy -- -D warnings   # promote every lint to a hard error (CI gate)
```

`rustfmt` is opinionated and near-universally adopted; there is effectively one Rust style, which kills formatting debates dead. `clippy` is a second linter that runs as a compiler plugin and knows Rust idioms: it will tell you to replace `x.len() == 0` with `x.is_empty()`, collapse a manual loop into an iterator, drop a needless `.clone()`, or flag `if let Some(_) = x` where a `.is_some()` reads better. Treat clippy lints as **pedagogical** — early on, each one teaches you the idiom the community has converged on. In CI, gate on `cargo clippy -- -D warnings` so regressions cannot land.

> **🦀 From your toolbox →** `clippy` is `clang-tidy`/ESLint, and `rustfmt` is `clang-format`/Prettier — but both ship *with the toolchain* via `rustup component add clippy rustfmt` and share rustc's exact parser and type information, so clippy reasons over the real HIR rather than a re-implemented model of the language. That is why it catches semantic things a text-level linter can't, e.g. that a `match` is redundant given the types. The analogy breaks where clippy makes type-directed suggestions no external linter could: it sees trait impls, lifetimes, and monomorphised instances.

### `rust-analyzer`

The official Language Server (LSP). It gives editors completion, go-to-definition, inline type hints, on-the-fly diagnostics, and refactorings, by incrementally type-checking your crate graph in the background. Install it as an editor extension; it is the reason a modern Rust editing experience rivals an IDE for a managed language despite Rust's far heavier static analysis.

### `cargo doc` / `rustdoc`

```bash
cargo doc --open                 # build HTML docs for your crate + deps, open in browser
cargo doc --no-deps              # just your crate
```

`rustdoc` turns your `///`/`//!` comments and item signatures into a browsable site (`target/doc/`). Because it understands the type system, every type, trait bound, and link between items is cross-referenced automatically. The same engine runs your doc tests, so "the docs build" and "the examples pass" are one check.

### Release profiles

A *profile* is a named bundle of compiler settings. Cargo has `dev` (used by `cargo build`/`test`) and `release` (`cargo build --release`); you can also define `[profile.bench]` and custom profiles. Override defaults in `Cargo.toml`:

```toml
[profile.dev]
opt-level = 0       # fast compile, no optimisation (the dev default)

[profile.release]
opt-level = 3       # full optimisation (the release default)
lto = "thin"        # link-time optimisation across crate boundaries
codegen-units = 1   # one codegen unit => more inlining, slower compile, faster binary
panic = "abort"     # skip unwinding tables; abort on panic (smaller, can't catch_unwind)
strip = "symbols"   # drop symbols from the binary
```

The dials that matter:

- **`opt-level`** — `0..=3`, plus `"s"`/`"z"` for size. `dev` defaults to `0` (compile fast, run slow); `release` to `3`.
- **`lto`** (link-time optimisation) — `false` / `"thin"` / `true` (fat). Lets the optimiser inline and specialise *across* crate boundaries, which monomorphisation otherwise can't reach once a dependency is compiled separately. Slower link, faster binary.
- **`codegen-units`** — rustc splits a crate into N units compiled in parallel; fewer units means better cross-function optimisation but less parallelism. `release` defaults to `16`; setting `1` squeezes out more performance at the cost of compile time.
- **`panic = "abort"`** — replaces stack-unwinding with immediate `abort`, removing landing-pad code and unwind tables. Smaller and slightly faster, but `std::panic::catch_unwind` no longer works and `#[should_panic]`/the test harness rely on unwinding, so tests force unwinding regardless.

> **🎓 Tripos link →** *Compiler Construction*: `codegen-units` and `lto` are knobs on exactly the inlining/specialisation trade-off you studied. Monomorphisation (covered in [generics & traits](08-generics-and-traits.md)) generates a concrete copy per type instantiation; with separate compilation, the copies in a dependency crate are opaque to your crate's optimiser. `lto = "thin"` re-opens those boundaries at link time so the inliner can act across them — the price is that more code is in scope for the optimiser, hence the slower link.

### Features and conditional compilation

`#[cfg(...)]` compiles code conditionally on a flag, target attribute, or *feature*. Features are named, optional capabilities declared in `Cargo.toml` that let downstream users opt into functionality (and the dependencies it pulls in):

```toml
[features]
default = ["serde"]
serde = ["dep:serde"]          # enabling `serde` feature pulls in the serde crate

[dependencies]
serde = { version = "1", optional = true }
```

```rust
#[cfg(feature = "serde")]
mod serialisation;   // compiled only when the `serde` feature is on

#[cfg(target_os = "linux")]
fn epoll_setup() { /* Linux-only */ }

#[cfg(test)]
mod tests { /* the test-only case you already met */ }
```

`#[cfg(...)]` is evaluated at compile time and the disabled code is *not even parsed for type-checking against the rest of the program* — it is excised before the borrow checker runs, unlike a runtime `if cfg!(...)` which compiles all branches.

### Publishing to crates.io and SemVer

`crates.io` is the central registry. To publish:

1. `cargo login <token>` — store your API token (from your crates.io account, GitHub-linked) in `~/.cargo/credentials.toml`. Treat the token as a secret.
2. Fill required `[package]` metadata. `cargo publish` *errors* without `description` and `license` (an SPDX identifier like `"MIT OR Apache-2.0"` — the community's conventional dual licence — or `license-file`).
3. `cargo publish` — packages the source, verifies it builds, and uploads.

Two rules that bite if ignored:

- **A publish is permanent.** A given `name@version` can never be overwritten or deleted (the registry must stay a reproducible archive for everyone's `Cargo.lock`). You bump the version and publish again; you never re-upload the same version.
- **To retract a broken release, `cargo yank --version 1.2.3`.** Yanking prevents *new* dependency resolutions from selecting that version while leaving existing `Cargo.lock` files that pin it untouched. `--undo` reverses it. Yanking does **not** delete code, so it cannot scrub a leaked secret — if you publish a key, rotate it immediately.

SemVer is a contract about API compatibility, and the test for each level is just "would existing user code still compile and behave the same?" `MAJOR.MINOR.PATCH`: bump PATCH for a backward-compatible bug fix (same API, fixed behaviour), MINOR for a backward-compatible addition (a new `pub` item — old code can't break because it never named the new thing), MAJOR for a breaking change (removing or changing the signature of something callers already use — their code stops compiling). Cargo's resolver leans on this promise: it freely upgrades a dependency within a compatible range without asking you. `cargo semver-checks` can mechanically compare two versions of your crate and tell you whether you've broken the contract before you publish.

To export a flat, convenient public API regardless of your internal module tree, re-export with `pub use`:

```rust
// users write `mycrate::Engine`, not `mycrate::core::internal::Engine`
pub use crate::core::internal::Engine;
```

See [project structure](19-project-structure.md) for the module and crate mechanics underneath this.

### `cargo install` and custom subcommands

`cargo install <crate>` builds a binary crate from crates.io (or `--git`, or `--path`) and drops the executable into `~/.cargo/bin` (ensure it's on `$PATH`). This is for developer tools, not a system package manager — e.g. `cargo install ripgrep` gives you `rg`.

Cargo is extensible with zero plumbing: any executable on `$PATH` named `cargo-foo` is invocable as `cargo foo`, and shows up in `cargo --list`. This is how the ecosystem ships subcommands. Worth installing:

- **`cargo-nextest`** — a faster, better-isolated test runner. It runs each test in its own *process* (not just thread), so a segfault or `abort` in one test doesn't take the harness down with it, and it gives much nicer output and per-test timing. Drop-in: `cargo nextest run`.
- **`cargo-expand`** — prints source with all macros expanded. Indispensable for understanding what a `#[derive]` or your own `macro_rules!`/proc-macro actually generates (see [macros](18-macros.md)).
- **`cargo-flamegraph`** — `cargo flamegraph` profiles a run and produces an interactive flamegraph SVG; the first reach for "where is the time going".
- **`criterion`** — a statistics-driven benchmarking *library* (stable Rust's built-in `#[bench]` is nightly-only). It runs your benchmark many times, controls for noise, and reports confidence intervals and regressions against the previous run.

## Mental-model recap

- A test is a `#[test]` function that *passes by returning and fails by panicking*; every assertion macro is a conditional `panic!`. The runner is built into `cargo`, not a framework you add.
- `#[cfg(test)]` excises unit tests from shipped artifacts; the equality macros statically require `PartialEq` + `Debug` because they format both operands on failure.
- Three tiers, distinguished by location and visibility: **unit** (in-module, sees privates), **integration** (`tests/`, separate crate, public API only, library crates only), **doc tests** (examples in `///`, compiled and run so docs can't rot).
- Tests run in parallel by default — shared external state (files, ports, env) races; isolate it or `--test-threads=1`. `cargo check` for the fast edit loop; `cargo clippy -- -D warnings` and `cargo fmt --check` as CI gates.
- Profiles (`opt-level`, `lto`, `codegen-units`, `panic`) are compile/runtime trade-off dials; publishing is permanent and SemVer-governed, with `yank` (not delete) to retract.

## Exercises

1. **Predict the error.** A teammate writes `assert_eq!(left, right)` where the type derives `Debug` but *not* `PartialEq`. Before compiling, predict which trait the error names and roughly what it says — and crucially, will the error point at the *type definition* or at the *macro call site*? Then flip it (derives `PartialEq` but not `Debug`) and predict that error too. Explain in one sentence why a single macro can demand two different traits.

2. **Which of these compiles, and why?** Given a fallible helper `fn load() -> Result<Config, std::io::Error>`, decide for each test whether it compiles and passes, fails to compile, or compiles but fails at runtime:
   (a) `#[test] fn t() -> Result<(), std::io::Error> { let c = load()?; assert!(c.is_valid()); Ok(()) }`
   (b) `#[test] #[should_panic] fn t() -> Result<(), std::io::Error> { load()?; Ok(()) }`
   (c) `#[test] fn t() { assert!(load().is_err()); }` when `load()` actually returns `Ok`.
   Explain each verdict.

3. **Design decision: where does this test live?** You're testing a `normalise_path` helper that is *not* `pub`, plus a `Client::connect` method that *is* `pub` and which you want to exercise exactly the way a downstream user would. For each, decide: unit test (in-module `#[cfg(test)]`), integration test (`tests/`), or doc test — and justify the choice in terms of what each tier can see. Then say what would go wrong if you tried to integration-test the private helper.

4. **Adapt to avoid a race.** Two tests both run `cargo test` happily in isolation but fail intermittently together; both call `std::env::set_var("MODE", ...)` and read it back. Adding `--test-threads=1` makes them pass. Explain *why* that fix works but is the wrong long-term answer, then redesign the tests so they pass under the default parallel runner *without* touching a shared env var. (Hint: pass the mode in as an argument instead of reading global state.)

5. **(★) When would you choose A over B?** You maintain a library crate and need both fast iteration and a lean, fast shipping binary. Decide which of these belong in `[profile.dev]` versus `[profile.release]`, and which you would *not* set at all: `opt-level = 0`, `lto = "thin"`, `codegen-units = 1`, `panic = "abort"`. For `panic = "abort"`, predict what happens when you then run `cargo test` — does the abort setting take effect for the test binary? Explain why the harness behaves as it does.
