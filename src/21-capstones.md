# 21. Capstones: a CLI tool & a Multithreaded Web Server

Twenty chapters of theory. Now we cash it in. This chapter is two projects, each engineered to force you to *use* the machinery — not the happy path, the load-bearing parts. Project A is a command-line utility: a "field cutter" that selects columns from delimited text. It exercises argument parsing, file vs. stdin I/O, the binary/library split, `Result`-based error handling with `?`, exit codes, and the stdout/stderr distinction. Project B is an HTTP server built from a raw `TcpListener` upward, culminating in a thread pool you write yourself out of channels, `Arc<Mutex<_>>`, and boxed closures — with graceful shutdown via `Drop`. Together they touch ownership, lifetimes, traits, generics, closures, concurrency, and the standard library, which is the point.

The designs here are mine, not the standard book's `grep` and `Hello!` server. The mechanics are identical; the framing is for an engineer who already knows what a socket and an `argv` are.

---

## Project A — `fcut`, a field cutter

The spec: read lines of delimited text, print only the requested fields. `fcut -d, -f 1,3 data.csv` prints columns 1 and 3 of `data.csv`, comma-separated. With no file argument, read stdin so the tool composes in a pipeline: `cat data.csv | fcut -f 2`. This is a deliberately small `cut(1)`, chosen because every interesting decision — parsing, error paths, I/O sources, testing — shows up at a scale you can hold in your head.

### Step 0: the bin/lib split, decided up front

The single most important structural decision in a Rust CLI is the one the standard library cannot make for you: **`main.rs` should be thin and untestable; `lib.rs` should be fat and tested.** The integration-test harness (see [chapter 20](20-testing-and-tooling.md)) compiles your crate as an external dependency and can only reach `pub` items in the library crate — it cannot call `fn main`. So anything you want to test must live in `lib.rs`.

Concretely, `main` is allowed exactly four jobs: parse the arguments, build a config, call the library's `run`, and turn any returned error into a message-on-stderr-plus-exit-code. Everything else — parsing config from raw args, splitting fields, formatting output — is library logic returning `Result`, so a test can assert on it directly.

```text
fcut/
├── Cargo.toml
├── src/
│   ├── main.rs      # thin: args → run() → exit code
│   └── lib.rs       # fat: Config, run(), cut_line(), errors
└── tests/
    └── cli.rs       # integration tests over the public API
```

> **🦀 From your toolbox →** In Java this is the reflex of extracting a `static` method off `Main` so JUnit can call it — you never test through `public static void main`. In Swift you'd push the logic into the framework target and keep the executable target a thin `@main` shim, because XCTest links against the framework, not the executable. In Python you guard the entry point with `if __name__ == "__main__":` and `import` the real functions into your `pytest` module. Rust *enforces* the same discipline through crate visibility rather than leaving it to convention: a `tests/` file genuinely cannot see private items or `main`, so the split is load-bearing, not stylistic. Where the analogy breaks: there is no separate "compile the test driver" step — `cargo test` builds both the lib and each `tests/*.rs` as its own crate automatically.

### Step 1: reading the arguments

`std::env::args()` returns an iterator of `String`. (`args_os()` yields `OsString` and never panics on non-UTF-8 paths; `args()` panics on invalid Unicode, which is fine for a learning tool, wrong for `find`-style filename munging.) The zeroth element is the program name, exactly as `argv[0]` in C — useful for usage messages, otherwise skipped.

For a tool this size I'll hand-roll the parser to show the seams, then tell you to never do that again. We parse into a `Config`, validating as we go and returning `Result` rather than panicking:

```rust
// src/lib.rs
pub struct Config {
    pub fields: Vec<usize>, // 1-based column numbers
    pub delimiter: char,
    pub path: Option<String>, // None ⇒ read stdin
}

impl Config {
    /// Parse already-collected args (excluding argv[0]).
    pub fn build(args: &[String]) -> Result<Config, String> {
        let mut fields = None;
        let mut delimiter = '\t';
        let mut path = None;
        let mut it = args.iter();

        while let Some(arg) = it.next() {
            match arg.as_str() {
                "-f" => {
                    let spec = it.next().ok_or("-f requires an argument")?;
                    fields = Some(parse_fields(spec)?);
                }
                "-d" => {
                    let d = it.next().ok_or("-d requires an argument")?;
                    delimiter = d.chars().next().ok_or("-d needs a character")?;
                }
                other if other.starts_with('-') => {
                    return Err(format!("unknown flag: {other}"));
                }
                other => path = Some(other.to_string()),
            }
        }

        Ok(Config {
            fields: fields.ok_or("missing required -f")?,
            delimiter,
            path,
        })
    }
}

fn parse_fields(spec: &str) -> Result<Vec<usize>, String> {
    spec.split(',')
        .map(|s| s.parse::<usize>().map_err(|_| format!("bad field: {s}")))
        .collect() // Result<Vec<_>, _>: the first Err short-circuits
}
```

Two things worth dwelling on. First, `parse_fields` collects an iterator of `Result<usize, String>` directly into a `Result<Vec<usize>, String>`. This is the `collect`-into-`Result` trick from [chapter 12](12-closures-and-iterators.md): `FromIterator` is implemented so that the first `Err` short-circuits and becomes the whole result. It is the iterator-level analogue of `?`. Second, `ok_or` converts an `Option` into a `Result` so the `?` in `build`'s callers (and the `?` here) can propagate a uniform error type.

> **🦀 From your toolbox →** `Option::ok_or` is the move you make in Swift when you turn an optional into a thrown error: `guard let x = opt else { throw MyError.missing }`. The optional says "maybe absent"; the conversion adds a *reason* for the absence so a caller can react. In Java the analogue is `Optional.orElseThrow(() -> new IllegalArgumentException("..."))`. `ok_or_else` is the lazy form — it takes a closure, so you don't build the error string unless the `None` case actually fires (just as `orElseThrow` only constructs the exception on the empty path). The break from Swift: there is no `throw` here, no stack unwinding — `ok_or` returns an ordinary `Result` value, and Rust's `?` then propagates it explicitly up the call chain.

### Step 1.5: in production, use `clap`

Hand-parsing teaches the shape; it does not scale to `--help`, abbreviations, subcommands, or value validation. The community default is **`clap`** with its derive API. You declare a struct and annotate it; the macro generates the parser, the help text, and the error messages:

```rust
use clap::Parser;

#[derive(Parser)]
#[command(version, about = "select fields from delimited text")]
struct Cli {
    /// Comma-separated 1-based field numbers, e.g. 1,3,5
    #[arg(short = 'f', value_delimiter = ',', required = true)]
    fields: Vec<usize>,

    /// Field delimiter
    #[arg(short = 'd', default_value_t = '\t')]
    delimiter: char,

    /// Input file; reads stdin if omitted
    path: Option<String>,
}

fn main() {
    let cli = Cli::parse(); // exits with a clap-formatted error on bad input
    // ...
}
```

Add it with `cargo add clap --features derive`. `Cli::parse()` reads `env::args_os()`, validates, and on failure prints a usage message to stderr and exits with code 2 — the conventional "usage error" code. The `value_delimiter = ','` attribute even reproduces our `parse_fields` for free. The derive macro is ordinary Rust code generated at compile time (see [chapter 18](18-macros.md)) and compiled like anything else: there is no reflection at runtime. Contrast Java, where annotation-driven frameworks like Spring or Jackson read annotations via reflection while the program is running; `clap` has done all of that work before `main` ever starts.

> **🎓 Tripos link →** `clap`'s derive is a hands-on Compiler Construction exercise. The `#[derive(Parser)]` macro reads the struct's tokens as input and *writes out* a parser as output — a source-to-source translation running inside `rustc`. The struct is effectively a tiny grammar; the generated code is the recogniser. You wrote argument and grammar parsers by hand in that course; here the macro writes one for you, and the compiler then type-checks it against the very struct that described the grammar.

> **🔧 In practice →** You're shipping a backup CLI and a teammate asks for `backup --dest ./snap --exclude '*.tmp' --exclude node_modules --dry-run -vvv`. With a hand-rolled loop you now own `--help`, repeated `--exclude`, the count-the-`v`s verbosity flag, and "did they pass a value?" checks — a day of fiddly bugs. With `clap` it's a struct:
> ```rust
> #[derive(Parser)]
> struct Backup {
>     #[arg(long)] dest: PathBuf,
>     #[arg(long)] exclude: Vec<String>,     // repeatable: collects every --exclude
>     #[arg(long)] dry_run: bool,            // presence ⇒ true
>     #[arg(short, long, action = clap::ArgAction::Count)] verbose: u8, // -vvv ⇒ 3
> }
> ```
> You reach for `clap` the moment a tool grows past two flags or a user might type `--help` — which is to say, almost always. Hand-roll only when you're learning the seams (as here) or you genuinely cannot take a dependency.

### Step 2: the I/O source — file or stdin, uniformly

We want `run` to treat "a file" and "standard input" identically. Both are byte streams; the standard-library abstraction for "a thing I can read lines from" is the `BufRead` trait. The trick is to obtain a `Box<dyn BufRead>` — a trait object (see [chapter 9](09-trait-objects-and-oop.md)) — so the rest of the code is agnostic about which concrete reader it holds:

```rust
use std::error::Error;
use std::fs::File;
use std::io::{self, BufRead, BufReader, Write};

pub fn run(config: Config, out: &mut impl Write) -> Result<(), Box<dyn Error>> {
    let reader: Box<dyn BufRead> = match &config.path {
        Some(p) => Box::new(BufReader::new(File::open(p)?)),
        None => Box::new(BufReader::new(io::stdin().lock())),
    };

    for line in reader.lines() {
        let line = line?; // io::Error per line (e.g. invalid UTF-8)
        writeln!(out, "{}", cut_line(&line, &config))?;
    }
    Ok(())
}

/// Pure function: trivially unit-testable.
pub fn cut_line(line: &str, config: &Config) -> String {
    let cols: Vec<&str> = line.split(config.delimiter).collect();
    config
        .fields
        .iter()
        .filter_map(|&f| cols.get(f - 1).copied()) // 1-based → 0-based; skip OOB
        .collect::<Vec<_>>()
        .join(&config.delimiter.to_string())
}
```

Note `run` takes `out: &mut impl Write` rather than printing directly. That single change makes the *entire output path* testable: a test passes a `Vec<u8>` (which implements `Write`) and inspects the bytes; `main` passes `io::stdout().lock()`. This is dependency injection done with a generic bound rather than an interface-typed field — and because `impl Write` monomorphises, there is zero dispatch cost for the real `stdout` case.

> **⚙️ Under the hood →** `Box<dyn BufRead>` is a fat pointer: one word to the heap-allocated reader, one word to its vtable. Each `reader.lines()` call dispatches through that vtable — one indirect call per line. `out: &mut impl Write`, by contrast, is *static* dispatch: the compiler emits one monomorphised `run` per concrete `W`, and `writeln!` inlines straight into a direct call. I mixed the two deliberately: the input side genuinely needs runtime polymorphism (the source is decided at runtime), the output side does not.

> **🦀 From your toolbox →** `io::stdin().lock()` returns a `StdinLock` that holds the global stdin mutex for as long as it lives. The pattern is Swift's `defer { handle.close() }` or Java's try-with-resources `try (var lock = stdin.lock())` — a resource whose release is tied to a scope rather than to a call you must remember. Reading line-by-line without locking re-acquires the lock per call; locking once and reading through the guard is both faster and the borrow that lets `BufRead` work. The difference from Swift/Java: you don't write `defer` or a `try (...)` block at all — the lock releases automatically when the value goes out of scope, the same "a destructor runs at the end of the block" idea you'd see in C++.

### Step 3: error handling, exit codes, stdout vs stderr

`run` returns `Result<(), Box<dyn Error>>`. The `Box<dyn Error>` (trait-object error, from [chapter 11](11-error-handling.md)) lets `?` absorb *any* error type that implements `std::error::Error` — `io::Error` from `File::open`, `ParseIntError`, our own `String`-via-`From` — without us declaring a bespoke enum. For a binary, where the only consumer is a human reading stderr, that erasure is the right trade; a library should expose a concrete error type so callers can match on it.

> **🔧 In practice →** You're writing a one-shot data-munging script: open a file, parse some JSON, look up an environment variable, write a report. Each step fails with a *different* error type (`io::Error`, `serde_json::Error`, `VarError`). Declaring a hand-written enum that wraps all four — plus the `From` impls to convert each — is pure boilerplate for a tool nobody calls as a library. So you type `Box<dyn Error>` once and let `?` flatten everything:
> ```rust
> fn report() -> Result<(), Box<dyn std::error::Error>> {
>     let raw = std::fs::read_to_string("metrics.json")?; // io::Error
>     let data: Metrics = serde_json::from_str(&raw)?;     // serde_json::Error
>     let bucket = std::env::var("BUCKET")?;               // VarError
>     // ... all three error types coerced into Box<dyn Error> by ?
>     Ok(())
> }
> ```
> The rule of thumb: `Box<dyn Error>` in application/binary code where the only reader is a human, a concrete enum in a library where a caller needs to `match` on which thing went wrong and recover differently.

`main` ties it together. It must do two things the library deliberately does *not*: choose the process exit code, and route messages to stderr.

```rust
// src/main.rs
use std::io::{self, Write};
use std::process;
use fcut::{run, Config};

fn main() {
    let args: Vec<String> = std::env::args().skip(1).collect();

    let config = Config::build(&args).unwrap_or_else(|err| {
        eprintln!("fcut: {err}");
        process::exit(2); // 2 = usage error, by convention
    });

    let stdout = io::stdout();
    if let Err(err) = run(config, &mut stdout.lock()) {
        eprintln!("fcut: {err}");
        process::exit(1); // 1 = runtime failure
    }
}
```

`eprintln!` writes to stderr; `println!`/`writeln!(stdout, …)` write to stdout. The distinction is not cosmetic — it is the contract that makes `fcut -f 2 data.csv > out.txt` work: the selected fields land in `out.txt` while any error message ("No such file") still reaches your terminal. Mix the two and you silently poison the output file with diagnostics. Test it: run with a missing file and `> /dev/null`; you should still see the error.

> **⚠️ Pitfall →** Reaching for `process::exit()` *inside* `run` to bail on an error. It compiles, runs, and is a latent bug: `process::exit` terminates immediately, skipping every destructor — open files are not flushed, your `Drop` impls do not run, and integration tests can't observe the failure because the test process dies too. The fix is structural: errors propagate up via `?` as `Result`, and *only `main`* converts a `Result` into an exit code. Keep `exit` out of the library entirely.

### Step 4: testing — unit, integration, and doc

Because the logic lives in `lib.rs` as pure-ish functions, the tests are cheap. Unit tests sit beside the code with `#[cfg(test)]`; integration tests in `tests/cli.rs` drive the public API as an external crate (see [chapter 20](20-testing-and-tooling.md)):

```rust
// in src/lib.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn cuts_selected_fields_in_order() {
        let cfg = Config { fields: vec![3, 1], delimiter: ',', path: None };
        assert_eq!(cut_line("a,b,c", &cfg), "c,a");
    }

    #[test]
    fn out_of_range_field_is_skipped_not_panicked() {
        let cfg = Config { fields: vec![9], delimiter: ',', path: None };
        assert_eq!(cut_line("a,b", &cfg), ""); // get() + filter_map ⇒ no panic
    }

    #[test]
    fn missing_f_is_an_error() {
        assert!(Config::build(&["data.csv".to_string()]).is_err());
    }
}
```

The `out_of_range` test is the one that matters: it pins down that `cols.get(f - 1)` returns `None` rather than indexing-and-panicking. Indexing `cols[f - 1]` would compile and pass the happy-path test, then panic in production on a short line — exactly the failure mode the borrow checker *cannot* catch, because bounds are a runtime property. Choosing `get` + `filter_map` over `[]` is the safety decision; the test enforces it.

A **doc test** doubles as documentation and a compiled, executed example — `cargo test` runs the code in `///` comments:

```rust
/// Selects fields from a delimited line.
///
/// ```
/// use fcut::{cut_line, Config};
/// let cfg = Config { fields: vec![2], delimiter: ' ', path: None };
/// assert_eq!(cut_line("alpha beta gamma", &cfg), "beta");
/// ```
pub fn cut_line(line: &str, config: &Config) -> String { /* ... */ }
```

If you change the API and forget to update the doc example, `cargo test` fails. Documentation that lies is now a build error — a property no amount of discipline gives you in C or Java.

---

## Project B — a multithreaded HTTP server from `TcpListener`

Now we go down a layer. We will speak HTTP over raw TCP using only `std::net`, then make it concurrent with a thread pool we build ourselves. The pedagogical payoff is the pool: it is where ownership, `Send`/`Sync`, `Arc`, `Mutex`, channels, closures, and trait objects all have to cooperate, and the borrow checker forces a correct design.

### Step 1: accept a connection

TCP and HTTP are both request/response protocols; TCP carries opaque bytes, HTTP defines what those bytes mean. `std::net::TcpListener::bind` returns a `Result` (binding fails if the port is taken), and `incoming()` yields a stream of `Result<TcpStream>` — one per *connection attempt*, which is why each can independently fail (the OS connection table can be full).

```rust
use std::net::{TcpListener, TcpStream};

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        handle(stream);
    }
}
```

> **🎓 Tripos link →** This is the Berkeley sockets model from Computer Networking, named in Rust types: `bind` is `bind(2)`, `incoming()` wraps `accept(2)`, and a `TcpStream` is a connected socket file descriptor. The difference from a C socket program is that the descriptor is *owned*: when `stream` is dropped at the end of the loop body, Rust calls `close(2)` for you — no leaked descriptors, no double-close. It is the same scope-tied cleanup as the file-handle locking above, now applied to a kernel resource: the destructor that runs at the end of the block closes the socket, the way Swift's `deinit` or a C++ destructor frees what it owns.

### Step 2: parse a request, route, respond

An HTTP request begins with a request line: `GET /path HTTP/1.1\r\n`, followed by headers and a blank line. For a minimal server we read just the first line, route on it, and write back a status line, a `Content-Length` header, a blank line, and a body. `TcpStream` implements `Read` and `Write`; wrapping it in a `BufReader` gives us `.lines()`.

```rust
use std::fs;
use std::io::{BufRead, BufReader, Write};
use std::net::TcpStream;
use std::thread;
use std::time::Duration;

fn handle(mut stream: TcpStream) {
    let buf = BufReader::new(&stream);
    let request_line = buf.lines().next().unwrap().unwrap();

    let (status, body) = match request_line.as_str() {
        "GET / HTTP/1.1" => ("200 OK", "<h1>up</h1>".to_string()),
        "GET /slow HTTP/1.1" => {
            thread::sleep(Duration::from_secs(5)); // simulate a slow handler
            ("200 OK", "<h1>eventually</h1>".to_string())
        }
        _ => ("404 NOT FOUND", fs::read_to_string("404.html").unwrap()),
    };

    let response = format!(
        "HTTP/1.1 {status}\r\nContent-Length: {}\r\n\r\n{body}",
        body.len()
    );
    stream.write_all(response.as_bytes()).unwrap();
}
```

We `match` on `request_line.as_str()` because `match` does no auto-deref coercion the way `==` does; matching a `String` against `&str` literals requires the explicit slice. Run it, hit `http://127.0.0.1:7878/` and `/slow` in two tabs, and you will watch the defect: the `/slow` request blocks the *entire* server for five seconds because the loop is serial. One slow client starves everyone. That is the problem the pool solves.

> **⚠️ Pitfall →** `buf.lines().next().unwrap().unwrap()` panics on a client that connects and sends nothing (the first `unwrap` hits `None`), or on a malformed line. A real server returns 400; here it crashes the worker thread. This is fine for a teaching server and a landmine in production — the lesson is that *every* `unwrap` on network input is a remote-triggerable panic. Production code propagates with `?` and a `Result`-returning `handle`.

### Step 3: the wrong fix, then the right interface

The naive fix is `thread::spawn(move || handle(stream))` per connection. It works and `/slow` no longer blocks `/` — but it is a denial-of-service vector: ten million connections spawn ten million threads and exhaust the OS. We want a *bounded* number of worker threads pulling jobs from a queue. The target interface mirrors `thread::spawn` so call sites barely change:

```rust
let pool = ThreadPool::new(4);
for stream in listener.incoming() {
    let stream = stream.unwrap();
    pool.execute(move || handle(stream));
}
```

Now we design `ThreadPool` so that this compiles. We will let the compiler's error messages drive the design — "compiler-driven development," the dual of the test-driven approach in Project A.

> **🔧 In practice →** You have 50,000 thumbnails to generate and resizing each one pegs a CPU core. Spawning 50,000 threads thrashes the scheduler and blows out memory; doing them one at a time wastes 15 of your 16 cores. A bounded pool sized to the core count is the sweet spot:
> ```rust
> let pool = ThreadPool::new(num_cpus::get()); // e.g. 16 workers
> for path in image_paths {
>     pool.execute(move || resize_and_save(path));
> }
> // pool drops at end of scope → all 50,000 jobs drained, workers joined
> ```
> The 50,000 jobs queue up; 16 workers chew through them with no thread ever idle and the OS never overwhelmed. You reach for a fixed-size pool whenever you have *many more units of work than cores* and each unit is CPU-bound — batch image/PDF processing, parallel checksums, a build system's compile steps. (For *I/O-bound* work waiting on the network, async tasks scale better — that's the `tokio` note at the end of the chapter. In real code you'd use `rayon` for the CPU case rather than hand-rolling this, but the shape is exactly what you're building here.)

### Step 4: what `execute` must accept

`execute` takes a closure and ships it to a worker. Three questions, all answered by the type system:

- **Which `Fn` trait?** Each job runs exactly once, so `FnOnce()` — matching the `Once` in the name (closures and the `Fn`/`FnMut`/`FnOnce` hierarchy: [chapter 12](12-closures-and-iterators.md)).
- **`Send`?** Yes — the closure is created on the main thread and *moves* to a worker thread, so its captured state must be safe to transfer across threads ([chapter 13](13-fearless-concurrency.md)).
- **`'static`?** Yes — the worker may outlive the stack frame that created the closure, so the closure may not borrow locals; it must own everything it captures.

```rust
pub fn execute<F>(&self, f: F)
where
    F: FnOnce() + Send + 'static,
{ /* ... */ }
```

To *store* such closures in a queue we need a uniform type. Closures each have a unique anonymous type, so we erase them behind a boxed trait object. The job type is:

```rust
type Job = Box<dyn FnOnce() + Send + 'static>;
```

`Box<dyn FnOnce()>` is a heap-allocated, type-erased callable — a fat pointer (data + vtable) just like `Box<dyn BufRead>` in Project A. This is the moment closures and trait objects unify: a closure that captures `stream` becomes an anonymous struct holding `stream`, boxed and dispatched dynamically.

> **⚙️ Under the hood →** Each closure compiles to an anonymous struct whose fields are its captures; the `FnOnce::call_once` impl is the body. `Box::new(f)` heap-allocates that struct; the `dyn FnOnce()` coercion attaches its vtable. Calling `job()` is one indirect call through the vtable. Historically `Box<dyn FnOnce()>` was hard to *call* — invoking `FnOnce` consumes `self` by value, but a `Box<dyn …>` has no statically known size to move. Modern Rust special-cases calling a boxed `FnOnce`, so `job()` just works. This is the runtime price of a thread pool: one allocation and one indirect call per job.

### Step 5: the queue — a channel behind `Arc<Mutex<Receiver>>`

The pool owns a channel's `Sender`; `execute` sends a `Job` down it. Each worker needs the `Receiver` to pull jobs. But `mpsc` is **m**ultiple-**p**roducer, **s**ingle-**c**onsumer: the `Receiver` is *not* `Clone`, and we have *N* consumers. So we share one `Receiver` across all workers — shared ownership across threads is `Arc`, and shared *mutable* access (each `recv` mutates the receiver's state) needs a `Mutex`. Hence the canonical `Arc<Mutex<Receiver<Job>>>` (smart pointers and interior mutability: [chapter 15](15-smart-pointers.md)).

```rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: Option<mpsc::Sender<Job>>, // Option so Drop can take() it
}

impl ThreadPool {
    /// Create a pool with `size` worker threads.
    ///
    /// # Panics
    /// Panics if `size == 0`.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }
        ThreadPool { workers, sender: Some(sender) }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);
        self.sender.as_ref().unwrap().send(job).unwrap();
    }
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>, // Option so Drop can take() it
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            // Lock, recv, UNLOCK, then run — order is critical.
            let message = receiver.lock().unwrap().recv();
            match message {
                Ok(job) => {
                    println!("worker {id}: running a job");
                    job();
                }
                Err(_) => {
                    println!("worker {id}: channel closed, exiting");
                    break;
                }
            }
        });
        Worker { id, thread: Some(thread) }
    }
}
```

The single subtlest line in the whole project is `let message = receiver.lock().unwrap().recv();`. The temporary `MutexGuard` returned by `lock()` lives only for this statement: we acquire the lock, `recv` one job, and the guard drops at the semicolon — *before* `job()` runs. That ordering is what makes the pool concurrent. If you instead wrote

```rust
// WRONG — serialises everything:
while let Ok(job) = receiver.lock().unwrap().recv() {
    job();
}
```

the guard's lifetime extends across the entire `while let` body (a `while let` holds its scrutinee's temporaries for the loop body), so a worker holds the mutex *while executing the job*, and the pool degrades to one-at-a-time — the very serialisation we set out to kill. The borrow checker won't flag this; it is a correctness bug hiding in a lifetime rule. The `match`/`break` form drops the guard at the `;`, then runs the job unlocked.

> **🎓 Tripos link →** This is the message-passing model from Concurrent & Distributed Systems realised with ownership. The channel `send` *moves* the `Job` from main to worker — there is no shared mutable job, so no data race on it; transferring ownership *is* the synchronisation. The `Arc<Mutex<Receiver>>` is the one genuinely shared resource, and the `Mutex` serialises access to *dequeueing* (taking a job off the queue, which must happen one worker at a time) while leaving job *execution* fully parallel. Holding the lock only across `recv` is the course's "keep the critical section as small as possible" rule — here it comes down to exactly where you put a semicolon.

> **⚙️ Under the hood →** `Arc<T>` is a pointer to a heap allocation carrying `T` plus two atomic counters (strong/weak). `Arc::clone` is an atomic increment, not a deep copy — all four workers point at the *same* `Mutex<Receiver>`. `Rc` would be faster (non-atomic refcount) but is `!Send`, so it cannot cross the `thread::spawn` boundary; the compiler rejects it. That `!Send`-ness is the type system encoding "this refcount is not safe to touch from two threads," the data-race-freedom proof from [chapter 13](13-fearless-concurrency.md) discharged structurally.

### Step 6: graceful shutdown via `Drop`

`Ctrl-C` kills threads mid-request. We want shutdown to (a) stop accepting jobs and (b) let in-flight jobs finish. The mechanism is RAII: implement `Drop` on `ThreadPool` to join every worker. Two obstacles, both instructive.

First, joining consumes the `JoinHandle` (`join(self)` takes ownership), but in `drop(&mut self)` we only have a *mutable borrow* of each worker — you cannot move a field out of a borrow (error E0507). That is why `Worker::thread` is `Option<JoinHandle<()>>`: `Option::take` swaps in `None` and hands us the owned handle, the standard move-out-of-`&mut` idiom.

Second, the workers `loop` forever on `recv`, so `join` would block eternally. We must *close the channel* to make `recv` return `Err` and break each loop. The `Sender` is also wrapped in `Option` so we can `take` and drop it first:

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        // 1. Close the channel: drop the only Sender.
        drop(self.sender.take());
        //    Every worker's blocked recv() now returns Err ⇒ its loop breaks.

        // 2. Join each worker so in-flight jobs finish.
        for worker in &mut self.workers {
            println!("joining worker {}", worker.id);
            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
```

Order is everything: dropping the `Sender` *before* joining is what unblocks the workers; reverse it and `drop` deadlocks. Now `main` can shut down deterministically — e.g. serve a fixed number of requests for a demo:

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);
    for stream in listener.incoming().take(4) { // demo: stop after 4
        pool.execute(move || handle(stream.unwrap()));
    }
    println!("shutting down");
    // pool drops here → Drop runs → channel closes → workers join
}
```

When `pool` leaves scope, `Drop` runs: the channel closes, each worker's `recv` returns `Err`, the loops break, the threads finish their current job and exit, and `join` collects them — clean shutdown, every destructor honoured.

> **🦀 From your toolbox →** Think of how cleanup works in the languages you know. In Java a `finally` block or `AutoCloseable.close()` joins/releases, but only if you remember to write the `try` and never let an early return skip it — and a garbage-collected object's finalizer runs *whenever* the GC feels like it, if at all. Swift's `deinit` is closer: it runs deterministically the moment the last reference drops, like a destructor at scope end. Rust gives you Swift-style determinism but stronger: `drop` runs exactly once at scope exit (unless you explicitly `std::mem::forget` or call `process::exit`), and after the pool moves elsewhere the compiler won't let you touch the moved-from value at all. The `Option`-`take` dance is the price of that strictness: `Drop::drop` only gets `&mut self`, and you can't move a field out of a borrow, so `take` swaps in `None` and hands you the owned `JoinHandle`. Java/Swift dodge this only because they let a finalizer reach into fields freely — and pay for it with the use-after-free and double-cleanup bugs Rust rules out.

> **⚠️ Pitfall →** A subtle double-panic hazard: `thread.join().unwrap()` inside `drop` panics if a worker panicked. Since `drop` itself can run *during* unwinding from another panic, a panic here causes a double-panic, which aborts the process immediately and skips remaining cleanup. Fine for a demo; production code logs the join error instead of unwrapping. The general rule: destructors should not panic.

---

## Where to go next: the ecosystem

Everything in Project B is real and works, and you should not ship it. The point was to see the gears; production Rust stands on crates that have already solved these problems at industrial quality.

- **`serde`** — the (de)serialisation framework. `#[derive(Serialize, Deserialize)]` turns a struct into JSON, TOML, MessagePack, and more, all monomorphised at compile time with no reflection. If `fcut` grew a config file or the server spoke JSON, `serde` is the answer.
- **`tokio`** — the dominant async runtime ([chapter 14](14-async.md)). Where our pool maps one OS thread per concurrent job, `tokio` multiplexes thousands of `async` tasks onto a small thread pool via an epoll/kqueue event loop. Crucially, the *concurrency model* — jobs handed to workers — is the same; tasks are just cheaper than threads, so you scale to tens of thousands of connections instead of dozens.
- **`axum` / `actix-web`** — production HTTP frameworks atop `tokio`. Routing, extractors, middleware, typed request/response bodies, TLS. Our entire Step 2 collapses to a router and a handler function; you never touch a raw `TcpStream`.
- **`rayon`** — data parallelism. `iter()` becomes `par_iter()` and your `map`/`filter`/`sum` pipeline runs across a work-stealing thread pool, kept data-race-free by the same `Send`/`Sync` bounds. It is the thread pool of Project B, productionised and made invisible.

And the honest note the standard book makes too: **the async version is how you would write Project B in production.** Our thread-per-job pool is a clear teaching model, but a server holding ten thousand connections wants ten thousand cheap tasks, not ten thousand OS threads. The async building blocks — futures, `await`, the runtime — are [chapter 14](14-async.md); now that you have built the synchronous pool by hand, you can see exactly which problem `tokio` is solving and why it is shaped the way it is. Many async runtimes, `tokio` included, *are* thread pools under the hood — you have built a small, honest version of the thing they generalise.

---

## A send-off

Twenty-two chapters ago the thesis was that Rust buys memory safety and data-race freedom *at compile time, with no garbage collector and no runtime check on the hot path*, by making ownership a typing discipline. Everything since has been that thesis cashed out: borrowing is "share many readers OR hand out one writer, never both"; a lifetime is just the stretch of code during which a borrow is allowed to be alive; `Send`/`Sync` are how the compiler proves you can't get a data race; `Drop` is scope-tied cleanup the compiler runs for you; traits are reusable behaviour the compiler stamps out a specialised copy of per type; `?` and `Result` are error handling without exceptions. The two projects here did not introduce one genuinely new idea — they *composed* old ones, and that composition is where Rust either rewards you or fights you. When it fights you, the error message is almost always pointing at a real bug you would have shipped in C.

You now know enough to read the standard library's source, contribute to a crate, and — most importantly — to argue with the borrow checker and usually concede it was right. Go build something that would have been a segfault in another language. That is the whole point.

## Mental-model recap

- **Bin/lib split is structural, not stylistic.** `main.rs` parses args, calls `run`, and maps `Result`→exit-code; everything testable lives in `lib.rs` because the integration harness can only reach `pub` library items and never `main`. Inject the output sink as `&mut impl Write` so the output path is testable too.
- **`process::exit` and panics skip destructors.** Keep `exit` in `main` only; propagate with `?` everywhere else. Route diagnostics to stderr and data to stdout so redirection works — that distinction is a contract, not decoration.
- **A thread pool unifies half the book:** jobs are `Box<dyn FnOnce() + Send + 'static>` (closures-as-trait-objects), shipped over an `mpsc` channel (ownership-transfer message passing), pulled by workers sharing one `Arc<Mutex<Receiver>>` (thread-safe shared ownership + synchronised dequeue). `Rc` fails here precisely because it is `!Send`.
- **Lock scope is concurrency.** Hold the `MutexGuard` only across `recv` (end the statement with `;`), never across `job()`. A `while let receiver.lock()…` accidentally extends the guard over the loop body and serialises the pool — a correctness bug the borrow checker cannot catch.
- **Graceful shutdown is `Drop` + closing the channel.** `Option::take` moves the `JoinHandle`/`Sender` out of `&mut self`; drop the `Sender` *first* to make `recv` return `Err` and break the worker loops, *then* `join`. Reverse the order and `drop` deadlocks.

## Exercises

1. **Predict the type, then a design call.** The chapter's `cut_line` returns a fresh `String`. Suppose you instead wanted it to return the selected columns *without copying* the text — borrowing slices straight out of the input line. Sketch the new signature. What lifetime relationship must hold between the input `&str` and the returned value, and write down (in words) what the compiler would refuse if you tried to return those slices from a function that *owned* its input line rather than borrowing it. Which version would you actually ship for `fcut`, and why does it barely matter here but matter a lot in a tight inner loop?

2. **A new error path the text didn't handle.** As written, a line shorter than the requested field is silently skipped (the `filter_map` drops it). A teammate argues that for `fcut -f 3` on a 2-column line, the *right* behaviour is to emit an empty field so columns stay aligned, not to drop it. Decide which behaviour you'd defend for a `cut`-style tool, then describe the one-line change to `cut_line` that switches between "skip missing fields" and "emit empty for missing fields." (You don't need to run it — reason about what `cols.get(...)` returns and how `filter_map` vs `map` would treat it.)

3. **Which of these compiles, and why.** For each `execute` bound below, say whether the pool would accept the closure `move || handle(stream)` and explain in one line:
   - `F: FnOnce() + Send + 'static` (the chapter's choice)
   - `F: Fn() + Send + 'static`
   - `F: FnOnce() + 'static` (dropped `Send`)
   - `F: FnMut() + Send` (dropped `'static`)
   Then answer: *why* did the chapter pick `FnOnce` over `Fn`, given that `Fn` is "more capable"? (Hint: what does requiring `Fn` *forbid* the closure from doing to its captures?)

4. **Adapt the pool to report results.** Right now `execute` is fire-and-forget — the worker runs the job and the caller learns nothing. Suppose each job computes a `u64` checksum and the main thread needs to collect all of them. Without writing the full implementation, design the change: what would the job's return type become, and what second channel (going which direction) would you add so workers can hand results *back*? Sketch the new `execute` signature and say where the receiving end lives.

5. (★) **Predict the failure before compiling.** You decide to swap `Arc<Mutex<Receiver>>` for `Rc<RefCell<Receiver>>` to skip the atomic-refcount cost, reasoning "the workers never touch the same receiver at the same instant anyway." Predict the *exact* trait bound that `thread::spawn` will report as unsatisfied, and on which type, **before** you compile. Then explain at the field level why no amount of "they don't actually race" reasoning persuades the compiler — what is it about `Rc`'s refcount that makes the rejection a sound call rather than an overcautious one? Finally: name one situation where `Rc<RefCell<_>>` *is* the right tool and `Arc<Mutex<_>>` would be wasteful overhead.
