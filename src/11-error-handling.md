# 11. Error Handling

Most languages you know have a single error-handling channel that is *invisible in the type signature*. In Java, `String readConfig()` can throw `IOException` and you only learn that from the `throws` clause (checked) or, worse, from the documentation or a stack trace at 3am (unchecked `RuntimeException`). In Swift, a function annotated `throws` tells you it *can* fail but not *how* — the error type is collapsed to a single `Error`, so you discover the concrete cases at the `catch`. In Python, *any* call can raise *any* exception and nothing in the signature warns you. C++ is similar: `std::string read_config()` can throw anything, and the signature says nothing — `noexcept` is opt-in and rarely used.

Rust makes a hard, deliberate split that maps onto a distinction every other language blurs:

- **Bugs** — violated invariants, "this can't happen" situations, programmer errors. These use `panic!`, are *unrecoverable* by default, and are deliberately **not** in the type signature, because no caller should be writing recovery logic for your bug.
- **Expected failure** — a file that might not exist, input that might be malformed, a network that might be down. These are values of type `Result<T, E>`, **in the signature**, and the compiler forces every caller to confront them.

There are no exceptions in Rust. There is no `throw`, no stack-unwinding control flow you can `catch` (panic unwinding exists but is not a general control-flow tool — see below). The closest thing you already know is Swift's `Result<Success, Failure>` and Java's `Optional` / `Either`-style result objects — but where Swift and Java still let you `throw` *and* return a result, Rust has only the one channel. A function that can fail says so in its return type, and the compiler enforces that you act on it. The `?` operator (below) makes that ergonomic enough to use everywhere.

> **🦀 From your toolbox →** `Result<T, E>` is Swift's `Result<Success, Failure>` enum almost exactly — two cases, `Ok`/`Err` versus `.success`/`.failure`, generic in both payloads — and it pattern-matches the same way. The big difference from Swift: in Swift you usually `throws` and reach for `Result` only at API boundaries; in Rust `Result` *is* the boundary, every time, with no `throws` alternative. Compared to Java's checked exceptions, a Rust function lists its failure in the *return* type rather than a separate `throws` clause, and the error is a first-class value you can store, map, and pass around — not a control-flow jump. Where the analogy breaks: Java/Swift can still throw past the type system (unchecked exceptions, `try!`), so the signature is advisory; Rust's `Result` is the only road, so the signature is a guarantee.

## Track one: `panic!` for unrecoverable bugs

`panic!` aborts the current thread's normal execution. You trigger it explicitly or — far more commonly — the standard library triggers it on your behalf when you violate a contract:

```rust
fn main() {
    let xs = vec![10, 20, 30];
    let _ = xs[99]; // panics: index out of bounds: the len is 3 but the index is 99
}
```

In Java this throws `ArrayIndexOutOfBoundsException`; in Swift it traps; in Python `xs[99]` raises `IndexError`. In C, the same access on a heap array is undefined behaviour: a buffer overread that hands you whatever bytes live past the allocation, a textbook security hole. Rust's `Index` impl for slices bounds-checks and panics instead — closer to the Java/Swift/Python behaviour than the C one. That is the whole philosophy in one line: **a contract violation is a bug, and continuing with corrupt assumptions is worse than stopping.**

A panic prints the message and source location, then by default *unwinds*:

```text
thread 'main' panicked at src/main.rs:3:13:
index out of bounds: the len is 3 but the index is 99
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Set `RUST_BACKTRACE=1` to get the call stack. Read it top-down and stop at the first frame in a file you wrote — that is where your bug lives; frames above are stdlib machinery (`core::panicking::panic_bounds_check` and friends). This is the same read-the-stack-trace skill you use with a Java exception or a Python traceback.

### Unwinding vs. aborting

> **⚙️ Under the hood →** A panic does one of two things, chosen at compile time. The default is **unwind**: rustc walks back up the stack frame by frame, running every `Drop` impl it passes — exactly like the destructor cleanup that runs when a value goes out of scope (it uses the same platform mechanism, the Itanium/SEH unwinder, and a landing pad per frame). This frees resources but bloats the binary with unwind tables. The alternative is **abort**: call `std::process::abort()` immediately, run no destructors, let the OS reclaim memory. You opt in per profile:
> ```toml
> [profile.release]
> panic = "abort"
> ```
> `abort` produces smaller binaries and is mandatory in some contexts (e.g. a panic crossing an `extern "C"` boundary is UB unless aborting — see [unsafe and FFI](16-unsafe-and-ffi.md)). With unwinding, a panic in a spawned thread kills only that thread; the parent observes it via the `JoinHandle` ([fearless concurrency](13-fearless-concurrency.md)).

`unwrap` and `expect` (below) are the most common ways `panic!` shows up in real code. The rule for when a panic is *correct* is the subject of the final section.

## Track two: `Result<T, E>` for recoverable failure

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

This is an enum with two cases that each carry a payload — directly comparable to a Swift enum with associated values, generic in both the success payload `T` and the error payload `E` ([generics and traits](08-generics-and-traits.md)). It is in the prelude, so `Ok`/`Err` are in scope unqualified, exactly like `Some`/`None`. A fallible function returns it, and the *only* way to get at the `T` is to deal with the possibility of the `E`:

```rust
use std::fs::File;

fn main() {
    let handle = match File::open("config.toml") {
        Ok(f) => f,
        Err(e) => panic!("could not open config: {e:?}"),
    };
    // use `handle`...
}
```

`File::open` returns `Result<File, std::io::Error>`. There is no way to forget the error: the value you have is a `Result`, not a `File`, and the type system will not let you call `File` methods on it. You must pattern-match (or use one of the combinators) to extract the `File` — just as a Swift `Result` forces you through a `switch` or a `try get()` before you touch the success value. This is the entire safety argument — **failure is encoded in the type, so the compiler can prove you handled it.**

`Result` is annotated `#[must_use]`. Ignore one and you get a warning, not silence:

```rust
fn main() {
    std::fs::remove_file("scratch.tmp"); // warning: unused `Result` that must be used
}
```

> **⚠️ Pitfall →** That warning catches a whole bug class directly: silently dropping a failure you never inspected — the C habit of writing `fclose(f);` and never checking the return, or the Python habit of an exception you swallow in a bare `except: pass`. Here, `let _ = std::fs::remove_file("scratch.tmp");` is the explicit "yes, I am deliberately discarding this" — but if you are discarding an I/O error you almost always have a bug. The idiomatic fix is to propagate it with `?`.

### Matching on the *kind* of error

`std::io::Error` carries a `kind()` you can branch on, so you can recover from "file not found" while still treating "permission denied" as fatal:

```rust
use std::fs::File;
use std::io::ErrorKind;

fn open_or_create(path: &str) -> std::io::Result<File> {
    match File::open(path) {
        Ok(f) => Ok(f),
        Err(e) if e.kind() == ErrorKind::NotFound => File::create(path),
        Err(e) => Err(e),
    }
}
```

(`std::io::Result<T>` is a type alias for `Result<T, std::io::Error>` — a common pattern for crates with one dominant error type.) Nested `match` gets verbose fast; the combinator methods (`map`, `and_then`, `unwrap_or_else`, `ok`) plus closures ([closures and iterators](12-closures-and-iterators.md)) usually read better. But the real ergonomic win is `?`.

## The `?` operator: propagation as syntax

Manually threading errors up the call stack is tedious. This:

```rust
fn read_to_string(path: &str) -> Result<String, std::io::Error> {
    let mut f = match File::open(path) {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => return Err(e),
    }
}
```

collapses to this:

```rust
use std::io::Read;

fn read_to_string(path: &str) -> Result<String, std::io::Error> {
    let mut f = std::fs::File::open(path)?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

The `?` after a `Result` means: if `Ok(v)`, evaluate to `v` and continue; if `Err(e)`, **return early** from the enclosing function with `Err(e)`. The whole expression `e1?` is shorthand for `match e1 { Ok(x) => x, Err(e) => return Err(e) }`. If you have used Swift's `try` keyword, the feel is similar — "do this, and if it fails, bail out of the current function with the error" — except Rust's `?` returns the error as a value rather than throwing it. Method chaining works because `?` is an expression that produces the unwrapped value: `File::open(path)?.read_to_string(&mut s)?`.

> **🔧 In practice →** You are writing a function in a web request handler that loads a user's profile: read a file, parse it as JSON, look up a field. Each step can fail, and you want the handler to return an error response rather than crash. `?` lets you write the happy path straight down the page and let any failure short-circuit out to the caller:
> ```rust
> fn load_profile(path: &str) -> Result<Profile, std::io::Error> {
>     let raw = std::fs::read_to_string(path)?;   // bail on read failure
>     let mut profile = Profile::default();
>     profile.name = raw.lines().next()
>         .unwrap_or("anonymous").to_string();
>     Ok(profile)
> }
> ```
> Without `?` this is a staircase of `match`es that buries the one line that actually does work. With it, the error path is invisible until it fires — which is exactly when you want to think about it.

### `?` is not *just* early-return — it calls `From::from`

The one detail that makes `?` compose across error types: on the `Err` path, the error is passed through `From::from` before being returned. So `e1?` actually desugars (roughly) to:

```rust
match e1 {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
}
```

If function `f` returns `Result<T, MyError>` and calls something returning `Result<_, io::Error>`, then `something()?` works **iff** `MyError: From<io::Error>`. The `?` converts the inner error into your function's declared error type automatically. This is the mechanism that lets one function aggregate failures from many sources behind a single error type — and it is why "implement `From` for your error type" is the central move in custom-error design below.

> **⚙️ Under the hood →** `?` is sugar over the `Try` trait (`std::ops::Try`) and `FromResidual`. For `Result<T, E>` the "residual" is `Result<Infallible, E>`, and `FromResidual` is where `From::from` on the error gets invoked. You will not implement `Try` yourself on stable, but knowing it is a trait explains why `?` works uniformly on `Result`, `Option`, and `ControlFlow`, and why the compiler error mentions `FromResidual`.

### Where `?` is allowed

`?` performs an early return, so the enclosing function's return type must be able to hold the residual. Use `?` on a `Result` in a function returning `()` and:

```text
error[E0277]: the `?` operator can only be used in a function that
returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:4:48
  |
3 | fn main() {
  | --------- this function should return `Result` or `Option` to accept `?`
4 |     let f = File::open("hello.txt")?;
  |                                    ^ cannot use the `?` operator in a
  |                                      function that returns `()`
```

> **⚠️ Pitfall →** This is the most common `?` error. The fix the compiler suggests — change the return type — is usually right: make the function return `Result<_, _>`. If you genuinely cannot (e.g. a trait method with a fixed signature), handle the `Result` with `match`, `if let`, or a combinator instead. You **cannot** `?` a `Result` in an `Option`-returning function or vice versa: `?` will not silently convert between the two enums. Bridge them explicitly with `.ok()` (`Result` → `Option`, discarding the error) or `.ok_or(err)` / `.ok_or_else(|| err)` (`Option` → `Result`).

### `?` in `main` and in `Option`-returning functions

`main` may return `Result<(), E>` for any `E: Debug`. The runtime prints the `Err` via `Debug` and exits with a non-zero code — mirroring the convention you know from Java's `System.exit` codes and C's `main` returning `0` on success, non-zero on failure (Rust implements the `std::process::Termination` trait to do this):

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let _config = std::fs::read_to_string("config.toml")?;
    Ok(())
}
```

`Box<dyn Error>` is a trait object ([trait objects and OOP](09-trait-objects-and-oop.md)) meaning "any type implementing the `Error` trait, heap-allocated behind a pointer." It is close in spirit to Swift's `any Error` or Java's `catch (Exception e)` — a single type that any concrete error fits into. It is the universal sink for `?`: because `Box<dyn Error>: From<E>` for every `E: Error`, any error converts into it. That is why it is the idiomatic error type for `main` and for quick application code.

`?` works on `Option` too, with `None` as the short-circuit value:

```rust
fn first_word_initial(text: &str) -> Option<char> {
    text.split_whitespace().next()?.chars().next()
}
```

If `next()` yields `None`, the function returns `None` immediately; otherwise `?` unwraps the `&str` and we keep going. Same mechanism, "fail = bail out now," applied to absence instead of error — the way Swift's optional chaining `a?.b?.c` quietly returns `nil` the moment any link is `nil`.

## `unwrap` and `expect`: deliberate panics

`unwrap` and `expect` collapse a `Result`/`Option` to its inner value, panicking on the failure variant. They are the bridge from track two back to track one — "I assert this cannot fail; if I'm wrong, that's a bug, so crash."

```rust
let port: u16 = "8080".parse().unwrap();          // panics on the Err with a generic message
let cfg = std::fs::read_to_string("config.toml")
    .expect("config.toml must exist; it ships with the binary");
```

`expect` takes a message and panics *with* it; `unwrap` uses a generic one. **Prefer `expect` always.** The string is not a user-facing error message — it documents the *invariant you are asserting*, so that if the panic fires, the backtrace tells the next maintainer what assumption was wrong. "config.toml must exist" is a far better failure than "called `Result::unwrap()` on an `Err` value".

> **🦀 From your toolbox →** `.unwrap()` on `Option` is Swift's force-unwrap `!` on an optional (`maybe!`), which traps on `nil`, and Java's `Optional.get()`, which throws `NoSuchElementException` on an empty optional. `.expect(msg)` is the same force-unwrap but with a custom message attached — Swift's `precondition`/`fatalError(msg)` is the closest feel. The key cultural difference: Swift and Java code force-unwrap or call `.get()` fairly freely; in idiomatic Rust an `unwrap`/`expect` in library code is a claim that *demands justification in the message or a comment*, because it converts a recoverable error into a crash on behalf of every caller.

When is `unwrap`/`expect` acceptable?
- **Examples, prototypes, tests.** A failed `unwrap` in a `#[test]` panics, which is exactly how a test reports failure (the same way an uncaught exception fails a JUnit test). Robust error handling would only obscure the example. (More in [testing and tooling](20-testing-and-tooling.md).)
- **When you can prove the `Err` is impossible** but the compiler cannot:
  ```rust
  use std::net::IpAddr;
  let loopback: IpAddr = "127.0.0.1".parse()
      .expect("hardcoded loopback address is always valid");
  ```
  `parse` returns a `Result` because *in general* a string may be malformed; here the literal is fixed and valid, so asserting it is fine — and the message records *why* it is safe, prompting a future reader to revisit if the input ever becomes dynamic.

## Custom error types and the `Error` trait

Returning `io::Error` everywhere works only while every failure really is an I/O error. A realistic function fails in several distinct ways (parse error, validation error, I/O error), and you want one type that names them all. Enums are the natural fit — one case per failure mode, just like a Swift `enum Error: Swift.Error` with associated values for each case:

```rust
use std::fmt;

#[derive(Debug)]
enum ConfigError {
    Io(std::io::Error),
    Parse { line: usize, msg: String },
    MissingKey(String),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Io(e) => write!(f, "I/O error: {e}"),
            ConfigError::Parse { line, msg } => write!(f, "parse error at line {line}: {msg}"),
            ConfigError::MissingKey(k) => write!(f, "missing required key `{k}`"),
        }
    }
}

impl std::error::Error for ConfigError {
    // Override `source` to expose the underlying cause for error-chain printing:
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::Io(e) => Some(e),
            _ => None,
        }
    }
}
```

The `std::error::Error` trait is the standard interface for "this is an error" — the role Java's `Throwable`/`Exception` and Swift's `Error` protocol play. Its contract: you must also implement `Debug` (for programmers) and `Display` (for users), and you may override `source()` to return the underlying cause, which lets tools print a chain ("parse error … caused by … I/O error … caused by …") — the same idea as Java's exception `getCause()` chain. Everything that takes `Box<dyn Error>` relies on this trait.

To make `?` compose, implement `From` for each wrapped error type:

```rust
impl From<std::io::Error> for ConfigError {
    fn from(e: std::io::Error) -> Self {
        ConfigError::Io(e)
    }
}
```

Now `?` on any `io::Error`-returning call inside a `Result<_, ConfigError>` function auto-converts via `From::from`:

```rust
fn load(path: &str) -> Result<String, ConfigError> {
    let text = std::fs::read_to_string(path)?; // io::Error --From--> ConfigError, automatically
    if text.is_empty() {
        return Err(ConfigError::MissingKey("root".into()));
    }
    Ok(text)
}
```

> **🔧 In practice →** This is the pattern you reach for the moment a function talks to more than one subsystem. Imagine a config loader that reads a file (`io::Error`) and parses an integer out of it (`ParseIntError`). You do not want callers juggling two unrelated error types, so you define one `ConfigError` enum with a variant for each, add a `From` impl per source, and then your function body stays clean:
> ```rust
> fn read_port(path: &str) -> Result<u16, ConfigError> {
>     let text = std::fs::read_to_string(path)?; // io::Error  -> ConfigError::Io
>     let port = text.trim().parse()?;           // ParseIntError -> ConfigError::Parse...
>     Ok(port)
> }
> ```
> Each `?` quietly converts a different upstream error into your one declared type. Callers match on `ConfigError` and never see the plumbing. (For the `parse()?` line to compile you would add `impl From<std::num::ParseIntError> for ConfigError` mapping it into a suitable variant.)

> **⚙️ Under the hood →** `Box<dyn Error>` is a fat pointer — data pointer plus vtable pointer — exactly the trait-object layout from [trait objects and OOP](09-trait-objects-and-oop.md). It heap-allocates and dynamically dispatches `Display`/`Debug`/`source`. A concrete enum like `ConfigError` is stack-sized (the largest variant plus a tag), monomorphised, statically dispatched, and exhaustively matchable by callers. The trade-off: `Box<dyn Error>` is maximally flexible and ergonomic but erases the type, so callers cannot match on *which* error; a concrete enum is recoverable-by-variant but couples callers to your error taxonomy. Libraries lean concrete; applications lean boxed.

## The ecosystem reality: `thiserror` and `anyhow`

Writing `Display` + `Error` + a dozen `From` impls by hand is exactly the boilerplate the two dominant error crates eliminate. The community split is sharp and worth internalising:

- **`thiserror`** — for *library* errors, where callers need to match on variants. It is a `derive` macro ([macros](18-macros.md)) that generates the `Display`, `Error`, and `From` impls from attributes. You still get a concrete, matchable enum — just without the hand-written boilerplate:
  ```rust
  use thiserror::Error;

  #[derive(Debug, Error)]
  pub enum ConfigError {
      #[error("I/O error")]
      Io(#[from] std::io::Error),          // generates From<io::Error> + source()
      #[error("parse error at line {line}: {msg}")]
      Parse { line: usize, msg: String },
      #[error("missing required key `{0}`")]
      MissingKey(String),
  }
  ```
  `#[error("...")]` writes `Display`; `#[from]` writes both the `From` conversion and `source()`. The result is byte-for-byte the type you'd write by hand. Crate consumers can still `match` on `ConfigError::Parse { .. }`.

- **`anyhow`** — for *application* code (binaries, top-level glue), where you just want errors to propagate and print with context, and no caller will ever match on the variant. `anyhow::Error` is a smarter `Box<dyn Error>`: it captures a backtrace, lets you attach context, and converts from any `Error` type, so `?` "just works" everywhere:
  ```rust
  use anyhow::{Context, Result}; // anyhow::Result<T> = Result<T, anyhow::Error>

  fn run() -> Result<()> {
      let text = std::fs::read_to_string("config.toml")
          .context("reading config.toml")?; // adds a human-readable layer
      let parsed: u32 = text.trim().parse()
          .context("config.toml must contain a single integer")?;
      println!("loaded: {parsed}");
      Ok(())
  }
  ```
  `.context(...)` wraps the underlying error with a message, building the cause chain that `anyhow` prints automatically. Different error types (`io::Error`, `ParseIntError`) flow through the same `?` with zero `From` impls because `anyhow::Error: From<E>` for all `E: Error + Send + Sync + 'static`.

The decision rule: **`thiserror` when your callers are code that needs to react to specific failures (a library); `anyhow` when the caller is a human reading a log (an application).** A library should not force `anyhow` on its consumers — that erases the variants they might want to handle. A binary using `thiserror` everywhere is needless ceremony. Many real crates use both: `thiserror` for the public API surface, `anyhow` in internal binaries and tests.

(These are external crates; per your project rules, fetch current API details via Context7 before relying on specific attributes, as their macros evolve.)

## `Option` ↔ `Result` conversions

The two short-circuiting types convert into each other through a small, memorisable set of methods:

| From | To | Method | Semantics |
|------|----|--------|-----------|
| `Result<T, E>` | `Option<T>` | `.ok()` | `Ok(v)→Some(v)`, `Err(_)→None` (**discards** the error) |
| `Result<T, E>` | `Option<E>` | `.err()` | `Ok(_)→None`, `Err(e)→Some(e)` |
| `Option<T>` | `Result<T, E>` | `.ok_or(e)` | `Some(v)→Ok(v)`, `None→Err(e)` |
| `Option<T>` | `Result<T, E>` | `.ok_or_else(\|\| e)` | same, but `e` computed lazily only on `None` |

Use `.ok_or_else` over `.ok_or` whenever constructing the error is non-trivial (allocates, formats) — `ok_or` evaluates its argument eagerly even on the `Some` path, the same eager-vs-lazy distinction as `unwrap_or` vs `unwrap_or_else`. These conversions are how you bridge an `Option`-producing call (a map lookup, `.first()`) into a `Result`-returning function so `?` applies.

> **🔧 In practice →** You are parsing a config map and a key is *optional in the data structure* but *required for this code path*. `HashMap::get` hands you an `Option`, but your function returns a `Result` so `?` can propagate. `.ok_or_else` is the join:
> ```rust
> fn host(cfg: &std::collections::HashMap<String, String>) -> Result<String, ConfigError> {
>     let raw = cfg.get("host")
>         .ok_or_else(|| ConfigError::MissingKey("host".into()))?; // None -> a real error
>     Ok(raw.trim().to_string())
> }
> ```
> The `.ok_or_else(...)` turns "absent" into a named failure your callers can react to, and the trailing `?` lets it flow out like any other error. Use the `_else` form here because building the `ConfigError` allocates a `String` you do not want to pay for on the common `Some` path.

## To panic or not to panic

The guiding principle: **returning `Result` is the default for any function that can fail, because it hands the decision to the caller.** Panicking decides *for* the caller that the situation is unrecoverable, which is presumptuous unless you are certain.

Panic when the program reaches a **bad state** — a broken invariant, contract, or guarantee — *and*:
- the bad state is unexpected (not routine, like malformed user input), and
- your subsequent code relies on not being in that state rather than re-checking, and
- there is no good way to encode the constraint in the types.

Return `Result` when failure is an **expected, routine possibility**: a parser meeting malformed input, an HTTP call hitting a rate limit, a file that may be absent. These are not bugs; they are the normal operating envelope, and the caller must choose how to respond.

A function's **contract** is the special case that justifies panic in library code. If your function's behaviour is only defined for certain inputs (a precondition), violating it is always a caller-side bug, and panicking is the right response — there is nothing for the caller to "handle," they must fix their code. This is exactly why slice indexing panics on out-of-bounds: operating on memory outside the structure is a security hole, so the stdlib refuses. Document any such panic in the function's API docs.

> **💡 Make the bad state unrepresentable →** Often the best move is not to choose between panic and `Result` at all, but to design the bad state out of existence. Encode the invariant in a type, and the compiler enforces it once, at construction, so no later code has to re-check. A newtype with a private field and a validating constructor turns a runtime contract into a compile-time guarantee:
> ```rust
> pub struct Percent(u8); // invariant: 0..=100, enforced at construction
>
> impl Percent {
>     pub fn new(v: u8) -> Result<Percent, String> {
>         if v <= 100 { Ok(Percent(v)) } else { Err(format!("{v} > 100")) }
>     }
>     pub fn get(self) -> u8 { self.0 }
> }
> ```
> Every function taking `Percent` (not `u8`) is now *guaranteed* to receive a valid percentage — no defensive check, no panic, no `Result` at the use site. The private field means the only way to make a `Percent` is through `new`, so the invariant holds by construction. (This is the same instinct as making a class field private with an invariant-checking constructor in Java or Swift — except here the type system, not convention, makes it the only path in.) Choosing `u32` over `i32` to forbid negatives, or a concrete type over `Option` to forbid absence, are the same move at smaller scale.

## Mental-model recap

- **Two channels, by design:** `panic!` for bugs/violated invariants (unrecoverable, *not* in the signature), `Result<T, E>` for expected failure (recoverable, *in* the signature). Rust has no catchable exceptions; failure that callers should handle is always a value in the type.
- **`?` is short-circuit-on-failure made into syntax:** it early-returns on `Err`/`None` and, crucially, runs `From::from` on the error — so implementing `From<InnerError> for YourError` is what lets `?` aggregate many failure sources behind one type.
- **`unwrap`/`expect` convert a `Result` back into a panic** — only justified in tests/prototypes or where you can prove the `Err` is impossible. Always `expect` with a message stating the invariant, never bare `unwrap` in real code.
- **Concrete enum (or `thiserror`) for library errors** so callers can match on variants; **`Box<dyn Error>` or `anyhow` for application errors** where the consumer is a human reading a log. `thiserror` derives the boilerplate; `anyhow` erases the type for maximum `?` ergonomics.
- **Prefer making bad states unrepresentable** (newtypes, validating constructors, non-`Option` types) over runtime checks. A constraint enforced by the type at construction is one you never check, never panic on, and never return a `Result` for at the use site.

## Exercises

1. **Predict the error, then fix it two ways.** Sketch `fn parse_pair(s: &str) -> Result<(i32, i32), std::num::ParseIntError>` that splits `s` on a comma and parses both halves with `?`. Now suppose you also want to fail cleanly when there is *no* comma. `s.split_once(',')` returns an `Option`, so adding `?` to it will not compile against a `ParseIntError` return type — predict the exact reason before you try it. Then make it work two ways: (a) bridge the `Option` with `.ok_or_else(...)` into a custom error enum, and (b) widen the return type to `Box<dyn Error>`. Which call site reads better for a *library* consumer, and which for a *binary*?

2. **Which of these compiles, and why?** Given `fn f() -> Result<i32, MyError>` where `MyError` implements `From<std::io::Error>` but *not* `From<std::num::ParseIntError>`, decide for each line whether it compiles inside `f`'s body and explain in one sentence why: (a) `let n = std::fs::read_to_string("x")?;` (b) `let n: i32 = "42".parse()?;` (c) `let n: i32 = "42".parse().map_err(MyError::from)?;`. Then say in one sentence why `?` is specified in terms of `From` rather than `Into`.

3. **Design decision.** You are writing a date-parsing library used by other crates, and separately a small CLI that loads one config file at startup. For each, decide whether you would reach for a hand-written enum, `thiserror`, `anyhow`, or `Box<dyn Error>`, and justify the choice in one sentence each. Then name one concrete thing a *caller* of the library could do that they could not do if you had returned `anyhow::Error` from the library.

4. (★) **Adapt `collect`.** Iterators of `Result` can be collected directly: write a function turning `Vec<&str>` into `Result<Vec<u32>, ParseIntError>` by calling `.collect()` on an iterator of parsed results, and explain *why* `collect` is allowed to short-circuit on the first `Err`. Then adapt it to instead produce `Vec<Result<u32, ParseIntError>>` (one result per element, no short-circuit). Describe a real situation where you want the second shape — where stopping at the first bad element would be the wrong behaviour.

5. **What guarantee does privacy buy?** Take the `Percent` newtype. Suppose a teammate argues "the `new` constructor already checks `v <= 100`, so making the field `pub` is harmless." Construct the one line of downstream code that proves them wrong, then state precisely which guarantee a *private* field gives you that the runtime check in `new` alone does not. (Hint: think about who can build a `Percent` and how.)
