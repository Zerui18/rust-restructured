# 11. Error Handling

Most languages you know have a single error-handling channel that is *invisible in the type signature*. In Java, `String readConfig()` can throw `IOException` and you only learn that from the `throws` clause (checked) or, worse, from the documentation or a stack trace at 3am (unchecked `RuntimeException`). In C++, `std::string read_config()` can throw anything, and the signature says nothing — `noexcept` is opt-in and rarely used. C has no channel at all: you smuggle failure through return codes, `errno`, sentinel values, and out-parameters, and the compiler never checks that you looked.

Rust makes a hard, deliberate split that maps onto a distinction every other language blurs:

- **Bugs** — violated invariants, "this can't happen" situations, programmer errors. These use `panic!`, are *unrecoverable* by default, and are deliberately **not** in the type signature, because no caller should be writing recovery logic for your bug.
- **Expected failure** — a file that might not exist, input that might be malformed, a network that might be down. These are values of type `Result<T, E>`, **in the signature**, and the compiler forces every caller to confront them.

There are no exceptions in Rust. There is no `throw`, no stack-unwinding control flow you can `catch` (panic unwinding exists but is not a general control-flow tool — see below). The closest thing you already know is OCaml's `result` type and the `Stdlib.result` combinators, or Haskell's `Either e a` in the `Either`/`ExceptT` monad. Rust takes that model and bakes it into the standard library, the prelude, and a piece of syntax (`?`) that makes it ergonomic enough to use everywhere.

> **🦀 From your toolbox →** `Result<T, E>` is OCaml's `('a, 'b) result` = `Ok of 'a | Error of 'b`, and the `?` operator is the `let*` of a `result` monad. Where the analogy breaks: OCaml lets you freely mix `result` with `raise`/exceptions, and most of the stdlib raises (`List.hd`, `Hashtbl.find`). Rust has no second escape hatch you can catch routinely — `panic!` is for bugs, and unwinding is not idiomatic control flow. Compared to Java/C++ exceptions: a Rust function that can fail *must* say so in its return type. There is no hidden `throws`.

## Track one: `panic!` for unrecoverable bugs

`panic!` aborts the current thread's normal execution. You trigger it explicitly or — far more commonly — the standard library triggers it on your behalf when you violate a contract:

```rust
fn main() {
    let xs = vec![10, 20, 30];
    let _ = xs[99]; // panics: index out of bounds: the len is 3 but the index is 99
}
```

In C, `xs[99]` on a heap array is undefined behaviour: a buffer overread that hands you whatever bytes live past the allocation, a textbook security hole. Rust's `Index` impl for slices bounds-checks and panics instead. That is the whole philosophy in one line: **a contract violation is a bug, and continuing with corrupt assumptions is worse than stopping.**

A panic prints the message and source location, then by default *unwinds*:

```text
thread 'main' panicked at src/main.rs:3:13:
index out of bounds: the len is 3 but the index is 99
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Set `RUST_BACKTRACE=1` to get the call stack. Read it top-down and stop at the first frame in a file you wrote — that is where your bug lives; frames above are stdlib machinery (`core::panicking::panic_bounds_check` and friends).

### Unwinding vs. aborting

> **⚙️ Under the hood →** A panic does one of two things, chosen at compile time. The default is **unwind**: rustc walks back up the stack frame by frame, running every `Drop` impl it passes — exactly like a C++ exception running destructors during stack unwinding (it uses the same platform mechanism, the Itanium/SEH unwinder, and a landing pad per frame). This frees resources but bloats the binary with unwind tables. The alternative is **abort**: call `std::process::abort()` immediately, run no destructors, let the OS reclaim memory. You opt in per profile:
> ```toml
> [profile.release]
> panic = "abort"
> ```
> `abort` produces smaller binaries and is mandatory in some contexts (e.g. a panic crossing an `extern "C"` boundary is UB unless aborting — see [unsafe and FFI](16-unsafe-and-ffi.md)). With unwinding, a panic in a spawned thread kills only that thread; the parent observes it via the `JoinHandle` ([fearless concurrency](13-fearless-concurrency.md)).

> **🎓 Tripos link →** Unwinding-and-running-destructors is exactly the RAII cleanup you saw in *Programming in C and C++*: the same landing-pad codegen a C++ compiler emits for exception safety. The difference is intent. C++ exceptions are a control-flow mechanism you `catch` and recover from; Rust unwinding is a best-effort cleanup on the way to thread death. `std::panic::catch_unwind` exists, but it is for FFI boundaries and thread-pool isolation, not for `try`/`catch`-style logic.

`unwrap` and `expect` (below) are the most common ways `panic!` shows up in real code. The rule for when a panic is *correct* is the subject of the final section.

## Track two: `Result<T, E>` for recoverable failure

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

This is just a sum type — an OCaml variant with two constructors, generic in both the success payload `T` and the error payload `E` ([generics and traits](08-generics-and-traits.md)). It is in the prelude, so `Ok`/`Err` are in scope unqualified, exactly like `Some`/`None`. A fallible function returns it, and the *only* way to get at the `T` is to deal with the possibility of the `E`:

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

`File::open` returns `Result<File, std::io::Error>`. There is no way to forget the error: the value you have is a `Result`, not a `File`, and the type system will not let you call `File` methods on it. You must pattern-match (or use one of the combinators) to extract the `File`. This is the entire safety argument — **failure is encoded in the type, so the compiler can prove you handled it.**

`Result` is annotated `#[must_use]`. Ignore one and you get a warning, not silence:

```rust
fn main() {
    std::fs::remove_file("scratch.tmp"); // warning: unused `Result` that must be used
}
```

> **⚠️ Pitfall →** That warning catches the C bug class directly: in C you write `fclose(f);` and never check the return, silently dropping write errors. Here, `let _ = std::fs::remove_file("scratch.tmp");` is the explicit "yes, I am deliberately discarding this" — but if you are discarding an I/O error you almost always have a bug. The idiomatic fix is to propagate it with `?`.

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

The `?` after a `Result` means: if `Ok(v)`, evaluate to `v` and continue; if `Err(e)`, **return early** from the enclosing function with `Err(e)`. It is the `let*` bind of OCaml's `result` monad — `e1?; e2` is `match e1 { Ok(x) => e2, Err(e) => return Err(e) }`. Method chaining works because `?` is an expression that produces the unwrapped value: `File::open(path)?.read_to_string(&mut s)?`.

> **🎓 Tripos link →** This is the monadic error pattern from *Foundations of Computer Science* made syntactic. `?` is the bind (`>>=`) of the `Result` monad specialised to short-circuit on `Err`, and `Ok` is `return`/`pure`. Haskell wraps the same idea in `do`-notation over `Either`; OCaml 4.08+ has `let*` via a `Result`-monad `let` operator. Rust commits to one monad — `Result`/`Option` — and gives it dedicated syntax rather than a general `do`. The desugaring is the operational-semantics rewrite you'd write on paper: a small-step rule that reduces `v?` to either `v`'s payload or an early `return`.

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

`main` may return `Result<(), E>` for any `E: Debug`. The runtime prints the `Err` via `Debug` and exits with a non-zero code — mirroring the C convention where `main` returns `0` on success and non-zero on failure (Rust implements the `std::process::Termination` trait to do this):

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let _config = std::fs::read_to_string("config.toml")?;
    Ok(())
}
```

`Box<dyn Error>` is a trait object ([trait objects and OOP](09-trait-objects-and-oop.md)) meaning "any type implementing the `Error` trait, heap-allocated behind a pointer." It is the universal sink for `?`: because `Box<dyn Error>: From<E>` for every `E: Error`, any error converts into it. That is why it is the idiomatic error type for `main` and for quick application code.

`?` works on `Option` too, with `None` as the short-circuit value:

```rust
fn first_word_initial(text: &str) -> Option<char> {
    text.split_whitespace().next()?.chars().next()
}
```

If `next()` yields `None`, the function returns `None` immediately; otherwise `?` unwraps the `&str` and we keep going. Same mechanism, different monad.

## `unwrap` and `expect`: deliberate panics

`unwrap` and `expect` collapse a `Result`/`Option` to its inner value, panicking on the failure variant. They are the bridge from track two back to track one — "I assert this cannot fail; if I'm wrong, that's a bug, so crash."

```rust
let port: u16 = "8080".parse().unwrap();          // panics on the Err with a generic message
let cfg = std::fs::read_to_string("config.toml")
    .expect("config.toml must exist; it ships with the binary");
```

`expect` takes a message and panics *with* it; `unwrap` uses a generic one. **Prefer `expect` always.** The string is not a user-facing error message — it documents the *invariant you are asserting*, so that if the panic fires, the backtrace tells the next maintainer what assumption was wrong. "config.toml must exist" is a far better failure than "called `Result::unwrap()` on an `Err` value".

> **🦀 From your toolbox →** `.unwrap()` on `Option` is OCaml's `Option.get`, which raises `Invalid_argument` on `None`; `.expect(msg)` is C++'s `value.value()` on a disengaged `std::optional` (throws `bad_optional_access`) but with a custom message. The key cultural difference: in OCaml/C++ these are often used freely; in idiomatic Rust an `unwrap`/`expect` in library code is a claim that *demands justification in the message or a comment*, because it converts a recoverable error into a crash on behalf of every caller.

When is `unwrap`/`expect` acceptable?
- **Examples, prototypes, tests.** A failed `unwrap` in a `#[test]` panics, which is exactly how a test reports failure. Robust error handling would only obscure the example. (More in [testing and tooling](20-testing-and-tooling.md).)
- **When you can prove the `Err` is impossible** but the compiler cannot:
  ```rust
  use std::net::IpAddr;
  let loopback: IpAddr = "127.0.0.1".parse()
      .expect("hardcoded loopback address is always valid");
  ```
  `parse` returns a `Result` because *in general* a string may be malformed; here the literal is fixed and valid, so asserting it is fine — and the message records *why* it is safe, prompting a future reader to revisit if the input ever becomes dynamic.

## Custom error types and the `Error` trait

Returning `io::Error` everywhere works only while every failure really is an I/O error. A realistic function fails in several distinct ways (parse error, validation error, I/O error), and you want one type that names them all. Enums are the natural fit — a sum type, one variant per failure mode:

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

The `std::error::Error` trait is the standard interface for "this is an error." Its contract: you must also implement `Debug` (for programmers) and `Display` (for users), and you may override `source()` to return the underlying cause, which lets tools print a chain ("parse error … caused by … I/O error … caused by …"). Everything that takes `Box<dyn Error>` relies on this trait.

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

> **🎓 Tripos link →** This is precisely the ADT-based error modelling from *Foundations of Computer Science*: an error is a value of a sum type, and handling it is exhaustive pattern matching. The compiler's exhaustiveness check on `match self` in your `Display` impl is the same coverage guarantee that catches a forgotten variant. `From` conversions are the structure-preserving maps between your error algebras; `?`'s automatic `From::from` is functorial lifting of an inner failure into the outer error type.

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

## To panic or not to panic

The guiding principle: **returning `Result` is the default for any function that can fail, because it hands the decision to the caller.** Panicking decides *for* the caller that the situation is unrecoverable, which is presumptuous unless you are certain.

Panic when the program reaches a **bad state** — a broken invariant, contract, or guarantee — *and*:
- the bad state is unexpected (not routine, like malformed user input), and
- your subsequent code relies on not being in that state rather than re-checking, and
- there is no good way to encode the constraint in the types.

Return `Result` when failure is an **expected, routine possibility**: a parser meeting malformed input, an HTTP call hitting a rate limit, a file that may be absent. These are not bugs; they are the normal operating envelope, and the caller must choose how to respond.

A function's **contract** is the special case that justifies panic in library code. If your function's behaviour is only defined for certain inputs (a precondition), violating it is always a caller-side bug, and panicking is the right response — there is nothing for the caller to "handle," they must fix their code. This is exactly why slice indexing panics on out-of-bounds: operating on memory outside the structure is a security hole, so the stdlib refuses. Document any such panic in the function's API docs.

> **🎓 Tripos link →** The strongest move from *Semantics of Programming Languages* is to make the bad state *unrepresentable* rather than checking for it. Encode the invariant in a type and let the type system discharge the check statically — "make illegal states unrepresentable." A newtype with a private field and a validating constructor turns a runtime contract into a compile-time guarantee:
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
> Every function taking `Percent` (not `u8`) is now *proved* to receive a valid percentage — no defensive check, no panic, no `Result` at the use site. The private field means the only way to make a `Percent` is through `new`, so the invariant holds by construction. This is the type-system-as-proof-system idea from *Logic and Proof* applied to data: you have moved a proof obligation from runtime to the type checker. Choosing `u32` over `i32` to forbid negatives, or a concrete type over `Option` to forbid absence, are the same move at smaller scale.

## Mental-model recap

- **Two channels, by design:** `panic!` for bugs/violated invariants (unrecoverable, *not* in the signature), `Result<T, E>` for expected failure (recoverable, *in* the signature). Rust has no catchable exceptions; failure that callers should handle is always a value in the type.
- **`?` is monadic bind for `Result`/`Option`:** it early-returns on `Err`/`None` and, crucially, runs `From::from` on the error — so implementing `From<InnerError> for YourError` is what lets `?` aggregate many failure sources behind one type.
- **`unwrap`/`expect` convert a `Result` back into a panic** — only justified in tests/prototypes or where you can prove the `Err` is impossible. Always `expect` with a message stating the invariant, never bare `unwrap` in real code.
- **Concrete enum (or `thiserror`) for library errors** so callers can match on variants; **`Box<dyn Error>` or `anyhow` for application errors** where the consumer is a human reading a log. `thiserror` derives the boilerplate; `anyhow` erases the type for maximum `?` ergonomics.
- **Prefer making bad states unrepresentable** (newtypes, validating constructors, non-`Option` types) over runtime checks. A constraint discharged by the type checker is one you never check, never panic on, and never return a `Result` for at the use site.

## Exercises

1. Write `fn parse_pair(s: &str) -> Result<(i32, i32), std::num::ParseIntError>` that splits `s` on a comma and parses both halves with `?`. Then try to make it also handle the "no comma" case — observe that `s.split_once(',')` returns an `Option`, so `?` will not compile against a `ParseIntError` return type. Fix it two ways: (a) `.ok_or_else(...)` into a custom error, and (b) returning `Box<dyn Error>`. Compare the call sites.

2. Define a `ConfigError` enum by hand with three variants, then write `Display`, `Error` (with a correct `source()`), and the `From` impls. Now rewrite it with `thiserror`. Diff the generated behaviour: does your hand-written `source()` match what `#[from]` produces? Verify by printing an error and its `.source()` chain.

3. (★) The compiler error `the ? operator can only be used in a function that returns Result or Option` appears for a function returning `()`. Construct a case where the function *does* return `Result`, but `?` still fails to compile — because the inner error type does not implement `Into` the outer error type. Read the resulting `E0277` (`From<...> is not satisfied`) and fix it by adding the missing `From` impl. Explain in one sentence why `?` needs `From` and not just `Into`.

4. (★) Write a function that, given a `Vec<&str>`, parses each element to `u32` and collects into `Result<Vec<u32>, ParseIntError>`. Use `.collect()` directly on an iterator of `Result`s and explain *why* `collect` can turn `Iterator<Item = Result<T, E>>` into `Result<Vec<T>, E>` (short-circuiting on the first `Err`). Then change it to collect into `Vec<Result<u32, ParseIntError>>` instead and explain when each shape is the one you want.

5. Take the `Percent` newtype from the Tripos-link box. Make the field `pub` instead of private and write a downstream function that constructs an invalid `Percent(200)` directly, bypassing `new`. Then revert to a private field and observe the compile error at the construction site. Articulate precisely which guarantee privacy buys you that a runtime check in `new` alone does not.

6. Set `RUST_BACKTRACE=1` and run a program that panics three call-frames deep inside your own code. Identify the frame where *your* bug originates versus the stdlib frames above it. Then flip `panic = "abort"` in a release profile and confirm a `Drop` impl that prints on drop does **not** run on panic — explaining why aborting skips unwinding.
