# Appendix. Mapping to the official book & your Tripos courses

This book is the official *Rust Programming Language* (21 chapters) reassembled by conceptual dependency rather than beginner ramp. Two consequences follow, and this appendix indexes both:

1. **Where is the "for everyone" version?** Every module below names the official chapter(s) it derives from, so when you want the canonical, slower, beginner-pitched treatment of a topic you can jump straight to it. Read the official book as the *reference manual*; read this one as the *theory*.
2. **What did I already learn that this leans on?** Each module names the Cambridge Tripos course(s) whose machinery it reuses, so you can revisit your own notes for the deeper framing. These are study anchors, not prerequisites.

A word on the chapter column: the official book is not a partition we re-shuffled one-to-one. We **merged** topics it splits (pattern matching, traits), **compressed** topics it stretches (the ch. 1–3 on-ramp), and **gathered** the three faces of the memory model (ownership, lifetimes, smart pointers) that it scatters across hundreds of pages. So several rows cite the same official chapter, and a few official chapters are deliberately dissolved across multiple modules. The "moved and why" notes below explain each non-trivial deviation.

## The table

| This book (module) | Official Rust Book ch. | Leans on (Tripos) |
|---|---|---|
| [00 · Orientation: the thesis of Rust](00-orientation.md) | Foreword, Introduction | Programming in C and C++; Computer Architecture; Operating Systems |
| [01 · The Language Delta](01-language-delta.md) | 1 Getting Started; 2 Guessing Game; 3 Common Programming Concepts | Foundations of CS (OCaml); Programming in C and C++; Compiler Construction |
| [02 · Ownership & Moves](02-ownership-and-moves.md) | 4.1–4.2 (Ownership; Move semantics) | Programming in C and C++; Compiler Construction; Operating Systems |
| [03 · References & Borrowing](03-references-and-borrowing.md) | 4.2 (References & Borrowing) | Programming in C and C++; Concurrent & Distributed Systems; Compiler Construction |
| [04 · Lifetimes as Regions](04-lifetimes.md) | 10.3 (Validating References with Lifetimes) | Compiler Construction; Operating Systems; Algorithms |
| [05 · Slices & the Owned/Borrowed Duality](05-slices-and-duality.md) | 4.3 (The Slice Type) | Programming in C and C++; Computer Architecture; Algorithms |
| [06 · Structs & Methods](06-structs-and-methods.md) | 5 Structs | Programming in C and C++; Further Java/OOP |
| [07 · Enums & Pattern Matching](07-enums-and-pattern-matching.md) | 6 Enums & Pattern Matching; 18 Patterns & Matching | Foundations of CS (OCaml); Compiler Construction; Algorithms |
| [08 · Generics & Traits](08-generics-and-traits.md) | 10.1–10.2 (Generics; Traits) | Foundations of CS (OCaml); Compiler Construction; Further Java/OOP |
| [09 · Trait Objects, Dynamic Dispatch & "OOP"](09-trait-objects-and-oop.md) | 17 OOP Features | Further Java/OOP; Compiler Construction; Programming in C and C++ |
| [10 · Common Collections](10-collections.md) | 8 Common Collections | Algorithms; Databases; Computer Architecture |
| [11 · Error Handling](11-error-handling.md) | 9 Error Handling | Foundations of CS (OCaml); Compiler Construction; Operating Systems |
| [12 · Closures & Iterators](12-closures-and-iterators.md) | 13 Functional Features | Foundations of CS (OCaml); Compiler Construction |
| [13 · Fearless Concurrency](13-fearless-concurrency.md) | 16 Fearless Concurrency | Concurrent & Distributed Systems; Operating Systems; Computer Architecture |
| [14 · Asynchronous Rust](14-async.md) | 17 Async/Await | Concurrent & Distributed Systems; Operating Systems; Computer Networking |
| [15 · Smart Pointers & Interior Mutability](15-smart-pointers.md) | 15 Smart Pointers | Programming in C and C++; Concurrent & Distributed Systems; Operating Systems |
| [16 · Unsafe Rust, FFI & the Memory Model](16-unsafe-and-ffi.md) | 20.1 (Unsafe Rust) | Programming in C and C++; Computer Architecture; Operating Systems; Cybersecurity |
| [17 · Advanced Traits & Types](17-advanced-traits-and-types.md) | 20.2–20.3 (Advanced Traits; Advanced Types) | Foundations of CS (OCaml); Compiler Construction; Further Java/OOP |
| [18 · Macros & Metaprogramming](18-macros.md) | 20.5 (Macros) | Compiler Construction; Foundations of CS (OCaml) |
| [19 · Crates, Modules & Workspaces](19-project-structure.md) | 7 Packages, Crates & Modules; 14 More About Cargo | Further Java/OOP; Compiler Construction; Operating Systems |
| [20 · Testing & Tooling](20-testing-and-tooling.md) | 11 Writing Automated Tests; 14 More About Cargo | Algorithms; Compiler Construction; Cybersecurity |
| [21 · Capstones (CLI tool & Multithreaded Web Server)](21-capstones.md) | 12 I/O Project (CLI); 21 Multithreaded Web Server | Operating Systems; Computer Networking; Concurrent & Distributed Systems; Algorithms |

## If you came from the official book, here's what moved and why

If you have read, or are cross-referencing, the official book, these are the structural deviations worth knowing about so a row above doesn't surprise you:

- **Ownership unified into one Part II (official 4 + 10.3 + 15).** The official book treats ownership (ch. 4), lifetimes (ch. 10.3), and smart pointers (ch. 15) as three unrelated topics hundreds of pages apart. They are one idea — a single memory model where each value has exactly one owner and the compiler tracks how long each borrow stays valid. [Modules 02–05](02-ownership-and-moves.md) teach moves, borrows, lifetimes-as-regions, and slices as a single contiguous theory; [module 15](15-smart-pointers.md) is its sequel, deferred only because it needs traits.
- **Pattern matching merged (official 6 + 18 → [module 07](07-enums-and-pattern-matching.md)).** The official book introduces `match` in ch. 6 and finishes the pattern grammar (refutability, guards, bindings, `@`, slice patterns) in ch. 18 — twelve chapters later. If you have used `switch` in Java or Swift, or pattern matching in OCaml, splitting this is pure friction; we teach the whole grammar once, next to the sum types it deconstructs.
- **Chapter 3 compressed into [the Language Delta](01-language-delta.md).** The official ch. 1–3 spend three chapters on installation, a guessing game, and variables/types/control flow. You do not need to be told what a `for` loop is. We collapse all of it into one chapter written as a *diff* against the languages you already know, keeping only what is genuinely Rust-specific (expression-orientation, shadowing, the integer-overflow contract, `let else`, exhaustive `match` as control flow).
- **Lifetimes pulled forward (official 10.3 → [module 04](04-lifetimes.md)).** The official book hides lifetimes inside the generics chapter, after structs, enums, collections, and error handling. But lifetimes *are* the precise statement of the borrow rules in [module 03](03-references-and-borrowing.md); leaving them late means every earlier borrow error is unexplained. We teach them immediately, framed as **regions**: a region is just the span of code over which a borrow must stay valid, and the compiler checks that the thing borrowed outlives it.
- **Traits unified and split from generics where it helps (official 10 + 20.2–20.3).** Core generics-and-traits live in [module 08](08-generics-and-traits.md); the advanced trait machinery the official book parks in ch. 20 (associated types, GATs-adjacent patterns, `dyn` upcasting, the newtype/`Deref` idioms, never type, sized-ness) gets its own [module 17](17-advanced-traits-and-types.md) rather than being smuggled into the introduction.
- **Smart pointers placed *after* traits, not before concurrency (official 15 → [module 15](15-smart-pointers.md)).** `Box`, `Rc`/`Arc`, `RefCell`, and `Deref`/`Drop` are *trait-defined* abstractions over the memory model. They cannot be explained before [traits (08)](08-generics-and-traits.md), and `RefCell`'s "either many readers or one writer, checked at runtime" only makes sense once the compile-time rule from [module 03](03-references-and-borrowing.md) is fully internalised — so they sit at the head of the systems-programming Part VI, feeding directly into [unsafe (16)](16-unsafe-and-ffi.md).
- **OOP and dynamic dispatch promoted out of "advanced" (official 17 → [module 09](09-trait-objects-and-oop.md)).** Trait objects and `dyn` dispatch belong beside static dispatch as the *other half* of the same monomorphisation-versus-vtable trade-off — the same choice Java makes with its virtual method tables — so they follow generics immediately rather than waiting for a late "OOP" chapter.

## Reading paths

**Fast track** (you want fluency in a fortnight): read [00](00-orientation.md)–[01](01-language-delta.md) for the framing, then *all* of Part II ([02](02-ownership-and-moves.md)–[05](05-slices-and-duality.md)) — it is non-negotiable and load-bearing — then [06](06-structs-and-methods.md)–[09](09-trait-objects-and-oop.md) for the type system, [11](11-error-handling.md)–[12](12-closures-and-iterators.md) for the idioms that pervade real code, and [13](13-fearless-concurrency.md); skim [10](10-collections.md) as reference and treat [14](14-async.md)–[20](20-testing-and-tooling.md) as need-driven lookups, returning for the [capstones (21)](21-capstones.md) once the borrow checker has stopped surprising you. **Complete path**: read straight through 00→21 in book order — the sequence is dependency-sorted, each Part assumes the last, and the capstones are written to integrate every preceding module, so they reward (and reveal gaps in) a full read.
